# Test: import a qcow2 image and start a VM

Use this to test a hypervisor after a rebuild: copy a ready-made image to it, import it in Cockpit, and boot a VM. Run everything from the control node.

## Before you start

1. **The clock is right on both hosts** (`date`). After a rollback see `docs/06_VM_ROLLBACK_AND_CLOCK.md`.
2. **The playbook has run**, so libvirt and the `default` storage pool exist. Check: `sudo virsh -c qemu:///system pool-list --all` shows `default` active.
3. **You use `sudo virsh`.** The role does not add `frqadmin` to the `libvirt` group and does not set a default connection, so a plain `virsh` as `frqadmin` reads an empty session connection. Every command below uses `sudo virsh -c qemu:///system`.

## The image

| | |
| :--- | :--- |
| File | `~/packer/output-rhel9/rhel9-hypervisor-golden.qcow2` |
| Size | 5.06 GB on disk, 64 GiB virtual |
| Firmware | UEFI (q35 machine with OVMF), virtio disk and network |
| Needs on the host | `edk2-ovmf` installed, about 10 GB free in `/var/lib/libvirt/images` |

## Results of the first test

- **192.168.20.40 (RHEL 10):** copied, moved into the `default` pool, imported in Cockpit, VM started.
- **192.168.20.30 (RHEL 9):** the pre-check passed (46 GB free, `edk2-ovmf-20241117-8.el9` installed).
- `rsync` is not installed on the hosts, so the copy uses `scp` (rsync fails with `rsync: command not found`).

---

## RHEL 9: 192.168.20.30

```bash
# 0. Check space and UEFI firmware
ssh -J frqadmin@192.168.10.160 frqadmin@192.168.20.30 'df -h /var/lib/libvirt/images; rpm -q edk2-ovmf'

# 1. Copy the image (about 5 GB)
scp -J frqadmin@192.168.10.160 \
  ~/packer/output-rhel9/rhel9-hypervisor-golden.qcow2 frqadmin@192.168.20.30:~/

# 2. Verify: the two hashes must match
sha256sum ~/packer/output-rhel9/rhel9-hypervisor-golden.qcow2
ssh -J frqadmin@192.168.10.160 frqadmin@192.168.20.30 'sha256sum ~/rhel9-hypervisor-golden.qcow2'

# 3. Move into the default pool, fix the SELinux label, refresh the pool
ssh -t -J frqadmin@192.168.10.160 frqadmin@192.168.20.30 \
  "sudo mv ~/rhel9-hypervisor-golden.qcow2 /var/lib/libvirt/images/test1.qcow2 \
   && sudo restorecon -v /var/lib/libvirt/images/test1.qcow2 \
   && sudo virsh -c qemu:///system pool-refresh default"
```
Enter the sudo password when the last command asks for it. It renames the file to `test1.qcow2` on the host.

## RHEL 10: 192.168.20.40

```bash
# 0. Check space and UEFI firmware
ssh -J frqadmin@192.168.10.160 frqadmin@192.168.20.40 'df -h /var/lib/libvirt/images; rpm -q edk2-ovmf'
```
If it says `edk2-ovmf` is not installed, install it. Otherwise skip this:
```bash
ssh -t -J frqadmin@192.168.10.160 frqadmin@192.168.20.40 'sudo dnf install -y edk2-ovmf'
```
Then:
```bash
# 1. Copy the image (about 5 GB)
scp -J frqadmin@192.168.10.160 \
  ~/packer/output-rhel9/rhel9-hypervisor-golden.qcow2 frqadmin@192.168.20.40:~/

# 2. Verify: the two hashes must match
sha256sum ~/packer/output-rhel9/rhel9-hypervisor-golden.qcow2
ssh -J frqadmin@192.168.10.160 frqadmin@192.168.20.40 'sha256sum ~/rhel9-hypervisor-golden.qcow2'

# 3. Move into the default pool, fix the SELinux label, refresh the pool
ssh -t -J frqadmin@192.168.10.160 frqadmin@192.168.20.40 \
  "sudo mv ~/rhel9-hypervisor-golden.qcow2 /var/lib/libvirt/images/test1.qcow2 \
   && sudo restorecon -v /var/lib/libvirt/images/test1.qcow2 \
   && sudo virsh -c qemu:///system pool-refresh default"
```

The last command asks for the sudo password too.

---

## Import in Cockpit (same on both hosts)

1. Open `https://192.168.20.30:9090` (or `https://192.168.20.40:9090`) and log in as `frqadmin`. Tick **Reuse my password for privileged tasks** on the login page. This gives Cockpit administrative access. Without it, `frqadmin` sees only an empty session connection and no pools or networks. If you forgot, click **Limited access** at the top right and enter the password.
2. **Virtual machines**, then **Import VM**.
3. Set:
   - **Name:** `test1`
   - **Connection:** System
   - **Disk image path:** `/var/lib/libvirt/images/test1.qcow2`
   - **Operating system:** Red Hat Enterprise Linux 9
   - **Memory:** 4 GiB, **vCPUs:** 2
   - **Firmware:** UEFI, if the dialog offers the choice
4. Click **Import**, then open the **Console** tab.

A NIC on `kvm_br0` gets an address from `192.168.100.10` to `.254`. That network is isolated (no internet). For internet access, the VM needs a NIC on the `default` network (NAT).

### RHEL 9: use `virt-install`, not the Cockpit import
The image is UEFI, and UEFI needs the **q35** machine type. On RHEL 9, Cockpit's Import VM creates the VM as `pc-i440fx-rhel7.6.0` (BIOS) and cannot change it. Changing the firmware to UEFI then fails with `Unable to find 'efi' firmware that is compatible with the current configuration`. On RHEL 10 there is no i440fx, so the Cockpit import already gives q35.

On RHEL 9, create the VM with `virt-install` instead (on the host):
```bash
sudo virt-install --connect qemu:///system \
  --name test1 --memory 4096 --vcpus 2 --import \
  --disk path=/var/lib/libvirt/images/test1.qcow2,bus=virtio \
  --network network=kvm_br0,model=virtio \
  --machine q35 --boot uefi \
  --os-variant rhel9-unknown --graphics vnc --noautoconsole
```
If `rhel9-unknown` is rejected, find a valid name with `osinfo-query os | grep rhel9`. The VM then shows in Cockpit with `pc-q35-rhel9...` and UEFI.

If you already imported a BIOS VM in Cockpit, remove its definition and keep the disk (no `--remove-all-storage`), then run the `virt-install` command:
```bash
sudo virsh -c qemu:///system destroy test1
sudo virsh -c qemu:///system undefine test1
```

---

## If something goes wrong

| Symptom | Cause and fix |
| :--- | :--- |
| `rsync: command not found` | rsync is not installed on the host. Use `scp` as above. |
| "No bootable device" | UEFI was not selected, or `edk2-ovmf` is missing. On RHEL 9 the VM may also be `pc-i440fx` (BIOS): recreate it with q35 (see above). |
| "Unable to find 'efi' firmware that is compatible with the current configuration" (RHEL 9) | The VM uses the `pc-i440fx` machine type. Recreate it with `virt-install --machine q35 --boot uefi` (see above). |
| "Permission denied" on the disk | The SELinux label step (`restorecon`) was skipped. |
| Virtual machines page shows no pools or networks | Cockpit is in limited access. Turn on administrative access (see the Cockpit steps). |
| Disk not listed in Cockpit, but the file is in the folder | Run `sudo virsh -c qemu:///system pool-refresh default`, then check `sudo virsh -c qemu:///system vol-list default`. If `test1.qcow2` is listed, Cockpit only needs a refresh: hard-reload the page (Ctrl+Shift+R), open **Storage pools**, **default**, **Storage volumes**, and check the connection is **System**. |
| Still not shown | Restart the connection between libvirt and Cockpit: `sudo systemctl restart libvirt-dbus`, then reload the page. You can also just type the path in the Import VM dialog: it has no file browser and does not need the volume list. |
| VM fails to start with "Permission denied" (file owner is `frqadmin`, mode `640`) | `sudo chown qemu:qemu /var/lib/libvirt/images/test1.qcow2` and `sudo chmod 660 /var/lib/libvirt/images/test1.qcow2`. Only needed if this error appears. |
| Guest is extremely slow | Nested virtualization is not passed through. On the host: `grep -c vmx /proc/cpuinfo`. A result of 0 means it is not. |

## Notes

- **The folder used above is the `default` pool's path** (`/var/lib/libvirt/images`). If you use another pool from `rhel_kvm_storage_pools`, put the image in that pool's `path` and refresh that pool (`pool-refresh <name>`). Do not use a path inside `/home`: the `qemu` user cannot enter a home folder, so the VM would not start.
- The import uses the copied file as the VM's own disk, so a test writes into it. To keep a clean copy on the host, copy it before importing: `sudo cp /var/lib/libvirt/images/test1.qcow2 /var/lib/libvirt/images/golden.qcow2 && sudo restorecon -v /var/lib/libvirt/images/golden.qcow2`.
- Copy into the home directory first, never straight into `/var/lib/libvirt/images`: `frqadmin` cannot write there, and a file copied in that way gets the wrong SELinux label.
- After a rollback, run step 0 again: the image and `edk2-ovmf` are gone with the snapshot state.

## Clean up the test VM

Not part of the tested steps. This deletes the VM and its disk, on either host:
```bash
sudo virsh -c qemu:///system destroy test1
sudo virsh -c qemu:///system undefine test1 --nvram --remove-all-storage
```
