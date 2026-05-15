# 🐧 Linux Performance — Field Cheatsheet

> Quick reference for the tools used in this training.
> Keep this open in a second terminal.

---

## `ps` — Process Snapshot

```bash
ps aux                          # All processes: user, CPU, MEM, state, command
ps auxf                         # Same, with ASCII process tree
ps -eo pid,ppid,stat,comm       # Custom columns: PID, parent, state, name
ps -eo pid,ppid,stat,wchan,args # Add wait-channel (what kernel fn it's blocked in)

# Filter by name
ps aux | grep nginx

# Show a specific PID
ps -p <PID> -o pid,ppid,stat,vsz,rss,comm
```

### STAT column decoded

| Code | Meaning |
|------|---------|
| `R`  | Running or runnable (on CPU or in run queue) |
| `S`  | Interruptible sleep (waiting for event — can receive signals) |
| `D`  | Uninterruptible sleep (waiting on I/O — **cannot** be killed) |
| `T`  | Stopped (SIGSTOP or debugger) |
| `Z`  | Zombie (exited, waiting for parent to call waitpid) |
| `+`  | Foreground process group |
| `s`  | Session leader |
| `l`  | Multi-threaded |

---

## `/proc` — Live Kernel State

```bash
# Process status (state, memory, threads, signals)
cat /proc/<PID>/status

# Key fields in /proc/<PID>/status
grep -E 'State|VmSize|VmRSS|VmSwap|Threads|SigCgt' /proc/<PID>/status

# What kernel function a process is sleeping in
cat /proc/<PID>/wchan

# Full command line (argv[0] through argv[n])
cat /proc/<PID>/cmdline | tr '\0' ' '

# Open file descriptors
ls -la /proc/<PID>/fd

# Memory map (virtual address space)
cat /proc/<PID>/maps

# Detailed per-region memory accounting
cat /proc/<PID>/smaps_rollup

# System-wide memory
cat /proc/meminfo | grep -E 'MemTotal|MemAvailable|SwapTotal|SwapFree'
```

### Key /proc/meminfo fields

| Field | Meaning |
|-------|---------|
| `MemTotal` | Total physical RAM |
| `MemAvailable` | RAM available for new allocations (better than MemFree) |
| `MemFree` | RAM not used at all (excludes reclaimable cache) |
| `Cached` | Page cache (reclaimable) |
| `SwapTotal` / `SwapFree` | Swap space |
| `Dirty` | Pages waiting to be written to disk |

---

## `vmstat` — System-Wide Memory + CPU

```bash
vmstat 1              # Update every 1 second (forever)
vmstat 1 30           # 30 samples, 1 second apart
vmstat -s             # Summary totals
vmstat -m             # Slab allocator stats
```

### vmstat output columns

```
procs --------memory--------- --swap- ---io-- -system- ------cpu-----
 r  b   swpd   free  buff cache  si  so   bi   bo   in   cs us sy id wa
```

| Column | Meaning |
|--------|---------|
| `r` | Processes in run queue (waiting for CPU) |
| `b` | Processes in uninterruptible sleep (D state) |
| `swpd` | Virtual memory used (kB) |
| `free` | Idle memory (kB) |
| `buff` | Memory used as buffer (kB) |
| `cache` | Memory used as cache (kB) |
| `si` | Swap-in rate (kB/s — pages read from disk) |
| `so` | Swap-out rate (kB/s — pages written to disk) |
| `bi` | Blocks received from block device (blocks/s) |
| `bo` | Blocks sent to block device (blocks/s) |
| `us` | CPU user time % |
| `sy` | CPU kernel/system time % |
| `id` | CPU idle % |
| `wa` | CPU I/O wait % |

> **Rule of thumb:** `si`/`so` > 0 means you're swapping — performance will degrade.
> `wa` > 20% means I/O is a bottleneck.

---

## `pidstat` — Per-Process Statistics (sysstat)

```bash
# Install sysstat if needed
sudo apt-get install -y sysstat

# CPU usage, 1-second samples
pidstat -u 1

# Memory (RSS, VSZ), 1-second samples
pidstat -r 1

# Disk I/O per process, 1-second samples
pidstat -d 1

# All metrics combined
pidstat -urd 1

# Save to file for later analysis
pidstat -u 1 | tee /tmp/cpu_log.txt
pidstat -r 1 | tee /tmp/mem_log.txt

# Filter for a specific PID
pidstat -p <PID> -u 1
```

### pidstat -u columns (CPU)

| Column | Meaning |
|--------|---------|
| `%usr` | CPU time in user space |
| `%system` | CPU time in kernel |
| `%guest` | CPU time in virtual machine |
| `%wait` | Time waiting for CPU (high = CPU contention) |
| `%CPU` | Total CPU usage |
| `Command` | Process name (from `/proc/pid/comm`) |

### pidstat -r columns (Memory)

| Column | Meaning |
|--------|---------|
| `minflt/s` | Minor page faults per second (no disk I/O) |
| `majflt/s` | Major page faults per second (disk read required) |
| `VSZ` | Virtual memory size (kB) — reserved, not necessarily in RAM |
| `RSS` | Resident Set Size (kB) — **actually in physical RAM** |
| `%MEM` | RSS as % of total RAM |

> **Key insight:** `RSS` is your real memory usage. `VSZ` can be much larger —
> virtual memory is cheap until you actually touch the pages.

---

## `iostat` — Disk I/O Statistics (sysstat)

```bash
iostat -xz 1          # Extended stats, skip idle devices, 1s interval
iostat -xz 1 10       # 10 samples

# Human-readable throughput
iostat -h 1
```

### iostat -x key columns

| Column | Meaning |
|--------|---------|
| `r/s` | Read requests per second |
| `w/s` | Write requests per second |
| `rkB/s` | Read throughput (kB/s) |
| `wkB/s` | Write throughput (kB/s) |
| `await` | Average I/O wait time (ms) — **the key latency metric** |
| `r_await` | Average read latency (ms) |
| `w_await` | Average write latency (ms) |
| `%util` | Device utilisation % — 100% = saturated |

> **Rule of thumb:** `await` > 20ms on SSD = problem. `%util` near 100% = bottleneck.

---

## `sar` — Historical System Activity (sysstat)

```bash
# CPU — 1-second intervals, 10 samples
sar -u 1 10

# Memory — 1-second intervals
sar -r 1 10

# Swap activity
sar -S 1 10

# Network
sar -n DEV 1 10

# Read today's recorded data (sysstat cron job)
sar -u -f /var/log/sysstat/saXX     # XX = day of month
sar -r -f /var/log/sysstat/saXX
```

---

## `free` — Memory Overview

```bash
free -h               # Human-readable (MB/GB)
free -m               # Megabytes
watch -n1 free -h     # Refresh every second
```

### free output

```
              total   used    free   shared  buff/cache  available
Mem:           7.8G   1.2G    5.1G    123M      1.4G       6.3G
Swap:          2.0G     0B    2.0G
```

> **Use `available`, not `free`** — available includes reclaimable cache.
> `free` alone is misleading on Linux (cache counts as "used" but is reclaimable).

---

## `top` / `htop` — Interactive Process Monitor

```bash
top                   # Classic interactive view
htop                  # Colourful interactive view (sudo apt install htop)

# top useful keys
P     # Sort by CPU
M     # Sort by memory
k     # Kill a process (enter PID)
1     # Toggle per-CPU vs aggregate
d     # Change refresh interval
q     # Quit
```

---

## `kill` — Send Signals

```bash
kill <PID>            # Send SIGTERM (15) — polite request to stop
kill -TERM <PID>      # Same as above
kill -KILL <PID>      # Send SIGKILL (9) — immediate, cannot be caught
kill -STOP <PID>      # Freeze a process (SIGSTOP — cannot be caught)
kill -CONT <PID>      # Resume a frozen process (SIGCONT)
kill -USR1 <PID>      # Send SIGUSR1 (application-defined)
kill -USR2 <PID>      # Send SIGUSR2 (application-defined)

# Kill by name
pkill nginx
killall nginx

# List all signal names
kill -l
```

### Signal quick reference

| Signal | Number | Catchable | Meaning |
|--------|--------|-----------|---------|
| `SIGTERM` | 15 | ✅ Yes | Graceful shutdown request |
| `SIGKILL` | 9  | ❌ No  | Immediate kill (kernel-delivered) |
| `SIGSTOP` | 19 | ❌ No  | Freeze process |
| `SIGCONT` | 18 | ✅ Yes | Resume stopped process |
| `SIGINT`  | 2  | ✅ Yes | Interrupt (Ctrl-C) |
| `SIGTSTP` | 20 | ✅ Yes | Terminal stop (Ctrl-Z) |
| `SIGHUP`  | 1  | ✅ Yes | Hangup / reload config |
| `SIGCHLD` | 17 | ✅ Yes | Child state changed |
| `SIGUSR1` | 10 | ✅ Yes | User-defined |
| `SIGUSR2` | 12 | ✅ Yes | User-defined |

---

## `strace` — System Call Tracer

```bash
# Trace a running process
strace -p <PID>

# Trace specific syscalls only
strace -e trace=read,write,open,close -p <PID>

# Trace with timestamps
strace -T -p <PID>

# Run a new command under strace
strace ls /tmp

# Summary: count syscalls
strace -c ls /tmp
```

---

## `lsof` — Open Files and Sockets

```bash
lsof -p <PID>         # All open files for a process
lsof -i               # All network connections
lsof -i :80           # Who's listening on port 80
lsof -u sam           # All files opened by user sam
lsof /var/log/syslog  # Who has this file open
```

---

## Quick Diagnostic Workflow

```bash
# 1. What's using CPU right now?
pidstat -u 1 60 | tee /tmp/cpu.log

# 2. What's using memory?
pidstat -r 1 30 | tee /tmp/mem.log
# Then: awk '$6 > 100000' /tmp/mem.log   (RSS > 100 MB)

# 3. Is the system swapping?
vmstat 1 10
# Watch: si, so columns — any value > 0 is concerning

# 4. Is disk I/O a bottleneck?
iostat -xz 1 10
# Watch: await > 20ms, %util approaching 100%

# 5. Find processes in bad states
ps aux | awk '$8 ~ /^[DZ]/'    # D-state or zombie processes

# 6. What is a specific process waiting on?
cat /proc/<PID>/wchan
cat /proc/<PID>/status | grep State

# 7. How much memory is a process really using?
cat /proc/<PID>/status | grep -E 'VmRSS|VmSwap|VmSize'
cat /proc/<PID>/smaps_rollup

# 8. Is a process leaking memory? (watch RSS over time)
watch -n2 'cat /proc/<PID>/status | grep VmRSS'
```
