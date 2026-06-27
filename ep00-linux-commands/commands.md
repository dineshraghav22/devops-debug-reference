# EP00 — Linux Commands Every DevOps Engineer Must Know Cold

Essential shell commands for file inspection, log analysis, grep, sed, awk, and pipeline tricks — the baseline every DevOps engineer needs cold.

Watch the episode: https://youtube.com/@raghavdinesh22-cloud

---

## File Inspection

```bash
# List files — newest first
ls -lath /var/log/

# List files — largest first
ls -lahS /var/log/

# Full file metadata (all timestamps, inode, permissions)
stat /var/log/auth.log

# Find logs not touched in 7+ days
find /var/log -name "*.log" -mtime +7

# Find files larger than 100MB anywhere on the filesystem
find / -xdev -size +100M -ls 2>/dev/null

# Find world-writable files owned by root in /tmp
find /tmp -type f -user root -perm /o+w 2>/dev/null

# Clean compressed logs older than 30 days
find /var/log -name "*.gz" -mtime +30 -delete && echo "cleaned"
```

---

## Reading Files

```bash
# View file with line numbers
cat -n /etc/crontab

# Follow a log in real time (stops if file is rotated)
tail -f -n 50 /var/log/nginx/access.log

# Follow a log across rotation (keeps tracking the new file)
tail -F /var/log/nginx/access.log
```

---

## grep

```bash
# Recursive search with line numbers
grep -rn "authentication failure" /var/log/

# Invert match — show everything except INFO lines
grep -v "INFO" /var/log/app.log | tail -20

# Count matches
grep -c "Failed password" /var/log/auth.log

# Match multiple patterns, count total
grep -E "error|warn|fail|panic" /var/log/syslog | wc -l

# Show 3 lines after and 2 lines before each match
grep -A3 -B2 "kernel panic" /var/log/kern.log

# Search and strip date prefix with sed
grep -n "error\|crit" /var/log/nginx/error.log | sed 's/2026\/06\/15 //g'
```

---

## sed

```bash
# Print only lines 5–10
sed -n '5,10p' /var/log/nginx/error.log

# Strip comments and blank lines
sed '/^#/d' /etc/ssh/sshd_config | sed '/^$/d'

# Safe in-place edit with backup
sed -i.bak 's/PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config

# Verify what changed
diff /etc/ssh/sshd_config.bak /etc/ssh/sshd_config
```

---

## awk

```bash
# Print selected columns from ps output
ps aux | awk '{print $1, $2, $3, $11}' | head -8

# List all human users (UID >= 1000) from /etc/passwd
awk -F: '$3 >= 1000 {print $1, $3, $7}' /etc/passwd

# Show only log dirs using megabytes, sorted by size
du -sh /var/log/* | awk '$1 ~ /M/ {print $0}' | sort -rh

# Sum the last column of a file
awk '{sum += $NF} END {print "Total requests:", sum}' /var/log/nginx/access.log

# Sample every 100th line (useful for large logs)
awk 'NR % 100 == 0' /var/log/syslog | head -5
```

---

## Pipeline Tricks

```bash
# Output to screen AND save to file simultaneously
cat /var/log/nginx/error.log | grep -E 'error|crit' | tee /tmp/errors.log | wc -l

# Capture command output for inspection
systemctl restart nginx 2>&1 | tee /tmp/restart.log
cat /tmp/restart.log

# Pass filenames from find/grep to another command
grep -l "error" /var/log/*.log | xargs wc -l

# Chain find + delete safely
find /tmp -name "*.tmp" -mtime +1 | xargs rm -f && echo "cleanup done"

# Re-use the last argument of the previous command
ls /var/log/*.log && echo "---" && !$
```
