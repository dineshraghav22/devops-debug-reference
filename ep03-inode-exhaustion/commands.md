# EP03 — Disk Full on a Server With Plenty of Space

**Inode exhaustion** — df says 60% used, but you cannot create a single new file.

Watch the episode: https://youtube.com/@raghavdinesh22-cloud

---

## Step 1 — Confirm inode exhaustion

```bash
# This is the command most engineers forget exists
df -i

# IFree = 0 and IUse% = 100% confirms inode exhaustion
# Disk space (df -h) being fine is irrelevant
```

---

## Step 2 — Find the culprit directory

```bash
# Drill down from root — find which top-level dir has the most files
for dir in /*; do echo "$(find $dir -xdev 2>/dev/null | wc -l) $dir"; done | sort -rn | head -10

# Drill into the offending directory (e.g. /var)
for dir in /var/*; do echo "$(find $dir -xdev 2>/dev/null | wc -l) $dir"; done | sort -rn | head -8

# Keep drilling until you find the exact directory
for dir in /var/spool/*; do echo "$(find $dir -xdev 2>/dev/null | wc -l) $dir"; done | sort -rn | head -5
```

---

## Step 3 — Identify what the files are

```bash
# For postfix deferred queue — inspect a sample
find /var/spool/postfix/deferred -type f | head -3 | xargs postcat 2>/dev/null | grep -E 'To:|From:|Subject:'

# For PHP sessions — check age
find /var/lib/php/sessions -type f -mtime +1 | wc -l

# For any directory — see the oldest and newest files
find /path/to/dir -type f -printf '%T+ %p\n' | sort | head -5
find /path/to/dir -type f -printf '%T+ %p\n' | sort | tail -5
```

---

## Step 4 — Free the inodes

```bash
# Postfix deferred queue — clear all deferred messages
postsuper -d ALL deferred

# PHP sessions — delete sessions older than 24 hours
find /var/lib/php/sessions -type f -mtime +1 -delete

# Generic temp files older than 1 day
find /path/to/dir -type f -mtime +1 -delete

# Verify recovery
df -i
touch /tmp/test && echo "file created OK"
```

---

## Prevention

### Monitor inode usage (add to your monitoring stack)
```bash
# Alert if any filesystem inode usage exceeds 80%
df -i | awk 'NR>1 {gsub("%","",$5); if($5+0 > 80) print "ALERT: "$6" inode usage: "$5"%"}'
```

### Fix cron mail — redirect output to a log file instead
```bash
# Bad — generates mail on every run
0 6 * * * /opt/myapp/scripts/daily-report.sh

# Good — output goes to log file, no mail
0 6 * * * /opt/myapp/scripts/daily-report.sh >> /var/log/daily-report.log 2>&1
```

### Cap postfix queue lifetime so messages don't pile up forever
```bash
postconf -e 'maximal_queue_lifetime = 1d'
postconf -e 'bounce_queue_lifetime = 1d'
postconf -e 'qmgr_message_active_limit = 1000'
postfix reload
```

### Disable postfix if you don't need it
```bash
systemctl disable postfix --now
```

---

## Decision Tree

```
"No space left on device" error
         |
         v
df -h shows > 90% disk usage?
  YES --> Real disk space issue
          find / -xdev -size +100M -ls 2>/dev/null
  NO  --> Run df -i immediately
         |
         v
df -i shows IUse% = 100%?
  YES --> Inode exhaustion
          Find culprit: for dir in /var/*; do echo $(find $dir -xdev | wc -l) $dir; done | sort -rn
         |
         v
Common culprits:
  /var/spool/postfix/deferred  --> postsuper -d ALL deferred
  /var/lib/php/sessions        --> find ... -mtime +1 -delete
  /tmp or app temp dirs        --> find ... -mtime +1 -delete
  Many small log files         --> consolidate logging, add rotation
```
