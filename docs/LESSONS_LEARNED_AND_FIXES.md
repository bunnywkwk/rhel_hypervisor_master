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

## 6. Verification Script Execution (Root Privileges)

The custom python validation scripts (`verify_hypervisor.py` and `verify_cockpit.py`) generate "false failures" if executed as a standard user.

- **Networking**: `virsh net-info` and `virsh pool-list` query `qemu:///session` instead of `qemu:///system` if not run as root.
- **Ports/Firewalls**: `ss -tulpn` and `firewall-cmd` hide bound process IDs and active zones from non-root users.
- **Rule of Thumb**: Always execute acceptance tests with `sudo` to ensure they audit the enterprise `qemu:///system` daemon.
