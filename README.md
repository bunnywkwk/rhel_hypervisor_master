# RHEL Hypervisor Master

Ansible project that turns a bare **RHEL 9** or **RHEL 10** machine into a hardened **KVM hypervisor with the Cockpit web console**.

It does not contain the automation logic itself. It orders four roles, each pulled from its own repository, and holds the settings for them:

| Role | What it does | Source |
| :--- | :--- | :--- |
| `rhel_kvm` | KVM and libvirt: monolithic `libvirtd` on RHEL 9, modular daemons on RHEL 10, storage pool, isolated network `kvm_br0`, IP forwarding | `bunnywkwk/rhel_kvm` |
| `rhel_cockpit` | Cockpit web console with VM management on TCP 9090, session idle timeout | `bunnywkwk/rhel_cockpit` |
| `rhel9_cis` | CIS Benchmark for RHEL 9, Level 1 Server | `ansible-lockdown/RHEL9-CIS`, pinned to a tag |
| `rhel10_cis` | CIS Benchmark for RHEL 10, Level 1 Server | `ansible-lockdown/RHEL10-CIS`, pinned to a tag |

---

## Repository layout

```text
.
├── ansible.cfg                      # inventory and roles path
├── requirements.yml                 # roles and collections to install
├── playbooks/
│   └── main_playbook.yml            # the three plays
├── sysconfig/
│   ├── inventory.yml                # hosts and groups
│   └── group_vars/
│       ├── all.yml                  # enable_kvm / enable_cockpit (default off)
│       ├── kvm_hosts/               # settings for the KVM and Cockpit roles
│       ├── cis_rhel9_host.yml       # CIS settings, RHEL 9 only
│       └── cis_rhel10_host.yml      # CIS settings, RHEL 10 only
├── docs/                            # documentation (see the end of this file)
└── roles/                           # installed by ansible-galaxy, not stored in git
```

CIS variables are kept apart from the role variables: they live in the `cis_*` files only.

---

## What the playbook does

`playbooks/main_playbook.yml` has three plays, run in this order:

1. **Phase 1, provision** (`kvm_hosts`): checks the Ansible version, then runs `rhel_kvm` and `rhel_cockpit`.
2. **Phase 2, harden** (`cis`): runs `rhel9_cis` on RHEL 9 hosts and `rhel10_cis` on RHEL 10 hosts.
3. **Post-deployment** (`kvm_hosts`): expires the `root` password (`chage -d 0 root`), so root must set a new one at the next interactive login.

The hypervisor is built first and then hardened. The CIS settings are chosen so hardening does not break it: `dnsmasq` is kept (libvirt needs it), IP forwarding stays on, and Cockpit stays installed.

---

## Requirements

- Ansible 2.15 or newer on the control machine.
- SSH access to the hosts as `frqadmin` through the jump host `192.168.10.160`. Copy the key through the jump host once:
  ```bash
  ssh-copy-id -o ProxyJump=frqadmin@192.168.10.160 frqadmin@<host-ip>
  ```
- `sudo` rights for `frqadmin` on the hosts (the playbook is run with `-K`).
- Hosts with a correct clock. After a VM snapshot rollback, check that `timedatectl` shows `System clock synchronized: yes` (docs/06_VM_ROLLBACK_AND_CLOCK.md).

---

## Usage

### 1. Install the roles and collections

```bash
ansible-galaxy install -r requirements.yml -p roles/ --force
```

This downloads the four roles into `roles/` and the required collections. It overwrites what is in `roles/`, so push any change to `rhel_kvm` or `rhel_cockpit` to GitHub first.

### 2. Check the inventory

Hosts, IP addresses and the SSH jump host are in `sysconfig/inventory.yml`. The two hypervisors are `rhel9_hypervisor` and `rhel10_hypervisor`.

### 3. Run the playbook

```bash
ANSIBLE_ROLES_PATH=roles ANSIBLE_LOCAL_TEMP=.ansible/tmp ansible-playbook -i sysconfig/inventory.yml playbooks/main_playbook.yml -K
```

`-K` asks for the sudo password.

### Run one role, or one host

```bash
ansible-playbook playbooks/main_playbook.yml --tags rhel_kvm --limit rhel9_hypervisor -K
```

Tags: `rhel_kvm` and `rhel_cockpit`. The CIS plays are skipped when a tag is given. Without `--limit`, both hosts are used.

### Re-running

Running the playbook again on a provisioned host should report `changed=0` for the `rhel_kvm` and `rhel_cockpit` tasks. The CIS roles and the root-expiry task report some changes on every run.

---

## Check the result

Run these on a hypervisor after the playbook has finished:

```bash
# RHEL 9 runs libvirtd
systemctl is-active libvirtd

# RHEL 10 runs the modular daemons
systemctl is-active virtqemud.socket

# Both: pool and network up with autostart, IP forwarding on
sudo virsh pool-list --all
sudo virsh net-list --all
sysctl net.ipv4.ip_forward                       # expect: 1
```

Cockpit: open `https://<host-ip>:9090` and log in with an administrator account. Idle sessions are logged out after 15 minutes.

---

## Documentation

Project documents (in `docs/`):

- [01_INVENTORY_AND_GROUP_VARS.md](docs/01_INVENTORY_AND_GROUP_VARS.md): the inventory groups and what each group_vars file holds.
- [02_VARIABLE_OVERRIDES.md](docs/02_VARIABLE_OVERRIDES.md): every CIS, `rhel_kvm` and `rhel_cockpit` override and why.
- [03_REQUIREMENTS_AND_PINNING.md](docs/03_REQUIREMENTS_AND_PINNING.md): what `requirements.yml` pulls and why it is pinned.
- [04_PLAYBOOK_WALKTHROUGH.md](docs/04_PLAYBOOK_WALKTHROUGH.md): the playbook, play by play.
- [05_TEST_VM_FROM_QCOW2.md](docs/05_TEST_VM_FROM_QCOW2.md): copy a qcow2 image to a hypervisor, import it in Cockpit and boot a test VM (copy-paste commands).
- [06_VM_ROLLBACK_AND_CLOCK.md](docs/06_VM_ROLLBACK_AND_CLOCK.md): fix the clock after a VM snapshot rollback (setup before the snapshot, and the commands to run after).
- [LESSONS_LEARNED_AND_FIXES.md](docs/LESSONS_LEARNED_AND_FIXES.md): problems met while building this and how each was fixed.
- [ARCHITECTURE_AND_JUSTIFICATIONS.md](docs/ARCHITECTURE_AND_JUSTIFICATIONS.md) and [KNOWLEDGE_BASE_QA.md](docs/KNOWLEDGE_BASE_QA.md): longer background and Q&A.

Each role has its own README and a `docs/TASK_WALKTHROUGH.md` in its repository.

---

## License and author

- **License**: Secret
- **Author**: Aeron (Trainee at AIRNAV)
