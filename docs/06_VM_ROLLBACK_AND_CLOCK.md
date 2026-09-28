# VM rollback and the clock

After a snapshot rollback, the VM's clock can be far behind the real time. A wrong clock breaks package signature checks and Red Hat repository certificates, so it must be fixed before the playbook runs (lessons item 9).

Symptom seen: `timedatectl` showed the system clock a day behind the RTC, and `System clock synchronized: yes` (a stale flag). `chronyc makestep` answered `200 OK` and changed nothing, because chrony had no time source yet.

There are two parts: a one-time setup before taking the snapshot, and a fix to run after every rollback.

---

## A. One-time setup, before taking the snapshot (on each VM)

Let chrony jump the clock at any time, not only in its first 3 updates:
```bash
grep -n makestep /etc/chrony.conf                         # see the current line
sudo sed -i 's/^makestep.*/makestep 1.0 -1/' /etc/chrony.conf
grep -n makestep /etc/chrony.conf                         # must now say: makestep 1.0 -1
```
If the first `grep` printed nothing, there is no line to replace, so add one instead:
```bash
echo 'makestep 1.0 -1' | sudo tee -a /etc/chrony.conf
```
Apply it and check the clock:
```bash
sudo systemctl restart chronyd
sleep 15
chronyc tracking | grep -E "Leap|System time"
timedatectl                                               # time and date must be the real, current ones
```
Take the snapshot only when `timedatectl` shows the correct time. A snapshot taken with a wrong clock brings the wrong clock back every time.

## B. Hypervisor level (not tested on this setup)

- **Take the snapshot without RAM** if the hypervisor gives that choice ("Include RAM" unchecked on Proxmox). A snapshot with RAM restores the guest's memory, including its stale clock. A disk-only snapshot makes the VM boot fresh and read the current time from the hypervisor.
- **Install and enable the guest agent** (`sudo dnf install qemu-guest-agent`, and enable it in the VM's options). The hypervisor can then set the guest's time after a restore.

---

## C. After every rollback, run this

On one host:
```bash
sudo hwclock --hctosys                 # set the clock from the hardware clock
sudo systemctl restart chronyd
sleep 15
sudo chronyc makestep
date                                   # must show the real current date and time
```
Both hosts from the control node:
```bash
for h in 192.168.20.30 192.168.20.40; do
  ssh -t -J frqadmin@192.168.10.160 frqadmin@$h \
    'sudo hwclock --hctosys && sudo systemctl restart chronyd && sleep 15 && sudo chronyc makestep && date'
done
```

### Why this order
- `hwclock --hctosys` copies the hardware clock (RTC) into the system clock. It needs no network. Check first that `timedatectl` shows a correct `RTC time`, so a wrong RTC is not copied.
- Restarting `chronyd` after that makes chrony poll its servers again. After a rollback it may have marked them offline.
- `chronyc makestep` works only when chrony has a time source to step from. That is why it did nothing on its own.

### If it still says `synchronized: no` after a minute
```bash
chronyc sources -v        # a line starting ^* or ^+ means a server was found; ^? means unreachable
chronyc activity          # number of sources online / offline
```
If the servers are unreachable, the problem is the network from that VM (DNS or routing), and no chrony command fixes it. The clock is still right from `hwclock`, which is what matters for signatures and certificates.

---

## Notes

- The CIS roles rewrite `makestep` back to `1.0 3` on hardened hosts (`rhel9cis_chrony_server_makestep`, `rhel10cis_chrony_server_makestep`). The rollback snapshots are taken before hardening, so the setup in part A applies to them.
- After every rollback, also check `date` on **both** hosts before the first run: a host that passed last time can still be wrong.
