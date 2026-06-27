# EP02 — Your Process Disappeared. No Error. No Crash Dump.

**OOM Killer** — how to prove it fired, find the root cause, and stop it happening again.

Watch the episode: https://youtube.com/@raghavdinesh22-cloud

---

## Commands Used in This Episode

### Confirm the process was killed (not crashed)
```bash
systemctl status <service>
# Look for: code=killed, signal=KILL — not "exited"
```

### Find the OOM kill event
```bash
dmesg -T | grep -i 'oom\|killed process\|out of memory'
```

### Read the full memory snapshot at time of kill
```bash
dmesg -T | grep -A 30 'Out of memory' | head -35
```

### Cross-reference with persistent kernel log
```bash
journalctl -k --since "HH:MM" | grep -i oom
```

### Check current memory state
```bash
free -h
```

### See who is using the most memory right now
```bash
ps aux --sort=-%mem | head -10
```

### Watch memory pressure in real time
```bash
vmstat 1 5
# si/so columns = swap in / swap out — nonzero means you're in trouble
```

### Check OOM score of a running process
```bash
cat /proc/$(pgrep <process>)/oom_score
cat /proc/$(pgrep <process>)/oom_score_adj
# Higher score = more likely to be killed. Range: 0–1000
```

### Check OOM scores for all key services
```bash
for svc in myapp postgres nginx redis; do
  pid=$(pgrep -x $svc 2>/dev/null | head -1)
  [ -n "$pid" ] && printf "%-12s score=%s adj=%s\n" \
    $svc $(cat /proc/$pid/oom_score) $(cat /proc/$pid/oom_score_adj)
done
```

---

## 5 Layers of Defense

### Layer 1 — Add swap immediately (no reboot, no code change)
```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

### Layer 2 — Contain the blast radius with MemoryMax
```bash
# Add to /etc/systemd/system/<service>.service.d/override.conf
[Service]
MemoryMax=3G
MemorySwapMax=512M
OOMScoreAdjust=-300

systemctl daemon-reload && systemctl restart <service>
```

### Layer 3 — Protect critical processes (database, message queue)
```bash
# Add to postgres/redis systemd unit override:
[Service]
OOMScoreAdjust=-800
# Range: -1000 (never kill) to +1000 (kill first)
```

### Layer 4 — Cap JVM heap at source
```bash
# Add to ExecStart in systemd unit or JAVA_OPTS:
-Xms512m -Xmx2g -XX:MaxMetaspaceSize=256m -XX:+UseG1GC
```

### Layer 5 — Install earlyoom (kills before system freezes)
```bash
apt install earlyoom -y

# Edit /etc/default/earlyoom:
EARLYOOM_ARGS="-m 5 -s 5 --prefer myapp --avoid postgres"

systemctl enable earlyoom --now
```

---

## Root Cause Decision Tree

```
Is RSS growing over time?
  YES → Memory leak → profile with jmap / heaptrack / valgrind
  NO  → Spike or overcommit

Were multiple processes killed?
  YES → Overcommit → sysctl vm.overcommit_memory=2
  NO  → One process, no MemoryMax set → add MemoryMax

Was swap full before the kill?
  EMPTY → No safety net → add swap now
  FULL  → Leak ran long enough to exhaust both
```
