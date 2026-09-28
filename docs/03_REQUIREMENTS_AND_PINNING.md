# requirements.yml: what is pulled and why it is pinned

`requirements.yml` lists every role and collection the project needs. One command installs them all:

```bash
ansible-galaxy install -r requirements.yml -p roles/ --force
```
`-p roles/` puts them in `roles/` (the path set in `ansible.cfg`). `--force` replaces what is already there. `roles/` is in `.gitignore`: these are downloaded, not stored in this repository.

## Roles

| Role | Source | Version | Pinned? |
| :--- | :--- | :--- | :--- |
| `rhel_kvm` | `github.com/bunnywkwk/rhel_kvm` | `main` (a branch) | No, it follows the latest commit |
| `rhel_cockpit` | `github.com/bunnywkwk/rhel_cockpit` | `main` (a branch) | No |
| `rhel9_cis` | `github.com/ansible-lockdown/RHEL9-CIS` | `2.4.0` (a tag) | Yes |
| `rhel10_cis` | `github.com/ansible-lockdown/RHEL10-CIS` | `1.1.0` (a tag) | Yes |

`scm: git` tells Galaxy to clone the repository (not use the Galaxy website). This is how the custom roles are "separate repositories pulled with Ansible Galaxy", as the task requires.

## Why the CIS roles are pinned to a tag
- **Same result every time.** A tag never moves. A branch like `main` changes with every commit, so the same playbook could harden differently next week.
- **The variables can change between versions.** Our `cis_*.yml` files use rule numbers and variable names from these exact versions. A new version may rename or add rules.
- **Security work should be reproducible and reviewable:** "we ran CIS role 2.4.0" is a statement you can defend.

## Why the custom roles use `main`
They are our own code and change while we build them. Once they are finished, they should get a tag too (for example `v1.0.0`) so the whole project is pinned.

## Collections
| Collection | Used for |
| :--- | :--- |
| `ansible.posix` | `sysctl` module (IP forwarding) and the CIS roles |
| `community.general` | `modprobe` and `sefcontext` modules (`rhel_kvm`) |
| `community.libvirt` | `virt_pool` and `virt_net` modules (`rhel_kvm`) |
| `ansible.utils` | listed, see the note below |

No versions are set for the collections: the newest is installed.

---

## Known limits (nothing changed)
- **Newer CIS releases exist.** GitHub's latest releases are `2.4.1` (RHEL 9, 2026-09-21) and `1.1.1` (RHEL 10, 2026-08-28). The task asks for the latest release, and the pins are one patch release behind. I compared them: every variable we set still exists in both newer tags. Moving to them means updating two lines and doing one more full run.
- `ansible.utils` is not used by anything I found in the repo, the roles or the CIS roles.
- Custom roles and collections are not pinned (see above).
- `ansible-galaxy install --force` replaces the roles in `roles/` with what is on GitHub. Push changes to `rhel_kvm` and `rhel_cockpit` first, or the local copies are overwritten.
