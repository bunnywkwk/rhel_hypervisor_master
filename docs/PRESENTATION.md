# Hardened KVM Hypervisors on RHEL 9 and RHEL 10

Presentation guide. Each slide has **On the slide** (what the audience sees) and **Say** (what you talk about). It is meant to be read top to bottom.

**Order:** the two roles first (Cockpit, then KVM), then the playbook that ties them together.
**Time:** about 25 minutes plus questions.

| Part | Slides | Time |
| :--- | :--- | :--- |
| Introduction | 1 to 4 | 4 min |
| Part 1: `rhel_cockpit` | 5 to 8 | 4 min |
| Part 2: `rhel_kvm` | 9 to 15 | 9 min |
| Part 3: the playbook | 16 to 21 | 8 min |
| Wrap-up and questions | 22 to 24 | rest |

---

## Slide 1: Title

**On the slide**
- Hardened KVM Hypervisors on RHEL 9 and RHEL 10
- Two Ansible roles, one orchestrator, CIS Level 1
- Aeron, Trainee, AIRNAV

**Say**
- "I will show how a bare RHEL machine becomes a hardened virtualization host with a web console, using Ansible."
- "Two roles that I wrote, two CIS roles from Ansible Lockdown, and one playbook that runs them in order."

---

## Slide 2: The task

**On the slide**
- Start: a bare RHEL 9 or RHEL 10 machine
- End: a hardened KVM host with a web console
- `rhel_kvm`: right libvirt setup for each OS, storage pool, virtual network
- `rhel_cockpit`: Cockpit with VM management, no virt-manager
- Each role in its own repository, pulled with Ansible Galaxy
- CIS role for the OS, pinned to a tag, Level 1 only, CIS settings kept separate

**Say**
- "The task has four parts: two roles, hardening, and a way to run it all."
- "It is finished when: it can be run twice with no unplanned changes, the right daemon runs on each OS, ansible-lint is clean, Cockpit still works after hardening, and each role has a README."

---

## Slide 3: The big picture

**On the slide**
```text
                 Control node  (ansible-playbook)
                          |
             SSH through the jump host 192.168.10.160
                 |                        |
        rhel9_hypervisor           rhel10_hypervisor
         192.168.20.30               192.168.20.40


   What each hypervisor ends up as (bottom to top):

   +------------------------------------------------------+
   |  Cockpit web console  https://host:9090   rhel_cockpit |
   +------------------------------------------------------+
   |  KVM + libvirt, storage pool, VM network    rhel_kvm    |
   +------------------------------------------------------+
   |  CIS Level 1 hardening         rhel9_cis / rhel10_cis  |
   +------------------------------------------------------+
   |  RHEL 9 or RHEL 10                                     |
   +------------------------------------------------------+
```

**Say**
- "One playbook, run from one machine, builds both hypervisors. The hosts are on a private network, so SSH goes through a jump host."
- "The same playbook handles RHEL 9 and RHEL 10. The differences between the two are inside the roles, not in the playbook."

---

## Slide 4: Where the code lives

**On the slide**
```text
   GitHub                                             Control node
   bunnywkwk/rhel_kvm            (branch main)  --+
   bunnywkwk/rhel_cockpit        (branch main)  --+   ansible-galaxy install
   ansible-lockdown/RHEL9-CIS    (tag 2.4.0)    --+   -r requirements.yml   ---->   roles/
   ansible-lockdown/RHEL10-CIS   (tag 1.1.0)    --+
```

| Repository | What it holds |
| :--- | :--- |
| `rhel_kvm` | my KVM role |
| `rhel_cockpit` | my Cockpit role |
| `rhel_hypervisor_master` | the orchestrator: playbook, inventory, variables, docs |

**Say**
- "Each role is its own repository, so it can be reused and versioned separately. The orchestrator pulls them with one command."
- "The two CIS roles come from Ansible Lockdown, pinned to a release tag so the hardening is the same every time."

---

## Part 1: rhel_cockpit

## Slide 5: What Cockpit is and why we use it

**On the slide**
```text
   Browser  ---->  https://host:9090  ---->  Cockpit  ---->  libvirt
                                                              (set up by rhel_kvm)
```
- A web console to manage the server and its virtual machines
- `cockpit-machines` is the VM page: create, start, stop, open a VM console
- The host stays headless: no desktop, no `virt-manager`

**Say**
- "Cockpit gives administrators a browser page for the VMs, so nothing graphical has to be installed on the server."
- "Cockpit is only the front. The VMs themselves are managed by libvirt, which the other role sets up."

---

## Slide 6: What the role does

**On the slide**
```text
   1. Check the OS       2. Install Cockpit      3. Write the config     4. Enable the socket
      RHEL 9 or 10          cockpit                 IdleTimeout = 15        cockpit.socket
                            cockpit-machines        (minutes)               listens on 9090
                            + 3 helper packages
```

**Say**
- "Four small steps."
- "Step 3 is the one hardening setting: an idle web session is logged out after 15 minutes. That is a CIS requirement."
- "Step 4 uses a socket: Cockpit starts only when someone opens the page, so it uses no memory when idle."

---

## Slide 7: Design choices

**On the slide**
- The required packages are protected in the role's `vars/`, extras go in `defaults/`
- The config file contains one setting only
- Things deliberately left alone: port, login banner, root login, firewall
- Every setting was checked against Cockpit's own manual page

**Say**
- "If the package list were an ordinary default, one line in the inventory could replace it and drop `cockpit-machines`. So the required list is protected, and users can only add extras."
- "I removed settings that looked useful but did nothing. For example the port cannot be set in the config file, and the banner needs a file path."
- "The firewall already allows Cockpit on a normal RHEL 9 and RHEL 10. I checked a fresh host, so the role does not touch the firewall."

---

## Slide 8: Proof it works

**On the slide**
- Open `https://192.168.20.30:9090` from another VM: the Cockpit login page appears
- Log in as `frqadmin`: the Virtual Machines page works once `rhel_kvm` has run
- `systemctl is-active cockpit.socket` gives `active`
- Reachable after CIS hardening (checked from another VM)

**Say**
- "This is the check I use after every full run: open the page from a different machine."
- "If the Virtual Machines page says libvirt is not running, it means `rhel_kvm` has not run yet or the host needs a reboot. That is why the KVM role runs first."

---

## Part 2: rhel_kvm

## Slide 9: What rhel_kvm builds

**On the slide**
- KVM and libvirt, with the right daemon design for each OS
- A storage pool for VM disks, with the correct SELinux label
- An isolated virtual network `kvm_br0` for VM-to-VM traffic (DHCP, no NAT)
- Kernel modules and IP forwarding that virtualization needs

**Say**
- "This is the engine of the project. Cockpit only shows what this role builds."
- "It is the harder of the two roles, because RHEL 9 and RHEL 10 handle libvirt differently."

---

## Slide 10: The main challenge: two libvirt designs

**On the slide**
```text
        RHEL 9  (required)                          RHEL 10

        libvirtd                                    virtqemud      runs the VMs
        one program does everything                 virtnetworkd   runs the networks
                                                    virtstoraged   runs the storage

        the role enables and starts it              libvirtd is masked
```

**Say**
- "RHEL 9 must run the single `libvirtd` daemon. That is a fixed requirement of this project."
- "RHEL 10 removed `libvirtd` and only has separate daemons for each job. The role enables the sockets of all drivers and starts `virtqemud`; the other daemons start on demand."
- "On RHEL 9 the role keeps it simple: it enables and starts `libvirtd` and does not touch the other daemons."

---

## Slide 11: How one role handles both

**On the slide**
```text
   tasks/main.yml  --->  loads  vars/RedHat-9.yml   or   vars/RedHat-10.yml
                                     |                         |
                                     +---- same variable names, different values ----+
                          sockets and services to start | daemons to mask
```
- No `if RHEL 9 ... else ...` in the tasks
- The tasks loop over whatever the vars file lists

**Say**
- "The role reads the OS version from the host and loads the matching settings file."
- "To support another version, I add a file. I do not change any task."

---

## Slide 12: The seven steps

**On the slide**
```text
   1 Load OS      2 Check       3 Install     4 IP           5 Start        6 Storage     7 Network
     settings       OS check,     packages      forwarding     libvirt        pool          kvm_br0
                    load kernel                                daemons
                    modules
```

**Say**
- "The order matters: packages first, then the daemons, then the pool and the network, because those need a running libvirt."
- "Every step is a separate small file, so I can explain them one at a time. Each is documented in the role's `TASK_WALKTHROUGH.md`."

---

## Slide 13: Storage and networking

**On the slide**
```text
   VM ----+                                   default (NAT)  -->  host  -->  LAN / internet
          |                                   virbr0, 192.168.122.0/24, needs IP forwarding
   VM ----+---- kvm_br0 (virbr1) ---- host
                192.168.100.1/24      isolated: VMs and host only
                DHCP .10 to .254
```
- Storage pool `default` at `/var/lib/libvirt/images`, restricted permissions (`0711`), SELinux label `virt_image_t`
- Both the pool and the network start on boot

**Say**
- "kvm_br0 is an isolated network: VMs on it talk to each other and to the host, and get addresses from the DHCP range. Internet access comes from the built-in default network, which is NAT."
- "NAT needs IP forwarding. CIS turns it off, and my role sets it back on in a file that is read later, so hardening does not break the default network."
- "The SELinux label is what allows the VM process to write its disks when SELinux is enforcing."

---

## Slide 14: Problems I hit and fixed

**On the slide**

| What happened | Fix |
| :--- | :--- |
| RHEL 10: package install failed with a GPG signature error | Update `redhat-release` first, it brings the new signing key |
| Network did not come back after a reboot | Set autostart in its own task |
| CIS set IP forwarding to 0 | Our setting loads later and wins |

**Say**
- "Each of these was a real error on the test hosts. All of them are written up with the exact error message in the lessons-learned document."
- "The lesson: test after a reboot and after hardening, not only after the first run."

---

## Slide 15: Proof it works

**On the slide**
```text
   RHEL 9    systemctl is-active libvirtd             active
   RHEL 10   systemctl is-active virtqemud.socket     active
   Both      virsh net-list --all    kvm_br0   active, autostart yes
             virsh pool-list --all   default   active, autostart yes
             sysctl net.ipv4.ip_forward         1
```

**Say**
- "This is the evidence that the right daemon is running on each OS, checked after hardening. On RHEL 9 I also checked it after a reboot."

---

## Part 3: the playbook

## Slide 16: Three phases

**On the slide**
```text
   Phase 1 : provision        Phase 2 : harden               Post-deployment
   kvm_hosts                  cis                            kvm_hosts
   rhel_kvm                   rhel9_cis   on RHEL 9          expire the root password
   rhel_cockpit               rhel10_cis  on RHEL 10         (root must set a new one)
```

**Say**
- "Build first, then harden. The hardening rules are tuned so they do not damage what was just built."
- "The playbook contains no package names or settings. It only orders the roles."

---

## Slide 17: Inventory and group variables

**On the slide**
```text
   Inventory groups                    Variable files (sysconfig/group_vars/)

   kvm_hosts        ----------------->  all.yml, kvm_hosts/   the on/off switches and role settings
   cis_rhel9_host   ----------------->  cis_rhel9_host.yml    CIS settings, RHEL 9 only
   cis_rhel10_host  ----------------->  cis_rhel10_host.yml   CIS settings, RHEL 10 only
```
- CIS variables and role variables are in separate files
- A new host needs no code: put it in a group

**Say**
- "Ansible loads the file that matches the group name. So the settings follow the host's group."
- "Keeping the CIS settings apart from the role settings is a requirement of the task, and it also lets a security reviewer read one file."
- "The same playbook serves both OSes because each OS group has its own settings file."

---

## Slide 18: How Level 1 is applied

**On the slide**

| | RHEL 9 | RHEL 10 |
| :--- | :--- | :--- |
| Level 2 rules in the CIS benchmark | 68 | 86 |
| Level 2 rules switched off | 68 | 86 |
| Level 2 rules left on | 0 | 0 |

- Kept on purpose: `dnsmasq` (libvirt needs it to give VMs addresses)
- Off on purpose: failed-login lockout (`pam_faillock`)
- Cockpit is not removed on either OS

**Say**
- "The CIS role runs everything by default. Its own level setting is only a label, so Level 1 is done by switching off every Level 2 rule."
- "The list of Level 2 rules comes from the official CIS benchmark spreadsheet. I checked my lists against it in both directions."
- "Everything that is Level 2 for the Server or the Workstation profile is off. That includes a few rules that are Level 1 for servers but Level 2 for workstations, for example USB storage and bluetooth."

---

## Slide 19: Pinning and requirements

**On the slide**

| What | Version | Why |
| :--- | :--- | :--- |
| CIS roles | a release tag (2.4.0, 1.1.0) | a tag never moves, so hardening is repeatable |
| My two roles | branch `main` | still under development, to be tagged when finished |
| Collections | newest | the modules the roles use |

**Say**
- "A branch changes with every commit. A tag does not. For security work, being able to say exactly which version was applied matters."
- "The CIS rule numbers and variable names can change between versions, so the variables in my files are tied to these versions."

---

## Slide 20: Running it

**On the slide**
```text
   Everything          ansible-galaxy install -r requirements.yml -p roles/
                       ansible-playbook playbooks/main_playbook.yml -K

   One role, one host  ansible-playbook playbooks/main_playbook.yml \
                         --tags rhel_kvm --limit rhel9_hypervisor -K
```
- Tags: `rhel_kvm`, `rhel_cockpit`
- With a tag, the hardening is skipped
- Run it again: the two roles should report no changes

**Say**
- "Install once, then run. You can re-apply one role on one host without touching the hardening."
- "Running it again is safe. That is how I check that the roles are idempotent."

---

## Slide 21: Documentation

**On the slide**
- Each role: `README.md` and `docs/TASK_WALKTHROUGH.md` (every task, what it does and why)
- `rhel_kvm` also has `docs/FAQ.md`: short answers to common questions about the code
- Orchestrator: `docs/01` to `04` (inventory, overrides, pinning, playbook)
- `LESSONS_LEARNED_AND_FIXES.md`: every problem with its real error message

**Say**
- "Every task in both roles is explained: what it does, why it exists, and the error that led to it, if there was one."

---

## Slide 22: Status against "done when"

Update this slide with your final run before you present.

**On the slide**

| Requirement | Status |
| :--- | :--- |
| Right daemon model on each OS, verified | `libvirtd` on RHEL 9, modular daemons on RHEL 10; confirm in the final run |
| Cockpit works after hardening | Done earlier; to be confirmed in the final run |
| Idempotent | The two roles are; to be confirmed in the final run |
| README per role | Done |
| CIS pinned to a tag, Level 1 only, variables separate | Done |
| ansible-lint clean | Open |

**Say**
- Be direct about what is finished and what is still open. Open items are listed on the next slide.

---

## Slide 23: Be ready for these questions

| Question | Short answer |
| :--- | :--- |
| Why is RHEL 9 monolithic? | It is a fixed requirement. The role enables and starts `libvirtd` and leaves the other daemons alone. |
| Why not modular on both? | Same: the requirement for RHEL 9. RHEL 10 has no `libvirtd`. |
| How do you get Level 1 only? | I switch off every Level 2 rule, from the official CIS spreadsheet. 68 on RHEL 9, 86 on RHEL 10. |
| Why did CIS not break IP forwarding? | CIS writes 0 in a file read early. The role writes 1 in a file read later, and the last one wins. |
| Why is Cockpit still there after CIS? | The RHEL 9 CIS role has no Cockpit rule. On RHEL 10 the rule is Level 2, so it is off. The firewall allows it by default. |
| Why pin the CIS roles? | A tag does not change. The same playbook gives the same hardening. |
| Why use `include_role` with tags in the playbook? | So `--tags` really reaches the tasks inside a role, and hardening is skipped when you run one role. |
| Why are USB storage and bluetooth rules off? | They are Level 2 for the Workstation profile, and every Level 2 rule of either profile is switched off. |

---

## Slide 24: Open items and next steps

**On the slide**
- `ansible-lint`: 6 small findings in my roles to clean up
- CIS roles: newer releases exist (2.4.1 and 1.1.1). Move the pins and run once more
- The root password expiry step always reports "changed" (it re-expires each run)
- Tag `rhel_kvm` and `rhel_cockpit` when finished, so the whole project is pinned

**Say**
- "These are known and small. I would rather show them than have them found."
- Close with: "One playbook, two OS versions, two roles, CIS Level 1, and a web console that still works after hardening."

**Thank you. Questions?**
