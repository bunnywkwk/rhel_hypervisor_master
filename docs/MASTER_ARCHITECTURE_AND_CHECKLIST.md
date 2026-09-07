# RHEL Hypervisor Master Orchestration - Architecture & Checklist

## 🎯 Project Mission

Provide an enterprise-grade Ansible Master Orchestrator Playbook to transform bare **RHEL 9** and **RHEL 10** servers into **CIS Benchmark Level 1 Hardened KVM Hypervisors with Cockpit Web Management**.

This master playbook pulls standalone roles (`rhel_kvm`, `rhel_cockpit`, and `ansible-lockdown/RHEL9-CIS` / `RHEL10-CIS`) via `requirements.yml`, executes a multi-stage deployment pipeline, enforces strict separation of CIS variables from role variables, and verifies hypervisor functionality.

---

## 🏗️ Master Project Directory Tree

```
rhel_hypervisor_master /
├── ansible.cfg                          # Points to sysconfig/inventory.yml & roles/
├── requirements.yml                     # Pinned Galaxy roles (rhel_kvm, rhel_cockpit, CIS roles)
│
├── playbooks/
│   └── site.yml                         # Master 3-Phase Playbook (KVM -> Cockpit -> CIS -> Verify)
│
├── sysconfig/
│   ├── inventory.yml                    # RHEL 9 & 10 host mapping & CIS sub-groups
│   │
│   └── group_vars/                      # STRICT VARIABLE SEPARATION
│       ├── all.yml                      # Global execution feature toggles
│       │
│       ├── kvm_hosts/                   # Role variables for Provisioning
│       │   ├── kvm.yml                  # Storage pools, bridges, admin users
│       │   └── cockpit.yml              # Port 9090, timeout, security banners
│       │
│       ├── cis_rhel9_hosts.yml          # CIS Level 1 Overrides for RHEL 9
│       └── cis_rhel10_hosts.yml         # CIS Level 1 Overrides for RHEL 10
│
├── docs/                                # Project Knowledge Base & Reference Guides
│   ├── MASTER_ARCHITECTURE_AND_CHECKLIST.md
│   └── KNOWLEDGE_BASE_QA.md
│
└── roles/                               # Pinned roles pulled via ansible-galaxy
    ├── rhel_kvm
    ├── rhel_cockpit
    ├── rhel9_cis
    └── rhel10_cis
```

---

## 🔄 Multi-Phase Deployment Pipeline (`playbooks/site.yml`)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Phase 1: Infrastructure Provisioning (Target: hosts: kvm_hosts)             │
│ • rhel_kvm: Deploys Monolithic (EL 9) or Modular (EL 10) daemons, storage    │
│   pools (/var/lib/libvirt/images), SELinux virt_image_t, and virbr0 bridge   │
│ • rhel_cockpit: Deploys Cockpit + cockpit-machines on port 9090 (No GUI)     │
└──────────────────────────────────────┬───────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ Phase 2: CIS Benchmark Level 1 Hardening (Target: hosts: cis)                │
│ • rhel9_cis (on RHEL 9): 200+ Level 1 controls with tailored overrides       │
│ • rhel10_cis (on RHEL 10): Modular socket and modern Level 1 controls        │
│ • Tailored Overrides: Preserves net.ipv4.ip_forward=1, port 9090, & kvm mods │
└──────────────────────────────────────┬───────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ Phase 3: Post-Deployment Compliance & Functional Validation                  │
│ • Runs /usr/local/bin/verify_hypervisor.py on every hypervisor host          │
│ • Asserts 100% PASS for Daemons, Storage Pools, Bridges, Sysctl, and Cockpit │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 📋 Done When Compliance Checklist

- [x] **Role Modularity**: `rhel_kvm` and `rhel_cockpit` developed as standalone, reusable Galaxy-ready roles with isolated documentation.
- [x] **Cross-Version Daemon Compatibility**:
  - RHEL 9: Monolithic `libvirtd.service` / `libvirtd.socket` verified.
  - RHEL 10: Modular daemons (`virtqemud`, `virtnetworkd`, `virtstoraged`) verified and monolithic `libvirtd` masked.
- [x] **Headless Architecture**: Zero X11/desktop GUI packages; `virt-manager` is explicitly absent; Cockpit web console managed via `cockpit.socket`.
- [x] **Idempotency**: All plays and roles achieve `changed=0` on repeated executions.
- [x] **Ansible-Lint Compliance**: Strict FQCN naming, proper indentation, and clean YAML syntax.
- [x] **Variable Separation**: CIS overrides isolated in `cis_rhel9_hosts.yml` and `cis_rhel10_hosts.yml`, completely separate from role provisioning variables.
- [x] **Post-Hardening Coexistence**: Tailored CIS rules ensure Cockpit port 9090 and KVM virtual networking remain 100% operational after Level 1 hardening.
