# Variable overrides: CIS, rhel_kvm, rhel_cockpit

An override is a variable we set in `group_vars` to change a role's default. This file lists every override and why it is there.

**Rule followed:** every CIS variable below exists in the CIS role's `defaults/` (checked). Only existing variables are set.

---

## 1. CIS: `cis_rhel9_host.yml` and `cis_rhel10_host.yml`

### How Level 1 is applied
The CIS roles run every rule by default. Their own `level_1` / `level_2` variables are only labels and do not switch rules on or off. So Level 1 is done the other way round: **every rule that the CIS benchmark marks as Level 2 (Server or Workstation) is set to `false`.**

The list of Level 2 rules comes from the CIS benchmark spreadsheets in `~/work-related/` (sheet `Combined Profiles`, column `Profile` filtered to `Level 2 - Server` and `Level 2 - Workstation`). These are the same editions the roles are built on: RHEL 9 Benchmark v2.0.0 and RHEL 10 Benchmark v1.0.1.

| | RHEL 9 | RHEL 10 |
| :--- | :--- | :--- |
| Level 2 recommendations in the spreadsheet (Server or Workstation) | 68 | 86 |
| Level 2 rules set to `false` in group_vars | 68 | 86 |
| Level 2 rules still enabled | 0 | 0 |
| Rules set to `false` in total | 71 | 89 |

The other 3 rules per OS that are set to `false` are the failed-login lockout rules (see below).

### Settings the role needs (not rules)
| Variable | Value | Why |
| :--- | :--- | :--- |
| `rhelNcis_dnsmasq_server` | `true` | CIS removes `dnsmasq` (rule 2.1.5 on RHEL 9, 2.1.6 on RHEL 10). libvirt needs it to give VMs IP addresses. |
| `rhelNcis_authselect_custom_profile_name` | `"rhel9"` / `"rhel10"` | The role stops with an error if this stays at its default `cis_example_profile`. |
| `rhelNcis_sshd_allowusers` | `"frqadmin"` | CIS writes `AllowUsers <value>` into `sshd_config`. Only this user can log in over SSH, so it must be the Ansible user. |
| `rhelNcis_dnsmasq_mask` | `false` | Same as the role default, so it changes nothing. |
| `rhel10cis_cockpit_server`, `rhel10cis_cockpit_mask` | `true` / `false` | Would keep Cockpit (rule 2.1.3); see the Cockpit note below. `mask: false` is the default. |

(`rhelN` = `rhel9` or `rhel10`.)

### Failed-login lockout is off on purpose
You asked for no lockout. Four rules per OS, all needed (the last one is a Level 2 rule):
- RHEL 9: `5_3_2_2`, `5_3_3_1_1`, `5_3_3_1_2`, `5_3_3_1_3`
- RHEL 10: `5_3_1_2`, `5_3_2_1_1`, `5_3_2_1_2`, `5_3_2_1_3`

### Cockpit
- **RHEL 9:** the RHEL 9 CIS role has no Cockpit rule, so nothing is needed (no `rhel9cis_cockpit_*` variables exist).
- **RHEL 10:** rule 2.1.3 removes Cockpit. It is a Level 2 rule, so it is already `false` in the Level 2 list, which is what keeps Cockpit. The two `rhel10cis_cockpit_*` lines are an extra layer.

### IP forwarding
Not overridden in CIS. CIS writes `ip_forward = 0` into `/etc/sysctl.d/60-netipv4_sysctl.conf`; `rhel_kvm` writes `1` into `99-kvm.conf`, which is read later and wins (checked on both hardened hosts: `1`).

---

## 2. rhel_kvm

One role override, in `kvm_hosts/kvm.yml`:

| Variable | Why |
| :--- | :--- |
| `rhel_kvm_storage_pools` | the storage pools the role creates, each with a `name` and a `path` |

Everything else (`kvm_br0` network, packages) uses the role's defaults. The OS-specific settings (monolithic vs modular) are in the role's `vars/RedHat-9.yml` and `RedHat-10.yml` and are deliberately **not** overridable from group_vars.

## 3. rhel_cockpit

No overrides. The role's defaults are used (`IdleTimeout = 15`, no extra packages).

## 4. The on/off switches

`enable_kvm` and `enable_cockpit` (in `all.yml`, set `true` in `kvm_hosts/`). See 01_INVENTORY_AND_GROUP_VARS.md.

---

## Check against the spreadsheets: no Level 2 rule is left enabled

Two independent checks give the same result: (1) each rule in group_vars compared with the `Combined Profiles` sheet filtered to Level 2; (2) the tasks of each CIS role that carry a Level 2 tag, compared with group_vars.

| | RHEL 9 | RHEL 10 |
| :--- | :--- | :--- |
| Level 2 rules still enabled | none | none |

On RHEL 10 the last two Level 2 rules were `3.3.1.1` (`net.ipv4.ip_forward`) and `6.3.1.1` (auditd packages installed). Both are now set to `false` (`rhel10cis_rule_3_3_1_1`, `rhel10cis_rule_6_3_1_1`).

With `3.3.1.1` off, CIS no longer writes `ip_forward = 0` on RHEL 10. `rhel_kvm` still sets `1` in `99-kvm.conf`, which stays needed for the built-in `default` NAT network. On RHEL 9 the equivalent rule is Level 1, so it stays enabled and `99-kvm.conf` still wins over it.
