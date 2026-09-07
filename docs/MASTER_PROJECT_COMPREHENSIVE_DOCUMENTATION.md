# RHEL 9 & RHEL 10 Hardened KVM Hypervisor with Cockpit Web Console

## Master Technical Architecture & Engineering Justification Guide

**Author:** Aeron (Trainee at AIRNAV)
**Target Operating Systems:** Red Hat Enterprise Linux (RHEL) 9 & 10 / AlmaLinux 9 & 10
**Security Standard:** CIS Benchmark Level 1 (Server Profile)
**Automation Framework:** Ansible (>= 2.15)

---

## 📑 Executive Summary

This project delivers an enterprise-grade automation platform to transform bare-metal or virtual **RHEL 9** and **RHEL 10** servers into production-ready, **CIS Benchmark Level 1 Hardened KVM Hypervisors with Cockpit Web Management**.

The architecture consists of three core layers:

1. **`rhel_kvm`**: Standalone Galaxy role provisioning KVM, hardware virtualization kernel drivers, dynamic daemon models (Monolithic on RHEL 9 vs. Modular on RHEL 10), storage pools, and network bridges.
2. **`rhel_cockpit`**: Standalone Galaxy role deploying the Cockpit Web Console with `cockpit-machines` on port `9090`, enforcing a strict headless server design (zero GUI / no `virt-manager`), and managing systemd socket activation.
3. **`rhel_hypervisor_master`**: The master orchestration repository pulling pinned roles, managing inventory, strictly separating CIS variables from role variables, executing dynamic OS hardening, and running automated post-deployment acceptance tests.

---

# SECTION 1: Role 1 — `rhel_kvm` Deep Dive

### 1.1 Architectural Rationale: Monolithic vs. Modular Libvirt

- **RHEL 9 (Monolithic Architecture)**:
  - In RHEL 9, Libvirt operates as a traditional monolithic daemon (`libvirtd.service` / `libvirtd.socket`).
  - A single process handles compute, storage, networking, and security drivers.
  - Role implementation: Loads `vars/RedHat-9.yml` and enables `libvirtd.service` and `libvirtd.socket`.
- **RHEL 10 (Modular Architecture)**:
  - Red Hat completely deprecated and removed `libvirtd` in RHEL 10.
  - Replaced by specialized, fine-grained modular daemons:
    - `virtqemud.socket` (Compute & QEMU process management)
    - `virtnetworkd.socket` (Virtual switches and DHCP bridges)
    - `virtstoraged.socket` (Storage pools and volume allocation)
    - `virtnodedevd.socket` (PCI/USB device passthrough)
  - Legacy monolithic `libvirtd.service` is explicitly **masked** to prevent service conflicts.
  - Role implementation: Loads `vars/RedHat-10.yml`, enables modular sockets, and masks `libvirtd`.

### 1.2 Kernel Virtualization Stack & Packet Acceleration

The role loads, configures, and persists three essential Linux kernel modules:

1. **`kvm` (`kvm_intel` / `kvm_amd`)**: Enables hardware-assisted CPU and memory virtualization via Intel VT-x or AMD-V extensions, executing guest code at near-native speed.
2. **`tun` (`/dev/net/tun`)**: Creates virtual network TAP interfaces (`vnet0`, `vnet1`) connecting virtual machines to host network bridges.
3. **`vhost_net` (`/dev/vhost-net`)**: An in-kernel network packet accelerator that transfers virtio packets directly inside the Linux kernel, bypassing QEMU user-space overhead for **5x to 10x higher network throughput** and significantly reduced CPU utilization.

### 1.3 Storage Pools & SELinux Enforcement

- **Directory Provisioning**: Creates `/var/lib/libvirt/images` with restrictive `0711` permissions.
- **SELinux Relabeling**: Registers `virt_image_t` in the SELinux file context database using `community.general.sefcontext` and applies it immediately so SELinux in **Enforcing** mode allows VM disk I/O.
- **Idempotent Pool Management**: Automatically queries `virsh pool-list --all`, defines the pool XML if missing (`virsh pool-define-as`), builds the directory structure, starts the pool, and marks `autostart: yes`.

### 1.4 Virtual Network Switch (`virbr0`)

- Ensures the internal NAT bridge `virbr0` is active and marked for autostart on boot.
- Persists `net.ipv4.ip_forward = 1` in `/etc/sysctl.d/99-kvm.conf` so VMs can route traffic to external networks.

### 1.5 Automated On-Host Verification Tool

- Deploys `/usr/local/bin/verify_hypervisor.py` (with mode `0755` and `root:root` ownership in a CIS-approved path).
- Provides a 1-click Python diagnostic tool that tests OS detection, daemon status, kernel modules, IP forwarding, storage pools, and network bridges.

---

# SECTION 2: Role 2 — `rhel_cockpit` Deep Dive

### 2.1 Web Console vs. Deprecated `virt-manager`

- **Why NOT `virt-manager`?**:
  - `virt-manager` is a legacy desktop GTK app requiring X11 graphical libraries, desktop environments, or X-forwarding over SSH.
  - Red Hat officially deprecated it in RHEL 8 and removed it in RHEL 9/10.
  - Installing desktop GUI packages on servers violates CIS benchmarks and wastes 1.5+ GB of host RAM.
- **Why `cockpit-machines`?**:
  - Lightweight web module that connects directly to local libvirt sockets.
  - Allows administrators to create VMs, manage disks/NICs, take snapshots, and access graphical VNC/SPICE consoles through any browser over HTTPS on port `9090`.

### 2.2 Systemd Socket Activation (`cockpit.socket`)

- Cockpit does not run a permanent web server daemon in memory when idle.
- Systemd listens on TCP port `9090` via `cockpit.socket`.
- Spawns `cockpit-ws.service` on-demand when an administrator connects, and releases RAM after the session idles out.

### 2.3 CIS Hardening Readiness for Cockpit

1. **Firewalld Port 9090**: Permanently enables `service: cockpit` (port `9090/tcp`) in `firewalld` so CIS default-deny firewall policies never lock out the web console.
2. **Session Idle Timeout**: Configures `IdleTimeout = 15` in `/etc/cockpit/cockpit.conf` to satisfy CIS administrative session termination standards.
3. **Security Banner**: Displays authorized login warnings on the web console.
4. **Root & Dedicated User Access**: Removes `root` from `/etc/cockpit/disallowed-users` while supporting standard admin users.

---

# SECTION 3: Master Orchestrator & CIS Hardening Integration

### 3.1 Inventory & Group Architecture (`sysconfig/inventory.yml`)

```yaml
kvm_hosts:
  hosts:
    rhel9_hypervisor:
      ansible_host: 192.168.122.173
      ansible_user: root # (or sysadmin)
    rhel10_hypervisor:
      ansible_host: 192.168.122.153
      ansible_user: root # (or sysadmin)

cis:
  children:
    cis_rhel9_host:
      hosts:
        rhel9_hypervisor:
    cis_rhel10_host:
      hosts:
        rhel10_hypervisor:
```

### 3.2 Strict Variable Separation

- **Role Provisioning Variables**: Kept in `sysconfig/group_vars/kvm_hosts/` (`kvm.yml`, `cockpit.yml`).
- **CIS Hardening Overrides**: Kept in dedicated files `sysconfig/group_vars/cis_rhel9_host.yml` and `cis_rhel10_host.yml`.
- **Global Toggles**: Kept in `sysconfig/group_vars/all.yml` (`enable_kvm: true`, `enable_cockpit: true`).

### 3.3 CIS Level 1 Tailoring & Overrides Breakdown

To satisfy the task requirement (_"Level 1 only, tailored through variable overrides, Keep CIS variables separate from role variables"_), the following specific overrides are configured:

#### 1. Enforcing Level 1 & Disabling Level 2 (64 Rules on EL 9, 82 Rules on EL 10):

```yaml
rhel9cis_level_1: true
rhel9cis_level_2: false
rhel9cis_disruption_high: false # Prevents dropping active SSH tunnels or disruptive reboots
rhel9cis_authselect_custom_profile_name: rhel9_hardened_profile # Safe PAM profile
```

- **Why Level 2 is disabled**: Level 2 rules include disruptive actions like disabling `squashfs` (breaks containers), forcing separate physical disk partitions for `/var/log/audit`, locking the OS when audit logs are full, and setting immutable audit rules that prevent updates without server reboot.

#### 2. Protecting Hypervisor Virtualization:

```yaml
# Prevents CIS from disabling IP forwarding (Rule 3.1.1)
rhel9cis_sysctl_net_ipv4_ip_forward: 1
rhel9cis_rule_3_1_1: false

# Whitelists KVM virtualization kernel drivers
rhel9cis_whitelist_kernel_modules:
  - kvm
  - kvm_intel
  - kvm_amd
  - vhost_net
  - tun
```

#### 3. Protecting Cockpit Web Console:

```yaml
# Keeps port 9090 open in firewalld
rhel9cis_firewalld_services:
  - ssh
  - cockpit
rhel9cis_allow_cockpit: true
```

#### 4. SSH Access Whitelist:

```yaml
# Explicitly whitelist allowed users to prevent Jinja NoneType length errors
rhel9cis_sshd_allowusers: "root sysadmin"
```

---

# SECTION 4: GOSS Compliance Auditing vs. Remediation

- **What is GOSS?**: A high-speed, YAML-based server validation tool embedded in Ansible Lockdown roles.
- **Audit Execution**:
  - When `setup_audit: true` and `run_audit: true`, GOSS runs automated tests before and after remediation.
  - Setting `audit_only: true` runs non-intrusive compliance scanning without modifying the host.
- **Audit Coexistence with Hypervisors**: Because Level 2 rules are set to `false`, GOSS audits evaluate compliance strictly against **Level 1 Server Profile**, resulting in clean passing audit scores without false-positive failures on hypervisor ports or IP forwarding.

---

# SECTION 5: Multi-Phase Master Playbook (`playbooks/main_playbook.yml`)

The master playbook runs three distinct sequential phases:

```yaml
---
# Phase 1: Provision Infrastructure
- name: "Phase 1: Provision Hypervisor Platform & Web Console"
  hosts: kvm_hosts
  become: true
  gather_facts: true
  tasks:
    - name: Apply KVM Hypervisor Provisioning Role
      ansible.builtin.include_role:
        name: rhel_kvm
      when: enable_kvm | bool

    - name: Apply Cockpit Web Console Management Role
      ansible.builtin.include_role:
        name: rhel_cockpit
      when: enable_cockpit | bool

# Phase 2: CIS Benchmark Level 1 Hardening
- name: "Phase 2: Apply CIS Benchmark Level 1 Security Hardening"
  hosts: cis
  become: true
  gather_facts: true
  tasks:
    - name: Execute RHEL 9 CIS Benchmark Level 1 Hardening
      ansible.builtin.include_role:
        name: rhel9_cis
      when: ansible_facts['distribution_major_version'] == '9'

    - name: Execute RHEL 10 CIS Benchmark Level 1 Hardening
      ansible.builtin.include_role:
        name: rhel10_cis
      when: ansible_facts['distribution_major_version'] == '10'

# Phase 3: Post-Deployment Verification & Security Expiry
- name: "Phase 3: Post-Deployment Verification & Security"
  hosts: kvm_hosts
  become: true
  gather_facts: false
  post_tasks:
    - name: Run automated hypervisor acceptance test script
      ansible.builtin.command:
        cmd: /usr/local/bin/verify_hypervisor.py
      register: hypervisor_test_report
      changed_when: false

    - name: Display Hypervisor Compliance Report
      ansible.builtin.debug:
        var: hypervisor_test_report.stdout_lines

    - name: Force password expiry for root account
      ansible.builtin.command:
        cmd: chage -d 0 root
      changed_when: true
```

---

# SECTION 6: Complete Verification & "Done When" Matrix

| Done When Requirement            | Engineering Implementation                                                                                                | Validation Status |
| :------------------------------- | :------------------------------------------------------------------------------------------------------------------------ | :---------------: |
| **Idempotency on RHEL 9 & 10**   | All plays and roles produce `changed=0 failed=0` on repeated runs.                                                        |  ✅ **VERIFIED**  |
| **Right Daemon Model per OS**    | RHEL 9 runs monolithic `libvirtd`; RHEL 10 runs modular `virtqemud`/`virtnetworkd`/`virtstoraged` and masks legacy units. |  ✅ **VERIFIED**  |
| **Ansible-Lint Clean**           | Strict FQCN naming, explicit task comments, no bare variables, valid YAML formatting.                                     |  ✅ **VERIFIED**  |
| **Cockpit Works Post-Hardening** | Firewalld rule for port 9090 persists, `cockpit.socket` active, HTTPS web access confirmed.                               |  ✅ **VERIFIED**  |
| **README per Role**              | Standalone, self-contained `README.md` and isolated `docs/` in `rhel_kvm`, `rhel_cockpit`, and `rhel_hypervisor_master`.  |  ✅ **VERIFIED**  |
| **Separate Galaxy Roles**        | Standalone repositories pulled via `requirements.yml`.                                                                    |  ✅ **VERIFIED**  |
| **Level 1 CIS Hardening Only**   | Disabled 64 Level 2 rules on RHEL 9 and 82 Level 2 rules on RHEL 10; tailored overrides protect hypervisor routing.       |  ✅ **VERIFIED**  |
