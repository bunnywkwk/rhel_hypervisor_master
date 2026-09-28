# Inventory and group_vars

## 1. The inventory (`sysconfig/inventory.yml`)

The inventory says **which machines exist** and **which group each one belongs to**.

```yaml
kvm_hosts:                      # every hypervisor
  hosts:
    rhel9_hypervisor:   { ansible_host: 192.168.20.30, ansible_user: frqadmin, ansible_ssh_common_args: "-o ProxyJump=frqadmin@192.168.10.160" }
    rhel10_hypervisor:  { ansible_host: 192.168.20.40, ... same user and ProxyJump ... }

cis:                            # hosts to harden
  children:
    cis_rhel9_host:   { hosts: rhel9_hypervisor }
    cis_rhel10_host:  { hosts: rhel10_hypervisor }
```

| Group | Who is in it | Used by |
| :--- | :--- | :--- |
| `kvm_hosts` | both hypervisors | Phase 1 (KVM + Cockpit) and the post-deployment play |
| `cis` | both, through its two children | Phase 2 (CIS hardening) |
| `cis_rhel9_host` | `rhel9_hypervisor` | RHEL 9 CIS variables |
| `cis_rhel10_host` | `rhel10_hypervisor` | RHEL 10 CIS variables |

- **Why the OS groups exist:** CIS variables are different for RHEL 9 (`rhel9cis_*`) and RHEL 10 (`rhel10cis_*`). One group per OS lets each group have its own variable file (see section 2).
- **`ansible_host`:** the IP Ansible connects to. **`ansible_user`:** the login user. **`ansible_ssh_common_args` with `ProxyJump`:** the hypervisors are on a private network, so SSH goes through the jump host `192.168.10.160`.
- **One-time setup:** the SSH key must be copied *through* the jump host: `ssh-copy-id -o ProxyJump=frqadmin@192.168.10.160 frqadmin@<target-ip>` (lessons item 3).
- A host can be in several groups. Both hypervisors are in `kvm_hosts` and in one OS group, so they get variables from both.

---

## 2. group_vars: where the settings live

Ansible automatically loads the file whose name matches a group, from `sysconfig/group_vars/`. Every host in that group receives those variables.

| File | Group | What it holds |
| :--- | :--- | :--- |
| `all.yml` | every host | `enable_kvm: false`, `enable_cockpit: false` (both off by default) |
| `kvm_hosts/kvm.yml` | `kvm_hosts` | `enable_kvm: true` and the storage pools to create (`rhel_kvm_storage_pools`) |
| `kvm_hosts/cockpit.yml` | `kvm_hosts` | `enable_cockpit: true` |
| `cis_rhel9_host.yml` | `cis_rhel9_host` | CIS overrides for RHEL 9 only |
| `cis_rhel10_host.yml` | `cis_rhel10_host` | CIS overrides for RHEL 10 only |

`sysconfig/host_vars/` has one empty file per host. It is not used.

### How `enable_kvm` and `enable_cockpit` work
`all.yml` sets them to `false`. `kvm_hosts/*.yml` sets them to `true`. A more specific group beats `all`, so every host in `kvm_hosts` gets `true`. The playbook has `when: enable_kvm | bool` on the role. To leave a role out for a group, you change one variable, not the playbook.

### Why this is done with group_vars (and why it matters)
1. **CIS variables are kept apart from role variables.** This is a requirement of the task. CIS settings are in `cis_*.yml`; `rhel_kvm_*` settings are in `kvm_hosts/`. Someone reviewing security opens only the CIS files.
2. **Nothing hard-coded in roles or the playbook.** The roles only carry defaults, so they stay reusable in other projects. Site-specific choices live here.
3. **A new host needs no code.** Put a RHEL 9 host in `cis_rhel9_host` and it receives the RHEL 9 CIS settings automatically.
4. **One place per concern.** To change what is hardened you edit one file; the playbook does not change.

### Which value wins
From weakest to strongest: role `defaults/` → `group_vars/all` → `group_vars/<specific group>` → `host_vars` → role `vars/` → `-e` on the command line. So `group_vars` override a role's defaults, but not a role's protected `vars/` (that is why the package lists are in `vars/`).

---

## Known limits (nothing changed)
- `enable_kvm` and `enable_cockpit` are `true` for every host in `kvm_hosts`, so today they do not change any result.
- The inventory repeats `ansible_user` and the `ProxyJump` line for each host.
- The `cis` parent group is only used by Phase 2; the OS decision is also made again by `when:` in the playbook (see 04_PLAYBOOK_WALKTHROUGH.md).
