# Hypervisor Provisioning: Lessons Learned & Fixes

This document records the critical configuration fixes, security exemptions, and architectural decisions discovered during the successful deployment of the RHEL 9 and RHEL 10 CIS-hardened KVM hypervisors.

## 1. The "KVM Shield" (CIS Hardening Overrides)

During Phase 2, the CIS Benchmark roles aggressively secure the host by deleting unapproved packages and blacklisting unused kernel modules. Without surgical overrides, the CIS role will physically destroy a working hypervisor.

We implemented critical exemptions in `cis_rhel9_host.yml` and `cis_rhel10_host.yml` to act as a "KVM Shield":

- **Cockpit Survival**: `rhelXcis_cockpit_server: true` prevents the CIS role from running `dnf remove cockpit`.
- **DNS/DHCP Survival**: `rhelXcis_dnsmasq_server: true` prevents the CIS role from uninstalling `dnsmasq`, which is strictly required by `libvirt` for the `virbr0` network bridge to assign IPs to VMs.

## 2. Sysctl IP Forwarding Resilience

Virtual machines require `net.ipv4.ip_forward = 1` for NAT internet routing.

- The CIS role (Rule 3.3.1.x) attempts to drop a configuration file (`/etc/sysctl.d/60-netipv4_sysctl.conf`) enforcing `0`.
- Instead of fighting the CIS role directly, the `rhel_kvm` role writes its configuration to `/etc/sysctl.d/99-kvm.conf`.
- **Precedence Victory**: Because `99` is processed after `60`, the KVM role naturally steamrolls the CIS restriction upon every boot. The hypervisor natively self-heals its routing configuration, rendering the CIS rule functionally irrelevant without breaking compliance logic.

## 3. End-to-End SSH ProxyJump Configuration

When provisioning a private hypervisor (e.g., `192.168.20.40`) through a Jump Host (`192.168.10.160`), the Ansible Control Node requires end-to-end key authentication.

- The SSH key must be copied _through_ the jump host to the target:
  ```bash
  ssh-copy-id -o ProxyJump=frqadmin@192.168.10.160 frqadmin@192.168.20.40
  ```
- This allows Ansible to authenticate directly to the target VM transparently, avoiding SSH timeouts or hanging tasks (which surface as 100% idle `top` graphs on the target).

## 4. Libvirt Ansible Module Idempotence

The `community.libvirt.virt_pool` and `community.libvirt.virt_net` modules have strict lifecycle requirements:

- You **cannot** use `state: active` on a resource that doesn't exist yet.
- The architecture requires splitting the tasks:
  1. `state: present` (Defines the XML configuration).
  2. `state: active` (Starts the defined configuration).
  3. `autostart: true` (Ensures it boots with the host).

## 5. RHEL 10 Beta GPG Keys

Early RHEL 10 Beta ISOs shipped with a broken initial keyring where RPM signatures failed validation during DNF operations.

- **The Fix**: Manually run `sudo dnf update redhat-release` on the target host before provisioning. This lays down the newer Red Hat Release Key 4 and resolves the signature mismatch, allowing Ansible's DNF module to install virtualization packages smoothly.

## 6. RHEL 9 Modular Daemons Silently Winning Over `libvirtd`

**Requirement**: RHEL 9 must run monolithic `libvirtd`. This is a fixed project requirement, not a preference — do not change `rhel_kvm_daemon_model` on RHEL 9 to modular to work around the issues below; fix the URI instead (item #7).

On a RHEL 9.8 host in the field, `systemctl status libvirtd` showed `inactive (dead)` while `virtqemud`, `virtnetworkd`, and `virtstoraged` were all `active running` — even `libvirtd.socket` itself was `inactive dead`, so nothing would ever wake `libvirtd` back up.

- **Root cause**: `vars/RedHat-9.yml` started `libvirtd` correctly but never masked the modular daemon set (it only had an unused, empty `rhel_kvm_disabled_sockets: []`). RHEL 9.8's own systemd presets bring `virtqemud`/`virtnetworkd`/`virtstoraged` up on their own, and since both daemon families ship on RHEL 9, the modular set wins the default `qemu:///system` connection out from under the never-masked `libvirtd`.
- **The Fix**: `vars/RedHat-9.yml` now sets `rhel_kvm_disabled_services` to the full modular daemon list (services + sockets for `virtqemud`, `virtnetworkd`, `virtstoraged`, `virtnodedevd`, `virtsecretd`, `virtnwfilterd`), same shape as how `RedHat-10.yml` masks legacy `libvirtd` units. `tasks/daemons.yml` needed no changes — it already loops over `rhel_kvm_disabled_services` generically. This alone gets `systemctl status libvirtd` looking correct, but is not the full fix — see item #7.
- **Manual remediation** on an already-drifted host:
  ```bash
  sudo systemctl disable --now virtqemud.service virtqemud.socket virtqemud-ro.socket virtqemud-admin.socket \
    virtnetworkd.service virtnetworkd.socket virtnetworkd-ro.socket virtnetworkd-admin.socket \
    virtstoraged.service virtstoraged.socket virtstoraged-ro.socket virtstoraged-admin.socket \
    virtnodedevd.service virtnodedevd.socket virtnodedevd-ro.socket virtnodedevd-admin.socket \
    virtsecretd.service virtsecretd.socket virtsecretd-ro.socket virtsecretd-admin.socket \
    virtnwfilterd.service virtnwfilterd.socket virtnwfilterd-ro.socket virtnwfilterd-admin.socket
  sudo systemctl mask <same unit list>
  sudo systemctl enable --now libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket
  sudo systemctl start libvirtd.service
  ```

## 7. Masking the Modular Daemons Isn't Enough — the libvirt Client Still Defaults to `virtqemud-sock`

After applying item #6's fix and re-running just the `rhel_kvm` role, `tasks/storage.yml`'s `community.libvirt.virt_pool` task failed anyway:

```
Failed to connect socket to '/var/run/libvirt/virtqemud-sock': Connection refused
```

- **Root cause**: this has nothing to do with which systemd units are enabled/masked. The libvirt **client** library (used internally by `community.libvirt.virt_pool`/`virt_net`, and by plain `virsh`) resolves a bare `qemu:///system` connection to the modular per-driver socket (`virtqemud-sock`) by default whenever the modular daemon packages are installed on the host — it does not fall back to `libvirtd-sock` on its own. Masking `virtqemud` (item #6) removes the daemon behind that socket but does nothing to change which socket path the client tries first, turning a silent wrong-daemon problem into an outright connection failure.
- **Rejected fix**: switching RHEL 9 to the modular daemon model (matching RHEL 10) so the client's default matches reality. This was tried and reverted — RHEL 9 is required to stay monolithic regardless of what the libvirt client would prefer by default. Don't repeat this.
- **The actual fix**: keep RHEL 9 monolithic and remove the ambiguity instead. `vars/RedHat-9.yml` defines `rhel_kvm_libvirt_uri: "qemu+unix:///system?socket=/var/run/libvirt/libvirt-sock"` — an explicit, fully-qualified URI that points straight at `libvirtd`'s own socket, so the client has nothing left to auto-resolve. Every `community.libvirt.virt_pool`/`virt_net` task in `tasks/storage.yml` and `tasks/networks.yml` uses `uri: "{{ rhel_kvm_libvirt_uri }}"` instead of a hardcoded `qemu:///system`, and `tasks/users.yml` exports the same value as `LIBVIRT_DEFAULT_URI` in `/etc/profile.d/libvirt.sh` so interactive `virsh` sessions get it too. RHEL 10's `vars/RedHat-10.yml` sets `rhel_kvm_libvirt_uri: "qemu:///system"` — already unambiguous there since `libvirtd` doesn't exist to compete with.
- **Manual remediation** on a host where `virtqemud` is masked and `libvirtd` is active-but-unreachable via the default URI:
  ```bash
  sudo virsh -c qemu+unix:///system?socket=/var/run/libvirt/libvirt-sock list --all
  ```
  If that connects, the daemon side is fine — it's just confirming the same fix Ansible now applies. No daemon-model change needed.
- **Lesson**: don't assume systemd unit state alone determines which libvirt daemon a client actually reaches — the client library's own connection-URI resolution can have a fixed preference order that ignores admin intent entirely. When a hard requirement (monolithic on RHEL 9) conflicts with a client's default behavior, force the URI explicitly rather than changing the requirement to match the default.

## 8. Verification Script Execution (Root Privileges)

The custom python validation scripts (`verify_hypervisor.py` and `verify_cockpit.py`) generate "false failures" if executed as a standard user.

- **Networking**: `virsh net-info` and `virsh pool-list` query `qemu:///session` instead of `qemu:///system` if not run as root.
- **Ports/Firewalls**: `ss -tulpn` and `firewall-cmd` hide bound process IDs and active zones from non-root users.
- **Rule of Thumb**: Always execute acceptance tests with `sudo` to ensure they audit the enterprise `qemu:///system` daemon.
