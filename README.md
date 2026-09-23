# RHEL Hypervisor Master Orchestration

An enterprise master orchestration repository to provision, configure, and harden **RHEL 9** and **RHEL 10** (and binary-compatible derivatives such as AlmaLinux) into production-grade, **CIS Benchmark Level 1 Hardened KVM Hypervisors with Cockpit Web Management**.

---

## 🏗️ Architecture Overview

This orchestrator coordinates custom standalone roles alongside upstream Ansible Lockdown security baselines:

1. **`rhel_kvm`**: Provisions QEMU/KVM, manages version-specific Libvirt daemons (Monolithic `libvirtd` on RHEL 9, required, vs. Modular `virtqemud`/etc. on RHEL 10), sets up storage pools with SELinux `virt_image_t`, configures bridged networking, and enforces IP forwarding routing.
2. **`rhel_cockpit`**: Deploys the Cockpit Web Console with `cockpit-machines` on TCP port `9090`, enforces a headless server design (no desktop GUI), and configures systemd socket activation.
3. **`rhel9_cis` / `rhel10_cis`**: Applies 200+ CIS Benchmark Level 1 controls using pinned releases from `ansible-lockdown`, customized with surgical variable overrides so hypervisor routing and Cockpit web management remain 100% operational.

---

## 🗂️ Directory Layout

```text
rhel_hypervisor_master /
├── ansible.cfg                          # Inventory and role path configurations
│
├── playbooks/
│   └── main_playbook.yml                # Master 2-Phase Playbook + KVM Post-Tasks
│
├── sysconfig/
│   ├── inventory.yml                    # Hypervisors and CIS sub-groups
│   │
│   └── group_vars/                      # Strict separation of role vs CIS variables
│       ├── all.yml                      # Global execution feature toggles
│       │
│       ├── kvm_hosts/                   # Role variables for Provisioning
│       │   ├── kvm.yml                  # Storage pools, bridges, admin users
│       │   └── cockpit.yml              # Port 9090, timeout, security banners
│       │
│       ├── cis_rhel9_host.yml           # CIS Level 1 Overrides for RHEL 9
│       └── cis_rhel10_host.yml          # CIS Level 1 Overrides for RHEL 10
│
├── docs/                                # Technical documentation
│   ├── LESSONS_LEARNED_AND_FIXES.md     # Real-world troubleshooting & SSH Proxy guides
│   ├── ARCHITECTURE_AND_JUSTIFICATIONS.md
│   └── KNOWLEDGE_BASE_QA.md             # Architectural Q&A reference
│
└── roles/                               # Standalone roles pulled via ansible-galaxy
    ├── rhel_kvm
    ├── rhel_cockpit
    ├── rhel9_cis
    └── rhel10_cis
```

---

## ⚙️ Deployment Pipeline

Execution is managed deterministically via `playbooks/main_playbook.yml`:

1. **Phase 1: Infrastructure Provisioning (`hosts: kvm_hosts`)**
   - Pre-flight Ansible version check (>= 2.15).
   - Executes `rhel_kvm` and `rhel_cockpit`.

2. **Phase 2: CIS Benchmark Level 1 Security Hardening (`hosts: cis`)**
   - Evaluates OS distribution major version dynamically to run either `rhel9_cis` or `rhel10_cis`.
   - Protects critical hypervisor functions (IP forwarding, Cockpit port 9090) while remediating 200+ security controls.

3. **Post-Deployment KVM Configuration (`hosts: kvm_hosts`)**
   - Configures a custom warning Message of the Day (MOTD) for administrators.
   - Enforces password expiry for `root` via `chage -d 0 root` to require a mandatory credential reset upon first interactive login.

---

## 🚀 Quickstart & Execution

### 1. Install Galaxy Dependencies

Download all pinned roles and required collections into `roles/`:

```bash
ansible-galaxy install -r requirements.yml -p roles/ --force
```

### 2. Configure Target Inventory

Update IP addresses, jump host proxy configurations, and credentials in `sysconfig/inventory.yml`:

```yaml
kvm_hosts:
  hosts:
    rhel10_hypervisor:
      ansible_host: 192.168.20.30
      ansible_user: frqadmin
      ansible_ssh_common_args: "-o ProxyJump=frqadmin@192.168.10.160"

cis:
  children:
    cis_rhel10_host:
      hosts:
        rhel10_hypervisor:
```

### 3. Execute the Master Playbook

Run the playbook, passing the sudo password flag (`-K`):

```bash
ANSIBLE_ROLES_PATH=roles ANSIBLE_LOCAL_TEMP=.ansible/tmp ansible-playbook -i sysconfig/inventory.yml playbooks/main_playbook.yml -K
```

### 4. Verify Idempotency (Zero Unplanned Changes)

Re-running the playbook against already provisioned hosts should produce no changes:

```bash
# Expected: changed=0 failed=0
```

---

## 🔍 Post-Deployment Verification & Health Checks

Open your web browser and navigate to:

```
https://<HYPERVISOR_IP>:9090
```

Log in using your administrator credentials. The Cockpit dashboard will provide full web-based virtual machine management, and you should see the warning banner upon login.

---

## 📄 Documentation Links

- [docs/LESSONS_LEARNED_AND_FIXES.md](docs/LESSONS_LEARNED_AND_FIXES.md): Real-world troubleshooting, SSH ProxyJump guides, and hypervisor resilience notes.
- [docs/ARCHITECTURE_AND_JUSTIFICATIONS.md](docs/ARCHITECTURE_AND_JUSTIFICATIONS.md): Complete architecture guide and engineering justifications.
- [docs/KNOWLEDGE_BASE_QA.md](docs/KNOWLEDGE_BASE_QA.md): Detailed architectural Q&A covering variable precedence and inventory groups.

---

## License & Author

- **License**: Secret
- **Author**: Aeron (Trainee at AIRNAV)
