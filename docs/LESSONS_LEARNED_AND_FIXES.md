# Lessons Learned & Fixes

Problems hit while building the CIS-hardened RHEL 9 / RHEL 10 KVM hypervisors, and how each was fixed. Each item: **Problem → Cause → Fix**.

| # | Problem (short) | Fix (short) |
| :- | :--- | :--- |
| 1 | CIS removes Cockpit / dnsmasq | Exempt them in `cis_rhelX_host.yml` |
| 2 | CIS sets `ip_forward = 0` | Our `99-kvm.conf` overrides it |
| 3 | Ansible hangs via jump host | `ssh-copy-id` through the jump host |
| 4 | Network/pool not autostarting | Separate "autostart" task |
| 5 | RHEL 10: GPG key error on install | Update `redhat-release` first |
| 6 | RHEL 9 libvirt setup | Keep it simple: enable and start `libvirtd` only |
| 7 | `virsh` shows nothing / false failures | Run checks with `sudo` or a login shell |
| 8 | dnf fails: `epel` has no baseurl | Fix the Zabbix role that created it |
| 9 | Clock wrong after VM rollback | `chronyc makestep` |
| 10 | Failed-login lockout | Disabled on purpose |
| 11 | `--tags` ran nothing | Add `apply: tags:` |
| 12 | Cockpit settings that did nothing | Removed; only `[Session] IdleTimeout` kept |

---

## 1. CIS Would Break the Hypervisor
**Cause:** CIS removes "unneeded" packages.
**Fix:**
- **dnsmasq** (both OSes): `rhelXcis_dnsmasq_server: true` in `cis_rhel9_host.yml` / `cis_rhel10_host.yml`. Libvirt needs `dnsmasq` to give VMs IPs.
- **Cockpit on RHEL 10:** the CIS rule that removes it (2.1.3) is switched off with `rhel10cis_rule_2_1_3: false`. `rhel10cis_cockpit_server: true` is also set but is redundant while that rule is off.
- **Cockpit on RHEL 9:** nothing to do. The RHEL 9 CIS role has no Cockpit rule at all (no `rhel9cis_cockpit_*` variables exist), so those two lines were removed from `cis_rhel9_host.yml`.

Only set variables that exist in the role's `defaults/`. Check with `grep -rn <variable> roles/<cis_role>/defaults`.

## 2. IP Forwarding
**Problem:** VMs need `net.ipv4.ip_forward = 1`; CIS writes `0` to `/etc/sysctl.d/60-netipv4_sysctl.conf`.
**Fix:** The `rhel_kvm` role writes `/etc/sysctl.d/99-kvm.conf`. Files load in order, so `99` beats `60`.
**Checked:** on both hardened hosts `sysctl net.ipv4.ip_forward` = `1`.

## 3. SSH Through the Jump Host
**Problem:** Ansible hangs on hosts behind the jump host.
**Fix:** Copy the key through it: `ssh-copy-id -o ProxyJump=frqadmin@192.168.10.160 frqadmin@<target-ip>`

## 4. Network / Pool Not Coming Back After Reboot
**Symptom:** Play succeeded, but after reboot `kvm_br0` showed `inactive` and `Autostart: no`.
**Cause:** In `community.libvirt.virt_net` / `virt_pool`, `autostart` is ignored when it is in the same task as `state`.
**Fix:** Three separate tasks: define (`present`) → start (`active`) → `autostart`. In `tasks/networks.yml` and `tasks/storage.yml`.

## 5. RHEL 10: GPG Signature Error
**Error:**
```
Failed to validate GPG signature for libvirt-daemon-log-11.10.0-12.4.el10_2.x86_64:
Public key for libvirt-daemon-log-...rpm is not installed
```
**Cause:** RHEL 10.1+ packages are signed with an extra post-quantum key that an older 10.1 image does not have. Ansible's `rpm_key` cannot import it (known Red Hat issue RHEL-126844), and importing the image's own key file changed nothing.
**Fix:** `tasks/packages.yml` runs `dnf update redhat-release` on RHEL 10 before installing packages (`rhel_kvm_update_redhat_release: true` in `vars/RedHat-10.yml`; `false` on RHEL 9). Tested on a fresh rollback: packages install.
**Notes:**
- `redhat-release` is a small package (OS identity files + Red Hat key files). Updating it changes the *reported* version (10.1 → 10.2), not the OS or kernel.
- Why it works is not fully explained. It is confirmed by testing only.
- Not the clock (item 9) and not a `failed_when` guard.
- RHEL 9 once showed `package ... is already installed` (Transaction test error) on an already-provisioned host. Cause never found; it did not come back on a fresh host or a re-run.

## 6. RHEL 9: Keep libvirt Simple
**Decision:** RHEL 9 runs `libvirtd`. The role only enables and starts `libvirtd.service`. It does not mask the other libvirt daemons, does not manage sockets and does not pin a connection address (mentor advice: keep it simple).
**Tried and removed:** masking the modular daemons, an explicit `qemu+unix` connection address, and a socket-ordering check. Each brought its own errors: `Failed to connect socket to '/var/run/libvirt/virtqemud-sock': Connection refused`, `Unable to start service libvirtd.socket: Job failed` (`already active, refusing`), and `Failed to connect socket to '/var/run/libvirt/libvirt-sock': No such file or directory`.
**Why the two designs can not run together:** in the libvirt unit files the modular units say `Conflicts=libvirtd.service` (`virtqemud.service`, `virtnetworkd.service`, `virtstoraged.service`) and `Conflicts=libvirtd.socket` (their sockets). Starting a modular daemon therefore stops `libvirtd`. This explains the earlier symptom (libvirtd inactive while the modular daemons ran). It matters on RHEL 9 only if the OS enables the modular sockets by default, so that a client connection starts a modular daemon. On a fresh host, check before running the role: `systemctl is-enabled virtqemud.socket virtnetworkd.socket virtstoraged.socket`. If they are `disabled`, nothing conflicts. If `enabled`, `libvirtd` can be stopped and the modular units need to be masked again. (Unit files read on Fedora 44, libvirt 12.0; not yet read on RHEL.)
**Check with:** `systemctl is-active libvirtd` and `virsh uri`.

## 7. False Failures When Checking by Hand
Run checks with `sudo`. Without root, `virsh` may query `qemu:///session` (empty) and `ss` / `firewall-cmd` hide details. Use `sudo virsh ...` or `virsh -c qemu:///system ...`.

## 8. dnf Error: `Cannot find a valid baseurl for repo: epel`
**Error:**
```
Cannot find a valid baseurl for repo: epel
```
(at "Install core KVM hypervisor packages", although the host had internet)
**Cause:** `/etc/yum.repos.d/epel.repo` contained only `[epel]` and `excludepkgs = zabbix*`. dnf refreshes *every* enabled repo, so it failed. The Zabbix role created that file: `ini_file` creates missing files by default.
**Fix:** In `zabbix_agent_deploy/tasks/repo_setup.yml` (and its vendored copy): check the file exists first (`stat`), use `create: false`, and drop `ignore_errors`.
**On an affected host:** `sudo rm /etc/yum.repos.d/epel.repo` (unless you want real EPEL; then move the stub aside and install `epel-release` with `--disablerepo=epel`).

## 9. Wrong Clock After Snapshot Rollback
**Symptom:** `subscription-manager` warns the clock is skewed; `timedatectl` says `System clock synchronized: no`.
**Cause:** A rollback restores the old time. chrony's default `makestep 1.0 3` only jumps the clock in its first 3 updates.
**Fix:** `sudo chronyc makestep`, then check `timedatectl`.
**Lab only:** set `makestep 1.0 -1` in `/etc/chrony.conf` so it always jumps (take a new snapshot after). CIS rewrites this to `1.0 3`; set `rhel9cis_chrony_server_makestep` / `rhel10cis_chrony_server_makestep` if it must survive hardening.

## 10. Failed-Login Lockout (`pam_faillock`) Disabled on Purpose
Set to `false` in `cis_rhel9_host.yml` / `cis_rhel10_host.yml`. All four rules are needed per OS, or lockout stays on for normal users:

| OS | Rules |
| :--- | :--- |
| RHEL 9 | `rhel9cis_rule_5_3_2_2`, `5_3_3_1_1`, `5_3_3_1_2`, `5_3_3_1_3` |
| RHEL 10 | `rhel10cis_rule_5_3_1_2`, `5_3_2_1_1`, `5_3_2_1_2`, `5_3_2_1_3` |

## 11. `--tags` Ran Nothing
**Symptom:** `--tags rhel_kvm` finished `ok=3 changed=0`; no role task ran.
**Cause:** A tag on a dynamic `include_role` tags only the include, not the tasks inside.
**Fix:** In `playbooks/main_playbook.yml`, add `apply: tags:` under `include_role` (done for both roles).
**Use:** `ansible-playbook playbooks/main_playbook.yml --tags rhel_kvm --limit rhel9_hypervisor -K`

## 12. `rhel_cockpit`: Settings That Did Nothing (Removed)
Checked against `man cockpit.conf`. The role wrote options Cockpit ignores or that were already its defaults:

| Removed | Why |
| :--- | :--- |
| `Port = 9090` | The port cannot be set in `cockpit.conf` (only via `cockpit.socket`); 9090 is the default anyway |
| `Banner = <text>` (in `[WebService]`) | `Banner` is a `[Session]` option and must be a *file path*, so no banner was ever shown. Not needed. |
| `IdleTimeout` under `[WebService]` | It only works under `[Session]` (that copy is kept) |
| `AllowUnencrypted = false` | Already the default; it was tied to the wrong variable |
| Remove `root` from `disallowed-users` | Loosened security for no stated need. Log in as an admin user. |
| Remove `virt-manager` | Not shipped in RHEL 9/10, so the task did nothing |
| `/etc/cockpit` directory task, `Reload firewalld` handler, `failed_when: false`, variables for constants (socket name/state), the `manage_config` switch | Redundant |
| Firewalld task (`cockpit` service in the `public` zone) | Already allowed by default: `firewall-cmd --permanent --zone=public --list-services` = `cockpit dhcpv6-client ssh` on stock RHEL 9 and 10; CIS roles do not remove services |

Kept: install `cockpit` + `cockpit-machines`, enable `cockpit.socket`, `[Session] IdleTimeout = 15` (CIS). Final check pending: after a full play with CIS, `https://<host>:9090` must still open from another VM; if not, the firewall task goes back.
**After this change** the first re-run shows `changed` once on the config task (the file content changed) and restarts `cockpit.socket`. Hosts provisioned before keep `root` removed from `disallowed-users` until you restore it: `echo root | sudo tee -a /etc/cockpit/disallowed-users`.
