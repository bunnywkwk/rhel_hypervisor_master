# Master Orchestration - Knowledge Base & Q&A Reference

This document provides deep-dive explanations and answers to core architectural questions regarding the Master Playbook, inventory structure, variable precedence, and CIS Level 1 integration.

---

## ❓ Core Q&A Reference
## Core Q&A Reference

### Q1: What does "Keep CIS variables separate from role variables" mean?

- **A**:
  - **Role Variables (`kvm.yml`, `cockpit.yml`)**: Configure our custom infrastructure roles (`rhel_kvm`, `rhel_cockpit`). E.g., `rhel_kvm_storage_pools`, `rhel_cockpit_port: 9090`.
  - **CIS Variables (`cis_rhel9_hosts.yml`, `cis_rhel10_hosts.yml`)**: Configure the 3rd-party Ansible Lockdown security baseline (`rhel9cis_*`, `rhel10cis_*`). E.g., `rhel9cis_level_1: true`, `rhel9cis_rule_3_1_1: false`.
  - **CIS Variables (`cis_rhel9_host.yml`, `cis_rhel10_host.yml`)**: Configure the 3rd-party Ansible Lockdown security baseline (`rhel9cis_*`, `rhel10cis_*`). E.g., `rhel9cis_level_1: true`, `rhel9cis_rule_3_1_1: false`.
  - **Why separate them**:
    1. **Portability**: Our custom roles (`rhel_kvm`, `rhel_cockpit`) remain pure, standalone, and reusable anywhere without dragging CIS variables along.
    2. **Auditor Clarity**: Security auditors can inspect `cis_rhel9_hosts.yml` directly to see all tailored security rules in one clean place without wading through infrastructure disk paths.
    2. **Auditor Clarity**: Security auditors can inspect `cis_rhel9_host.yml` directly to see all tailored security rules in one clean place without wading through infrastructure disk paths.
    3. **Safety**: Changing a hypervisor disk path will never accidentally alter a security benchmark setting.

---

### Q2: Why does `cis` have `children` (`cis_rhel9_hosts`, `cis_rhel10_hosts`) while `kvm_hosts` does not?
### Q2: Why does `cis` have `children` (`cis_rhel9_host`, `cis_rhel10_host`) while `kvm_hosts` does not?

- **A**:
  - **`kvm_hosts` (Uniform Setup)**: Both RHEL 9 and RHEL 10 execute the exact same hypervisor provisioning (both install KVM, both install Cockpit, both use `/var/lib/libvirt/images`, both use `virbr0`). They share a single group.
  - **`cis` (Version-Specific Security Rules)**: RHEL 9 uses `rhel9_cis` role with `rhel9cis_*` variables, while RHEL 10 uses `rhel10_cis` with `rhel10cis_*` variables.
  - Creating child sub-groups (`cis_rhel9_hosts` and `cis_rhel10_hosts`) allows Ansible to automatically load `cis_rhel9_hosts.yml` for RHEL 9 and `cis_rhel10_hosts.yml` for RHEL 10, while still allowing the playbook to target `hosts: cis` to run hardening across all machines simultaneously.
  - **`kvm_hosts` (Uniform Setup)**: Both RHEL 9 and RHEL 10 execute the exact same hypervisor provisioning (both install KVM, both install Cockpit, both use `/var/lib/libvirt/images`, both use `virbr0` and `kvm_br0`). They share a single group.
  - **`cis` (Version-Specific Security Rules)**: RHEL 9 uses the `rhel9_cis` role with `rhel9cis_*` variables, while RHEL 10 uses `rhel10_cis` with `rhel10cis_*` variables.
  - Creating child sub-groups (`cis_rhel9_host` and `cis_rhel10_host`) allows Ansible to automatically load `cis_rhel9_host.yml` for RHEL 9 and `cis_rhel10_host.yml` for RHEL 10, while still allowing the playbook to target `hosts: cis` to run hardening across all machines simultaneously.

---

### Q3: How does Ansible Variable Precedence work? (Why does `kvm_hosts/` win over `all.yml`?)

- **A**:
  - In Ansible, **"The MORE SPECIFIC rule ALWAYS wins over the MORE GENERAL rule."**
  - **Precedence Ranking (From Lowest to Highest Power)**:
    1. `role/defaults/main.yml` (Lowest power — fallback default)
    1. `role/defaults/main.yml` (Lowest power fallback default)
    2. `group_vars/all.yml` (Global settings for ALL machines)
    3. `group_vars/kvm_hosts/` (Settings for hypervisors — **WINS over `all.yml`**)
    4. `group_vars/cis_rhel9_hosts.yml` (Settings for specific OS sub-groups)
    4. `group_vars/cis_rhel9_host.yml` (Settings for specific OS sub-groups)
    5. `host_vars/<hostname>.yml` (Settings for 1 individual host — highest power)

---

### Q4: Why make `kvm_hosts/` a directory instead of a single file (`kvm_hosts.yml`)?

- **A**:
  1. **Zero Git Merge Conflicts**: Developer A can edit `kvm.yml` while Developer B edits `cockpit.yml` simultaneously with 0 merge conflicts.
  1. **Zero Git Merge Conflicts**: Developer A can edit `kvm.yml` while Developer B edits `cockpit.yml` simultaneously with zero merge conflicts.
  2. **1-to-1 Role Mapping**: Clean modularity where each file corresponds directly to one role.
  3. **Granular Vault Encryption**: Allows encrypting sensitive passwords in `group_vars/kvm_hosts/vault.yml` while keeping other files in readable plain text.
  4. **Scalability**: New roles (e.g. `monitoring.yml`, `backup.yml`) can be dropped into the folder without bloating a monolithic file.

---

### Q5: Can `all.yml` override variables in the CIS role?

- **A**:
  - **YES!** Variables defined in `group_vars/all.yml` have higher precedence than the CIS role's internal `defaults/main.yml`.
  - However, if the same variable is also defined in `cis_rhel9_hosts.yml`, the more specific child group (`cis_rhel9_hosts.yml`) will win over `all.yml`.
  - However, if the same variable is also defined in `cis_rhel9_host.yml`, the more specific child group (`cis_rhel9_host.yml`) will win over `all.yml`.

---

### Q6: Why is the master playbook named `site.yml`?
### Q6: Why is the master playbook named `main_playbook.yml`?

- **A**:
  - In enterprise IT and datacenters, the entire infrastructure or datacenter deployment is called a **"Site"**.
  - Red Hat and Ansible established the universal convention that the master top-level playbook orchestrating all infrastructure and security across an entire environment is named **`site.yml`** ("Deploy my entire site").
  - In this project, `playbooks/main_playbook.yml` clearly signals that it is the primary orchestrator that coordinates all roles, environments, and phases across the entire lifecycle.
  - Red Hat and standard Ansible conventions also frequently name this top-level orchestrator `site.yml` (representing "Deploy my entire datacenter or site"). Both names are accepted industry standards for multi-role orchestration.

---

### Q7: How does the Master Playbook dynamically apply RHEL 9 vs RHEL 10 CIS roles?

- **A**:
  - In `playbooks/site.yml`, Ansible evaluates `ansible_distribution_major_version`:
    - When `ansible_distribution_major_version == '9'` ➔ executes `rhel9_cis`.
    - When `ansible_distribution_major_version == '10'` ➔ executes `rhel10_cis`.
  - In `playbooks/main_playbook.yml`, Ansible evaluates `ansible_facts['distribution_major_version']`:
    - When `ansible_facts['distribution_major_version'] == '9'`: executes `rhel9_cis`.
    - When `ansible_facts['distribution_major_version'] == '10'`: executes `rhel10_cis`.

---

### Q8: What critical CIS variable overrides prevent hypervisors from breaking?

- **A**:
  1. **Kernel IP Forwarding**: `rhel9cis_sysctl_net_ipv4_ip_forward: 1` and `rhel9cis_rule_3_1_1: false` (prevents CIS from disabling packet forwarding on VM NAT bridges).
  2. **Cockpit Port 9090**: `rhel9cis_firewalld_services: ['ssh', 'cockpit']` (ensures the web console remains reachable through the firewall).
  3. **Kernel Modules**: `rhel9cis_whitelist_kernel_modules: ['kvm', 'kvm_intel', 'kvm_amd', 'vhost_net', 'tun']` (ensures hypervisor kernel drivers are not blacklisted).
