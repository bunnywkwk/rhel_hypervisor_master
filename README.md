
# RHEL Hypervisor Master Orchestration


An enterprise master orchestration repository to provision, configure, and harden **RHEL 9** and **RHEL 10** (and binary-compatible derivatives such as AlmaLinux) into production-grade, **CIS Benchmark Level 1 Hardened KVM Hypervisors with Cockpit Web Management**.

---

## 🏗️ Architecture & Requirements
## Architecture Overview

This repository pulls two standalone custom roles alongside the official Ansible Lockdown CIS benchmark roles:
This orchestrator coordinates standalone custom roles alongside upstream Ansible Lockdown security baselines:

1. **`rhel_kvm`**: Provisions QEMU/KVM, manages the version-specific daemon model (Monolithic on RHEL 9 vs. Modular on RHEL 10), configures storage pools (`/var/lib/libvirt/images`) with SELinux `virt_image_t`, enables `net.ipv4.ip_forward = 1`, and starts the default network bridge (`virbr0`).
2. **`rhel_cockpit`**: Deploys the Cockpit Web Console with `cockpit-machines` on TCP port `9090`, enforces headless server design (no GUI / no `virt-manager`), and manages systemd socket activation (`cockpit.socket`).
3. **`rhel9_cis` / `rhel10_cis`**: Applies 200+ CIS Benchmark Level 1 controls using pinned releases from `ansible-lockdown`, customized with variable overrides so KVM and Cockpit remain 100% operational.
1. **`rhel_kvm`**: Provisions QEMU/KVM, manages version-specific Libvirt daemons (Monolithic on RHEL 9 vs. Modular on RHEL 10), sets up storage pools (`/var/lib/libvirt/images`) with SELinux `virt_image_t`, configures bridged networking, and enables `net.ipv4.ip_forward = 1`.
2. **`rhel_cockpit`**: Deploys the Cockpit Web Console with `cockpit-machines` on TCP port `9090`, enforces a headless server design (zero desktop GUI / no `virt-manager`), and configures systemd socket activation (`cockpit.socket`).
3. **`rhel9_cis` / `rhel10_cis`**: Applies 200+ CIS Benchmark Level 1 controls using pinned releases from `ansible-lockdown`, customized with surgical variable overrides so hypervisor routing and Cockpit web management remain 100% operational.
4. **Acceptance Test Suite**: Deploys and executes automated Python validation tools (`verify_hypervisor.py` and `verify_cockpit.py`) in Phase 3 to assert post-hardening compliance.

---

## 🗂️ Project Directory Layout
## Directory Layout

```
rhel_hypervisor_master /
├── ansible.cfg                          # Inventory and role path configurations
├── requirements.yml                     # Pinned Galaxy roles (rhel_kvm, rhel_cockpit, CIS roles)
rhel_hypervisor_master/
├── ansible.cfg                          # Central Ansible engine configuration
├── requirements.yml                     # Pinned Galaxy roles and required collections
├── .gitignore                           # Standard gitignore (temp files, caches, retries)
│
├── playbooks/
│   └── site.yml                         # Master 3-Phase Playbook (KVM -> Cockpit -> CIS -> Verify)
│   └── main_playbook.yml                # Master 3-Phase Playbook (Provision -> Harden -> Verify)
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
│   ├── group_vars/                      # Strict separation of role vs CIS variables
│   │   ├── all.yml                      # Global execution feature toggles
│   │   │
│   │   ├── kvm_hosts/                   # Role variables for Provisioning
│   │   │   ├── kvm.yml                  # Storage pools, bridges, admin users
│   │   │   └── cockpit.yml              # Port 9090, timeout, security banners
│   │   │
│   │   ├── cis_rhel9_host.yml           # CIS Level 1 Overrides for RHEL 9
│   │   └── cis_rhel10_host.yml          # CIS Level 1 Overrides for RHEL 10
│   │
│   └── host_vars/                       # Host-specific overrides (NICs, disk paths)
│       ├── rhel9_hypervisor.yml
│       └── rhel10_hypervisor.yml
│
├── docs/                                # Knowledge Base & Technical Q&A
│   ├── MASTER_ARCHITECTURE_AND_CHECKLIST.md
│   └── KNOWLEDGE_BASE_QA.md
├── docs/                                # Technical documentation
│   ├── ARCHITECTURE_AND_JUSTIFICATIONS.md # Master architecture and design justifications
│   └── KNOWLEDGE_BASE_QA.md             # Architectural Q&A reference
│
└── roles/                               # Roles downloaded via ansible-galaxy
└── roles/                               # Standalone roles pulled via ansible-galaxy
    ├── rhel_kvm
    ├── rhel_cockpit
    ├── rhel9_cis
    └── rhel10_cis
```

---

## ⚙️ Strict Variable Separation
## Three-Phase Deployment Pipeline

Execution is managed deterministically via `playbooks/main_playbook.yml`:

1. **Phase 1: Infrastructure Provisioning (`hosts: kvm_hosts`)**
   - Pre-flight Ansible version check (>= 2.15).
   - Executes `rhel_kvm`: Hardware virtualization drivers, Libvirt daemon lifecycle, storage pools, bridge networking, and sysctl tuning.
   - Executes `rhel_cockpit`: Headless Cockpit web console, VM management module, and systemd socket activation.

2. **Phase 2: CIS Benchmark Level 1 Hardening (`hosts: cis`)**
   - Evaluates OS distribution major version dynamically:
     - On RHEL / AlmaLinux 9: Runs `rhel9_cis` with overrides from `sysconfig/group_vars/cis_rhel9_host.yml`.
     - On RHEL / AlmaLinux 10: Runs `rhel10_cis` with overrides from `sysconfig/group_vars/cis_rhel10_host.yml`.
   - Protects critical hypervisor functions (IP forwarding, KVM kernel drivers, Cockpit port 9090) while remediating 200+ security controls.

3. **Phase 3: Post-Deployment Verification & Security (`hosts: kvm_hosts`)**
   - Runs `/usr/local/bin/verify_hypervisor.py` to assert hypervisor daemon, kernel, storage, and bridge health.
   - Runs `/usr/local/bin/verify_cockpit.py` to assert Cockpit socket, port 9090, timeout, and HTTPS health.
   - Enforces password expiry for `root` via `chage -d 0 root` to require a mandatory credential reset upon first interactive login.

---

## Strict Variable Separation

As mandated by enterprise security standards, **CIS Hardening variables are kept strictly separated from role provisioning variables**:

- **Global Toggles**: Defined in `sysconfig/group_vars/all.yml` (`enable_kvm: false`, `enable_cockpit: false`).
- **Role Provisioning Variables**: Defined in `sysconfig/group_vars/kvm_hosts/kvm.yml` and `cockpit.yml`.
- **CIS Hardening Overrides**: Defined in `sysconfig/group_vars/cis_rhel9_hosts.yml` and `cis_rhel10_hosts.yml`.
- **Global Toggles**: Defined in `sysconfig/group_vars/all.yml`.
- **CIS Hardening Overrides**: Defined in `sysconfig/group_vars/cis_rhel9_host.yml` and `cis_rhel10_host.yml`.

This prevents variable collisions, ensures custom roles remain modular and portable, and gives security auditors a single, clear location to review compliance tailoring.

---

## 🚀 Quickstart & Execution
## Quickstart & Execution

### 1. Install Galaxy Dependencies

Pull all pinned roles and required collections:
Download all pinned roles and required collections into `roles/`:

```bash
ansible-galaxy install -r requirements.yml -p roles/ --force
```

### 2. Configure Target Inventory

Update IP addresses in `sysconfig/inventory.yml`:
Update IP addresses and credentials in `sysconfig/inventory.yml`:

```yaml
kvm_hosts:
  hosts:
    rhel9_hypervisor:
      ansible_host: 192.168.122.173
      ansible_user: root
    rhel10_hypervisor:
      ansible_host: 192.168.122.153
      ansible_user: root

cis:
  children:
    cis_rhel9_host:
      hosts:
        rhel9_hypervisor:
    cis_rhel10_host:
      hosts:
        rhel10_hypervisor:
```

### 3. Execute the Master Playbook
### 3. Verify Playbook Syntax

```bash
ansible-playbook -i sysconfig/inventory.yml playbooks/site.yml
ansible-playbook --syntax-check playbooks/main_playbook.yml
```

### 4. Verify Idempotency (Zero Changes)
### 4. Execute the Master Playbook

```bash
ansible-playbook -i sysconfig/inventory.yml playbooks/site.yml
ansible-playbook playbooks/main_playbook.yml
```

### 5. Verify Idempotency (Zero Unplanned Changes)

Re-running the playbook against already provisioned hosts should produce no changes:

```bash
ansible-playbook playbooks/main_playbook.yml
# Expected: changed=0 failed=0
```

---

## 🔍 Verification & Health Checks
## Post-Deployment Verification & Health Checks

1. **Automated 1-Click Verification**:
   SSH into any provisioned host and execute:
   ```bash
   /usr/local/bin/verify_hypervisor.py
   ```
2. **Cockpit Web Access**:
   Navigate to `https://<HYPERVISOR_IP>:9090` in your browser and log in with your credentials.
### 1. Automated Hypervisor Acceptance Check
Log in to any provisioned host and execute:
```bash
/usr/local/bin/verify_hypervisor.py
```
Expected output: 100% PASS for OS detection, daemons, kernel modules, sysctl, storage pools, and network bridges.

### 2. Automated Cockpit Acceptance Check
Log in to any provisioned host and execute:
```bash
/usr/local/bin/verify_cockpit.py
```
Expected output: 100% PASS for packages, port 9090 socket listening, session idle timeout, firewalld service, and HTTPS handshake.

### 3. Cockpit Web Access
Open your web browser and navigate to:
```
https://<HYPERVISOR_IP>:9090
```
Log in using your administrator credentials. The Cockpit dashboard will provide full web-based virtual machine management via `cockpit-machines`.

---

## 📄 License & Author
## Documentation Links

- [docs/LESSONS_LEARNED_AND_FIXES.md](docs/LESSONS_LEARNED_AND_FIXES.md): Real-world troubleshooting, SSH ProxyJump guides, and hypervisor resilience notes.
- [docs/ARCHITECTURE_AND_JUSTIFICATIONS.md](docs/ARCHITECTURE_AND_JUSTIFICATIONS.md): Complete architecture guide, Old vs. New comparison, and engineering justifications.
- [docs/KNOWLEDGE_BASE_QA.md](docs/KNOWLEDGE_BASE_QA.md): Detailed architectural Q&A covering variable precedence, inventory groups, and CIS rules.

---

## License & Author

- **License**: MIT
- **Author**: Aeron (Trainee at AIRNAV)
