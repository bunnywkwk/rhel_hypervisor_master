# RHEL Hypervisor Master Orchestration Playbook

An enterprise master orchestration repository to provision and harden **RHEL 9** and **RHEL 10** bare-metal or virtual machines into production-ready, **CIS Benchmark Level 1 Hardened KVM Hypervisors with Cockpit Web Management**.

---

## 🏗️ Architecture & Requirements

This repository pulls two standalone custom roles alongside the official Ansible Lockdown CIS benchmark roles:

1. **`rhel_kvm`**: Provisions QEMU/KVM, manages the version-specific daemon model (Monolithic on RHEL 9 vs. Modular on RHEL 10), configures storage pools (`/var/lib/libvirt/images`) with SELinux `virt_image_t`, enables `net.ipv4.ip_forward = 1`, and starts the default network bridge (`virbr0`).
2. **`rhel_cockpit`**: Deploys the Cockpit Web Console with `cockpit-machines` on TCP port `9090`, enforces headless server design (no GUI / no `virt-manager`), and manages systemd socket activation (`cockpit.socket`).
3. **`rhel9_cis` / `rhel10_cis`**: Applies 200+ CIS Benchmark Level 1 controls using pinned releases from `ansible-lockdown`, customized with variable overrides so KVM and Cockpit remain 100% operational.

---

## 🗂️ Project Directory Layout

```
rhel_hypervisor_master /
├── ansible.cfg                          # Inventory and role path configurations
├── requirements.yml                     # Pinned Galaxy roles (rhel_kvm, rhel_cockpit, CIS roles)
│
├── playbooks/
│   └── site.yml                         # Master 3-Phase Playbook (KVM -> Cockpit -> CIS -> Verify)
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
│       ├── cis_rhel9_hosts.yml          # CIS Level 1 Overrides for RHEL 9
│       └── cis_rhel10_hosts.yml         # CIS Level 1 Overrides for RHEL 10
│
├── docs/                                # Knowledge Base & Technical Q&A
│   ├── MASTER_ARCHITECTURE_AND_CHECKLIST.md
│   └── KNOWLEDGE_BASE_QA.md
│
└── roles/                               # Roles downloaded via ansible-galaxy
```

---

## ⚙️ Strict Variable Separation

As mandated by enterprise security standards, **CIS Hardening variables are kept strictly separated from role provisioning variables**:

- **Role Provisioning Variables**: Defined in `sysconfig/group_vars/kvm_hosts/kvm.yml` and `cockpit.yml`.
- **CIS Hardening Overrides**: Defined in `sysconfig/group_vars/cis_rhel9_hosts.yml` and `cis_rhel10_hosts.yml`.
- **Global Toggles**: Defined in `sysconfig/group_vars/all.yml`.

---

## 🚀 Quickstart & Execution

### 1. Install Galaxy Dependencies

Pull all pinned roles and required collections:

```bash
ansible-galaxy install -r requirements.yml -p roles/ --force
```

### 2. Configure Target Inventory

Update IP addresses in `sysconfig/inventory.yml`:

```yaml
kvm_hosts:
  hosts:
    rhel9_hypervisor:
      ansible_host: 192.168.122.173
      ansible_user: root
    rhel10_hypervisor:
      ansible_host: 192.168.122.153
      ansible_user: root
```

### 3. Execute the Master Playbook

```bash
ansible-playbook -i sysconfig/inventory.yml playbooks/site.yml
```

### 4. Verify Idempotency (Zero Changes)

```bash
ansible-playbook -i sysconfig/inventory.yml playbooks/site.yml
# Expected: changed=0 failed=0
```

---

## 🔍 Verification & Health Checks

1. **Automated 1-Click Verification**:
   SSH into any provisioned host and execute:
   ```bash
   /usr/local/bin/verify_hypervisor.py
   ```
2. **Cockpit Web Access**:
   Navigate to `https://<HYPERVISOR_IP>:9090` in your browser and log in with your credentials.

---

## 📄 License & Author

- **License**: MIT
- **Author**: Aeron (Trainee at AIRNAV)
