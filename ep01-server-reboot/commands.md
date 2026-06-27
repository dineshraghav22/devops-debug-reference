# EP01 — Server Rebooted at 3AM. No Alert. No One Knows Why.

How to forensically reconstruct an unplanned reboot — kernel logs, disk health, smartd monitoring — and make sure you catch the next one before it takes the server down.

Watch the episode: https://youtube.com/@raghavdinesh22-cloud

---

## Step 1 — Establish the Timeline

```bash
# When did the server last reboot?
last reboot

# How long has it been up since?
uptime
```

---

## Step 2 — Find Errors Around the Reboot Time

```bash
# Check the previous boot for errors (priority: error and above)
journalctl -b -1 -p err

# Narrow to the exact window around the reboot
journalctl -b -1 --since "06:08" --until "06:12"

# Check syslog for anything suspicious
cat /var/log/syslog | grep -i error | tail -20
```

---

## Step 3 — Check for Hardware / Disk Errors

```bash
# Kernel ring buffer — look for disk controller / I/O errors
dmesg -T | grep -iE 'ata|sda|error|fail' | tail -10
```

---

## Step 4 — Check Disk Health with SMART

```bash
# Install smartmontools if not present
apt install smartmontools -y 2>&1 | tail -2

# Full SMART report — health, pending sectors, CRC errors
smartctl -a /dev/sda | grep -A2 -E 'health|Pending|Uncorrect|CRC'
```

**What to look for:**
- `SMART health: FAILED` — disk is dying, replace immediately
- `Reallocated_Sector_Ct` nonzero — sectors are failing
- `Current_Pending_Sector` nonzero — unstable sectors, reads may fail
- `Offline_Uncorrectable` nonzero — sectors that couldn't be recovered
- `UDMA_CRC_Error_Count` nonzero — cable or controller issue

---

## Step 5 — Set Up Continuous Disk Monitoring

```bash
# Enable smartd to monitor disk health in background
systemctl enable smartd --now && systemctl status smartd | head -5

# View smartd config (controls what gets monitored and alerted)
cat /etc/smartd.conf
```

**Recommended smartd config line for email alerts:**
```
/dev/sda -a -o on -S on -s (S/../.././02|L/../../6/03) -m root -M exec /usr/share/smartmontools/smartd-runner
```

---

## Root Cause Decision Tree

```
last reboot → When exactly?
     |
     v
journalctl -b -1 -p err → Any kernel errors?
     |
     +-- ata/sda/I/O errors → smartctl -a /dev/sda (disk dying)
     |
     +-- oom / kill --------→ dmesg -T | grep -i oom (OOM killer — see EP02)
     |
     +-- kernel panic ------→ dmesg -T | grep -i panic
     |
     +-- no kernel errors --→ journalctl -b -1 -u SERVICE (app crash?)
     |
     +-- nothing at all ----→ check power / hypervisor / watchdog logs
```
