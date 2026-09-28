# The playbook: `playbooks/main_playbook.yml`

Three plays, run in order. The order matters: build the hypervisor first, then harden it.

| Play | Hosts | What it does |
| :--- | :--- | :--- |
| Phase 1 | `kvm_hosts` | installs KVM/libvirt and the Cockpit web console |
| Phase 2 | `cis` | applies the CIS Level 1 role for the host's OS |
| Post-deployment | `kvm_hosts` | forces a root password change |

---

## Phase 1: provision (`hosts: kvm_hosts`)

```yaml
become: true
gather_facts: true
```
- `become: true`: run as root through `sudo` (the login user is `frqadmin`, the tasks need root).
- `gather_facts: true`: collect facts about each host (OS name and version, addresses). The roles read them, for example `distribution_major_version` picks `RedHat-9.yml` or `RedHat-10.yml`.

**`pre_tasks`** (run before the roles):
- *Verify minimum Ansible version:* stops the run if the control machine has Ansible older than 2.15. `run_once: true` makes it run only once, not per host, because it checks the control machine.
- *Display Deployment start notification:* prints which host and OS is starting. Information only.

**`tasks`:**
```yaml
- name: Apply KVM Hypervisor Provisioning Role
  ansible.builtin.include_role:
    name: rhel_kvm
    apply:
      tags: [rhel_kvm]
  when: enable_kvm | bool
  tags: [rhel_kvm]
```
- `include_role`: runs the role at this point of the play.
- `when: enable_kvm | bool`: the on/off switch from group_vars (true for `kvm_hosts`).
- `tags: [rhel_kvm]` twice: the tag on the task lets `--tags rhel_kvm` select it; the tag under `apply:` copies the tag onto every task inside the role. Without `apply`, `--tags rhel_kvm` runs the include but none of the role's own tasks (lessons item 11).
- `rhel_cockpit` is the same with its own switch and tag. Order: KVM first, because Cockpit's VM page needs libvirt.

Run only one role: `ansible-playbook playbooks/main_playbook.yml --tags rhel_kvm --limit rhel9_hypervisor -K`. The CIS play is skipped because its `include_role` task has no tag.

---

## Phase 2: CIS hardening (`hosts: cis`)

```yaml
- name: Execute RHEL 9 CIS Benchmark Level 1 Hardening
  ansible.builtin.include_role:
    name: rhel9_cis
  when: ansible_facts['distribution_major_version'] == '9'
```
- Two tasks, one per OS. Each runs only on hosts whose major version matches, so a RHEL 9 host gets `rhel9_cis` and a RHEL 10 host gets `rhel10_cis`.
- The CIS variables come from `cis_rhel9_host.yml` / `cis_rhel10_host.yml` (see 02_VARIABLE_OVERRIDES.md).
- **Why after Phase 1:** the CIS roles change the firewall, kernel modules, packages and services. The hypervisor is built first and the hardening rules are shaped so they do not break it (dnsmasq kept, IP forwarding still `1`, Cockpit kept).

---

## Post-deployment (`hosts: kvm_hosts`, `gather_facts: false`)

- `gather_facts: false`: the tasks need no host facts, so this saves a step.
- **Force password expiry for root:** `chage -d 0 root`. Root must set a new password at the next interactive login. `changed_when: true` because the `command` module cannot tell whether anything changed.
- **Display success notice:** prints a message.

---

## Why it is built this way
- **Roles hold the logic, the playbook only orders them.** No package or service names appear in the playbook.
- **Variables are outside the playbook** (group_vars), so the same playbook works for RHEL 9 and RHEL 10.
- **`include_role` with `apply`** gives working tags, so one role can be re-run alone.

---

## Known limits (nothing changed)
- **Idempotency:** `chage -d 0 root` has `changed_when: true`, so the post-deployment play reports one change on every run, and it re-expires root's password each time the playbook is run. The KVM and Cockpit roles are idempotent; the CIS roles report some changes on every run by design.
- **`ansible-lint` is not clean yet** on our own code (6 findings): the variable `cockpit_packages` has no `rhel_cockpit_` prefix; a blank line at the end of `rhel_kvm/tasks/main.yml`; `state: latest` on the `redhat-release` task; a `shell` command containing a `|` in `preflight.yml`; a `file` task without `mode` in `storage.yml`; a missing final newline in `rhel_kvm/tests/inventory.yml`. Lint must exclude the two third-party CIS roles.
- **The OS decision is made twice:** by the inventory groups and again by `when:` in Phase 2.
- **"Right daemon model verified"** is done by hand (`systemctl`, `virsh`) today. The playbook has no automated check for it or for Cockpit after hardening.
- The "start notification", "success notice" and the `enable_*` switches do not change any result; they can be removed if you want a smaller playbook.
