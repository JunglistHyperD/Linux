# Observability Tool Reference — Chapter 4

Every tool named in Gregg's Observability Tools chapter, sorted into its categories and drilled with the commands that matter — parameters shown as `<CAPITALS>` and explained. A standing reference to keep open while troubleshooting.

## How to read this guide

Every tool below carries a tag line using the chapter's own axes, so you can see at a glance what kind of instrument it is. The dimensions:

- **Scope** — *system-wide* (the whole machine) or *per-process* (one PID).
- **Data** — *counters* (fixed running totals the kernel always keeps; effectively free to read) or *event-based* (instrumenting activity). Event-based splits into *profiling* (sampling a subset, coarse but cheap) and *tracing* (capturing every event, detailed but cost scales with event rate).
- **Role** — *observability* (watching the system work) or *static* (reading configuration at rest, to catch misconfiguration).
- **Instrumentation** (for tracers) — *static* (hard-coded probes: tracepoints, USDT — stable, safe to build on) or *dynamic* (added to running code: kprobes, uprobes — can see anything, but brittle across kernel versions).
- **Cost** — free (counters), predictable (sampling), or scales-with-events (tracing).

So a tag reading *system-wide · event-based (tracing) · dynamic · cost scales with events* tells you exactly what you're pointing at a production box before you run it.

The four core buckets, mapped onto scope × data type:

| | Counters (cheap, always on) | Events (profiling / tracing) |
| --- | --- | --- |
| **System-wide** | vmstat, mpstat, iostat, free, nstat, sar | perf, Ftrace, BCC, bpftrace, tcpdump |
| **Per-process** | ps, top, pidstat, pmap | strace, gdb |

Many tools straddle buckets — top(1) has a system-wide summary, and system-wide event tools usually filter to one PID — but the quadrant is the mental model: start in the cheap counter corners to localise, cross into events once you know where to point them.

---

## System-wide counter tools

*Scope: system-wide · Data: fixed counters · Role: observability · Cost: free.* These read kernel counters (mostly from /proc) and summarise the whole machine. Cheap, always-available, and where every investigation starts. Most take an **\<INTERVAL\> \<COUNT\>** convention: print every \<INTERVAL\> seconds, \<COUNT\> times, then exit; the first line is usually a since-boot summary, so watch the *second* line onward for current activity.

### vmstat

Virtual-memory, CPU, and I/O summary for the whole system on one line per interval. Your fastest read on swapping, run-queue length, and the user/system/idle CPU split.

```bash
vmstat [-S <UNIT>] <INTERVAL> <COUNT>
vmstat -S M 1 5
```

- **\<INTERVAL\>** — seconds between samples (1 = per second).
- **\<COUNT\>** — how many samples to print, then exit (omit to run until Ctrl-C).
- **-S \<UNIT\>** — memory unit: `M` for MiB, `k` for KiB. Easier to read than the default.
- Watch: `r` (run-queue length — threads wanting CPU), `si`/`so` (swap in/out — any non-zero is memory pressure), `us`/`sy`/`id`/`wa` (CPU breakdown).

**Reading the output** — sample from an idle Ubuntu VM:

```sh
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 0  0      0  12181     83   2105    0    0   221    16 1151    2  1  1 98  0  0  0   <- since-boot average, ignore
 0  0      0  12181     83   2105    0    0     0     0  666  745  1  2 97  0  0  0
 0  0      0  12181     83   2105    0    0     0     0  523  527  3  1 96  0  0  0
```

CPU columns are a % of total CPU time across all cores (a row sums to ~100):

| Col | Meaning | High value means |
| --- | --- | --- |
| `us` | user — running application code | An app is CPU-bound → find it with `top` / `pidstat` |
| `sy` | system — in the kernel (syscalls, interrupts, scheduling) | Chatty I/O, heavy networking, lock contention |
| `id` | idle | Headroom — low `id` + `r` > core count = CPU saturation |
| `wa` | idle *while* waiting on I/O | Disk bottleneck → `iostat -sxz 1` |
| `st` | steal — vCPU wanted to run, hypervisor gave the core elsewhere | Noisy neighbour / overcommitted host — **check on every VM**, nothing inside the guest fixes it |
| `gu` | guest — time running *this* box's own VMs (newer procps) | Only non-zero on a KVM host |

The rest at a glance:

- **procs** — `r` runnable (running + queued for CPU); `b` blocked in uninterruptible sleep (usually I/O). Both 0 = quiet.
- **memory** (MiB with `-S M`) — `swpd` swap used, `free` totally unused, `buff`/`cache` page cache (reclaimable, effectively available).
- **swap** — `si`/`so` pages swapped in/out per second; sustained non-zero = short on RAM.
- **io** — `bi`/`bo` blocks read/written per second.
- **system** — `in` interrupts/s, `cs` context switches/s.

Verdict on the sample: ~97% idle, no `wa`, no `st`, no swapping, ~12 GiB free — nothing to chase. To watch the columns move, run `stress --cpu 2` in another terminal alongside `vmstat 1`.

**In the field — has this box actually swapped since boot?**

```bash
vmstat -s | grep -i swap
```

### mpstat

Per-CPU utilisation breakdown — the tool that exposes a single hot core hiding inside a healthy-looking average.

```bash
mpstat -P ALL <INTERVAL>
mpstat -P ALL 1
```

- **-P ALL** — report every CPU separately (`-P 0` would report only CPU 0). Without it you get only the aggregate.
- **\<INTERVAL\>** — seconds between reports.
- Watch: one CPU near 100% `%usr`+`%sys` while others idle = poor thread scaling or a single-threaded bottleneck.

**Reading the output** — sample from an idle 2-vCPU Ubuntu VM (`mpstat -P ALL 1 3`):

```text
07:12:13 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
07:12:14 PM  all    0.52    0.00    1.05    0.00    0.00    1.05    0.00    0.00    0.00   97.38
07:12:14 PM    0    1.04    0.00    1.04    0.00    0.00    1.04    0.00    0.00    0.00   96.88
07:12:14 PM    1    0.00    0.00    1.05    0.00    0.00    1.05    0.00    0.00    0.00   97.89
```

`all` = average across cores (same as vmstat's CPU split); `0`, `1` = each core; a final `Average:` block summarises the run. Unlike vmstat there's no since-boot row — every block is live. Each column is % of that CPU's time (row sums to ~100):

| Col | Meaning | High value means |
| --- | --- | --- |
| `%usr` | Running user code | App CPU-bound → `top` / `pidstat` |
| `%nice` | User code at lowered priority (`nice`) | Background/batch jobs soaking spare CPU — usually harmless |
| `%sys` | In the kernel (syscalls, scheduling, drivers) | Chatty I/O, many small syscalls, lock contention |
| `%iowait` | Idle *while* waiting on disk I/O | Storage bottleneck → `iostat -sxz 1` |
| `%irq` | Hardware interrupt handling (NIC, disk, timer) | A device flooding interrupts |
| `%soft` | Softirqs — deferred kernel work, mostly network packets | Heavy network load, often pinned to one core |
| `%steal` | vCPU ready, hypervisor gave the core elsewhere | Overcommitted host — nothing inside the guest fixes it |
| `%guest` / `%gnice` | Running this box's own VMs (normal / niced) | Only non-zero on a KVM host |
| `%idle` | Nothing to do, no I/O pending | Headroom |

vmstat folds `%nice`/`%irq`/`%soft` into `us`/`sy`; mpstat splits them out — that's what exposes interrupt-bound cores.

Verdict on the sample: both cores ~97.5% idle and balanced, no iowait or steal — nothing to chase. A problem looks like `all` at 50% hiding one core at 0% idle (single-threaded app), or one core with high `%soft` while the others idle (NIC interrupts on one CPU — confirm below and with `cat /proc/interrupts`).

**In the field — is a NIC interrupt pinning one core?**

```bash
mpstat -I SUM -P ALL 2
```

**What a problem looks like:**

```text
CPU   %usr  %sys  %soft  %idle
all   50.0   ...    ...   50.0     <- average looks fine
  0  100.0   0.0    0.0    0.0     <- one core pinned (single-threaded app)
  1    0.0   0.0    0.0  100.0
```

Or one core with high `%soft` while the others idle. That points to network interrupts landing on one CPU. Check with `mpstat -I SUM -P ALL 1` and `cat /proc/interrupts`.

### iostat

Per-disk I/O statistics from the block layer: IOPS, throughput, wait time, and percent busy.

```bash
iostat -sxz <INTERVAL>
iostat -sxz 1
```

- **-x** — extended stats (adds `await`, `%util`, queue size — the useful columns).
- **-s** — short/compact output.
- **-z** — omit devices with zero activity (cuts the noise).
- **\<INTERVAL\>** — seconds between reports.
- Watch: `%util` near 100 with rising `await` = a saturated disk; compare `r/s`+`w/s` (IOPS) against throughput.

**Reading the output** — sample from an idle Ubuntu VM (`iostat -sxz -t 2 5 /dev/sda`):

```text
11/12/2026 07:17:57 PM                                      <- -t timestamp
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.69    0.03    1.14    0.01    0.00   98.12

Device             tps      kB/s    rqm/s   await  areq-sz  aqu-sz  %util
sda               3.62    208.30     2.00    0.54    57.56    0.00   0.10   <- since-boot average, ignore

11/12/2026 07:18:01 PM
...
Device             tps      kB/s    rqm/s   await  areq-sz  aqu-sz  %util
sda               2.00      8.00     0.00    1.25     4.00    0.00   0.05
```

- **First report = since boot** (the 208 kB/s is boot-time reads); live data starts at the second.
- **Empty `Device` block** = `-z` hiding a device that did no I/O in that interval — not an error.
- **`avg-cpu`** — same CPU split as vmstat/mpstat; `%iowait` is the one to cross-check against the disk stats.

Device columns (`-s` short form merges reads and writes — drop `-s` to see `r/s`, `w/s`, `r_await`, `w_await` separately):

| Col | Meaning | Worry when |
| --- | --- | --- |
| `tps` | I/O requests completed per second (IOPS) | Near the device's rated IOPS |
| `kB/s` | Throughput, read + write | Near the device's rated bandwidth |
| `rqm/s` | Requests merged per second (adjacent I/Os combined before dispatch) | Informational — high = sequential workload |
| `await` | Avg ms per I/O, **queue time + service time** — what the app feels | Sustained > ~10 ms on SSD, > ~20 ms on HDD |
| `areq-sz` | Avg request size in KB | Small (4 KB) = random I/O; large = sequential |
| `aqu-sz` | Avg number of requests queued/in flight | Sustained > 1 on a single disk = I/O backing up |
| `%util` | % of time the device had at least one I/O in flight | Near 100 on a spinning disk = saturated |

**`%util` caveat:** on SSD, NVMe, RAID, or cloud volumes that serve many I/Os in parallel, 100% only means "never idle", not "full". Judge those by `await` and `aqu-sz` rising instead.

Verdict on the sample: 1–2 tiny (4–6 KB) requests every couple of seconds, ~1 ms latency, nothing queued, `%util` ≈ 0 — the disk is idle. Saturation looks like `await` climbing with `aqu-sz` > 1 and `%iowait` rising in the CPU block; then find the culprit with `pidstat -d 1` or `biosnoop`.

**In the field — one indexer's disk suspect during a bucket roll; timestamp it and watch just that device:**

```bash
iostat -sxz -t 2 /dev/nvme1n1
```

### free

System memory usage, including the all-important file-system cache.

```bash
free [-h | -m]
free -m
```

- **-m** — show values in MiB (`-g` for GiB).
- **-h** — human-readable, auto-scaled units.
- Watch: `available` (what apps can actually get) and `buff/cache` (reclaimable cache — a shrinking cache often precedes disk-read pain).

**In the field — watch the page cache shrink live as a search runs:**

```bash
watch -n1 free -m
```

### nstat

TCP/IP stack counters — a cleaner, delta-based alternative to the legacy netstat statistics.

```bash
nstat [-n | -a | -z] [<PATTERN>...]
nstat
nstat -a TcpOutSegs TcpRetransSegs
```

- Each run prints the **delta since the previous run** (kept in a per-user history file), then updates the history.
- **-n** — print nothing, just update the history: sets a baseline for a measurement window.
- **-a** — absolute totals since boot instead of the delta.
- **-z** — also show counters that are zero (normally hidden). *Not* a reset — `-r` resets the history.
- **\<PATTERN\>** — optional counter names to show (globs allowed), e.g. `TcpRetransSegs`.
- Watch: retransmit and error counters climbing during a network complaint.

**In the field — measure TCP retransmits over a 10-second window:**

```bash
nstat -n; sleep 10; nstat | grep -iE 'retrans|drop'
```

Baseline, wait, then print only what changed in those 10 s. No output = nothing retransmitted or dropped. Columns: counter name, delta over the window, smoothed rate (ignore unless running `-d`).

| Counter | Meaning |
| --- | --- |
| `TcpRetransSegs` | Total segments resent — the headline number |
| `TcpExtTCPFastRetrans` | Resent after duplicate ACKs — mild loss, quick recovery |
| `TcpExtTCPLostRetransmit` / `TcpExtTCPSlowStartRetrans` | Timeout-driven or repeat resends — connections stalling |
| `TcpExtTCPSynRetrans` | Connection attempts resent — far end unreachable, overloaded, or firewalled |
| `TcpExtListenDrops` / `TcpExtListenOverflows` | **This host's** listen queue full — app not `accept()`ing fast enough or `somaxconn` too low |
| `TcpExtTCPBacklogDrop` / `TcpExtTCPRcvQDrop` | Socket receive buffer full — app not reading fast enough |

Judge retransmits as a ratio: `TcpRetransSegs` ÷ `TcpOutSegs` above ~1% on a LAN = real loss (cable/NIC, congested switch, duplex mismatch → `ethtool`). Listen/backlog drops are a server-side problem, not the network. nstat is system-wide — to find *which* connection, use `ss -ti` and check per-socket `retrans:`.

## Per-process counter tools

*Scope: per-process · Data: fixed counters · Role: observability · Cost: free.* These read the kernel's per-PID counters (from /proc/\<PID\>) to show which processes are consuming what. Where you go once a system-wide tool says “CPU is busy” and you need to know *who*.

### ps

A snapshot of process status and statistics — one shot, not a live feed. Best for scripting and for a quick sorted list.

```bash
ps -eo pid,comm,%cpu,%mem,rss --sort=-%cpu
ps aux --sort=-%cpu
```

- **-e** — every process (system-wide, not just your shell's).
- **-o \<FIELDS\>** — choose exactly which columns to print (`pid`, `comm` = command, `%cpu`, `%mem`, `rss` = resident memory in KiB). Precise output beats the default clutter. Field names must be **lowercase** — `%CPU` fails with *improper AIX field descriptor* (ps reads `%C` as an old AIX format code).
- **RSS (Resident Set Size)** — the part of a process's memory physically in RAM right now (KiB), as opposed to `VSZ`, everything mapped whether touched or not. Shared library pages count in full in *every* process's RSS, so summed RSS overstates real use and killing a process frees less than its RSS. For a fair split use **PSS** (shared pages divided among users); for what killing it frees, **USS**/`Private_Dirty` — see `pmap -X`, `/proc/<PID>/smaps_rollup`, `smem`. In `top` it's the `RES` column.
- **--sort=-\<KEY\>** — sort by \<KEY\>; the leading `-` means descending (so `-%cpu` puts the heaviest first).
- `aux` is the classic BSD-style form: all processes, with user and resource columns.
- **`%CPU` gotcha:** it's CPU time ÷ process lifetime — a *lifetime average*, not current usage. A process busy right now can show 0.8%; one that spiked hours ago stays inflated. For "now", use `top` or `pidstat 1`. (`RSS`/`%MEM` *are* current.)
- ps often lists **itself** at an absurd `%CPU` (e.g. 500% on a 2-CPU box): a few ms of life rounded up to a 10 ms clock tick. Ignore that line. Legitimate values can exceed 100% for multi-threaded processes, up to 100% × cores.

**Reading the output** — sample from an Ubuntu VM running Jenkins:

```text
    PID COMMAND         %CPU %MEM   RSS
   5837 ps               400  0.0  4492     <- ps measuring itself, ignore
   2383 java             0.8  2.5 418796    <- Jenkins: ~409 MiB resident, biggest real consumer
   5640 kworker/1:0-eve  0.7  0.0     0     <- kernel thread: no user memory, so RSS 0
   2039 gnome-shell      0.6  1.8 303776
```

| Col | Meaning | Real-world note |
| --- | --- | --- |
| `PID` | Process ID | Feed it to `top -p`, `pidstat -p`, `pmap`, `strace -p` |
| `COMMAND` (`comm`) | Executable name, **truncated to 15 chars** | All JVMs just say `java` — use `args` to see the full command line |
| `%CPU` | Lifetime-average CPU (see gotcha above) | Ranking hint only, not current load |
| `%MEM` | `RSS` ÷ total RAM | Quick share-of-box view |
| `RSS` | Resident memory in KiB — what's actually in RAM | 418796 KiB ≈ 409 MiB; the column to sort on when hunting memory |

Extra columns in `ps aux`:

- **`VSZ`** — virtual size (all mapped address space, much never touched). Large VSZ alone is normal, especially for Java; watch `RSS`.
- **`STAT`** — state: `R` running, `S` sleeping, **`D` uninterruptible (stuck on I/O — counts toward load average)**, `Z` zombie, `T` stopped, `I` idle kernel thread. Modifiers: `s` session leader, `l` multi-threaded, `+` foreground, `<` high / `N` low priority.
- **`TTY`** — controlling terminal; `?` = daemon or kernel thread.
- **`START`** / **`TIME`** — start time / *total* CPU time consumed. `TIME` growing fast between two runs = genuinely busy now.

**In the field — find what's stuck in `D` state (blocked on I/O):**

```bash
ps -eo pid,stat,wchan:20,args | awk '$2 ~ /D/'
```

**In the field — full command line and age of a suspect process:**

```bash
ps -o pid,etime,time,rss,args -p <PID>
```

**In the field — what's eating memory right now?**

```bash
ps -eo pid,comm,rss,%mem --sort=-rss | head
```

### top

The live, auto-refreshing process table, sorted by CPU (or another key). Also carries a system-wide summary header, so it straddles the quadrant.

```bash
top -p <PID>
top -o %MEM
```

- **-p \<PID\>** — restrict to one or more PIDs (comma-separated) instead of the whole system.
- **-o \<FIELD\>** — sort by \<FIELD\> (e.g. `%MEM` to hunt memory, `%CPU` is the default).
- Interactive keys once running: `P` sort by CPU, `M` by memory, `1` expand per-CPU lines, `H` show threads.
- Note: on very busy, many-process systems, top's own /proc reads can make *it* a top consumer — a real gotcha.

**In the field — watch only the Splunk processes:**

```bash
top -p $(pgrep -d, splunkd)
```

#### top cheat sheet — confirm and name the culprit

The overview. The header repeats the earlier checks, and the list shows which process is responsible. Press `1` for per-core view and `M` to sort by memory.

```bash
top
```

`top` is on every Linux box and answers one question: which process is using the machine? Read the header first, then sort the list. Keys are case-sensitive.

##### Reading the header

| Line | What it tells you |
| --- | --- |
| `load average` | Same as `uptime`: 1, 5, 15 minutes. Compare to core count |
| `Tasks` | Totals. A growing `zombie` count means a parent isn't reaping children |
| `%Cpu(s)` | `us` user, `sy` kernel, `ni` nice, `id` idle, `wa` I/O wait, `hi`/`si` interrupts, `st` stolen by the hypervisor |
| `MiB Mem` | Watch `avail Mem` (on the Swap line), not `free`. Cache is reclaimable |
| `MiB Swap` | `used` rising over time means memory pressure |

##### Reading the columns

| Column | Meaning |
| --- | --- |
| `PR` / `NI` | Scheduling priority and nice value. Higher nice = lower priority |
| `VIRT` | Address space mapped. Often huge; rarely the problem |
| `RES` | Physical RAM in use. The memory figure that matters |
| `SHR` | Part of `RES` shared with other processes |
| `S` | State: `R` running, `S` sleeping, `D` waiting on disk (uninterruptible), `Z` zombie, `T` stopped |
| `%CPU` | Per-core share, so a multi-threaded process can exceed 100 |
| `TIME+` | Total CPU time used since it started |

Many processes in state `D` alongside high `wa` points to a disk problem.

##### Keys inside top

| Key | Does |
| --- | --- |
| `P` | Sort by CPU |
| `M` | Sort by memory |
| `T` | Sort by cumulative CPU time |
| `R` | Reverse the sort order |
| `1` | Show each CPU core separately |
| `c` | Toggle full command line |
| `H` | Show threads instead of processes |
| `V` | Tree (forest) view |
| `u` | Filter by user |
| `o` | Add a filter, e.g. `COMMAND=splunkd`. Press `=` to clear |
| `i` | Hide idle processes |
| `d` or `s` | Change refresh interval (seconds) |
| `Space` | Refresh now |
| `k` | Kill a process: enter PID, then signal (15 first, 9 last resort) |
| `r` | Renice a process (raising nice is safe; lowering needs root) |
| `f` | Choose and order columns |
| `E` / `e` | Cycle memory units in the header / in the list |
| `W` | Save your layout for next time |
| `q` | Quit |

##### Command-line options

```bash
top -b -n 1 | head -20
```

One snapshot to paste into a ticket (`-b` batch mode, `-n 1` one iteration).

```bash
top -u splunk
```

Only the `splunk` user's processes.

```bash
top -H -p $(pgrep -o splunkd)
```

Threads of the oldest `splunkd` process.

```bash
top -b -n 5 -d 2 > /tmp/top.txt
```

Five samples, two seconds apart, saved to a file for later.

### htop cheat sheet

`htop` shows the same data as `top` but is easier to drive: per-core bars on top, a scrolling process list, a function-key menu along the bottom, and mouse support. Install with `sudo apt install htop` if it is missing. Key names below match recent versions; press `F1` to check yours.

#### Function keys

| Key | Does |
| --- | --- |
| `F1` | Help and colour legend |
| `F2` | Setup: meters, columns, display options |
| `F3` or `/` | Search by name (jumps to next match) |
| `F4` or `\` | Filter: show only matching processes |
| `F5` or `t` | Tree view (parent and child) |
| `F6` | Choose the sort column |
| `F7` / `F8` | Lower / raise nice value (`F7` raises priority and needs root) |
| `F9` | Kill: pick a signal, 15 first, 9 last resort |
| `F10` or `q` | Quit |

#### Other keys worth knowing

| Key | Does |
| --- | --- |
| `P` / `M` / `T` | Sort by CPU / memory / time |
| `I` | Invert the sort order |
| `u` | Show only one user's processes |
| `Space` | Tag a process; act on all tagged ones (e.g. kill) |
| `c` | Tag a process and its children |
| `U` | Untag everything |
| `+` / `-` | Expand / collapse a branch in tree view |
| `H` / `K` | Hide / show user threads and kernel threads |
| `s` | Trace system calls with `strace` (needs `strace`, run as root) |
| `l` | List open files with `lsof` (needs `lsof`) |
| `e` | Show a process's environment variables |
| `a` | Pin a process to chosen CPU cores |
| `F` | Follow the selected process as the list re-sorts |

#### Reading the display

- **CPU bars (top left):** one per core. Colours by default: blue low-priority, green normal user, red kernel. `F1` lists the rest, including I/O wait and steal in detailed mode.
- **Mem and Swp bars:** green is used memory, blue buffers, yellow or orange cache. A full bar is normal if most of it is cache; look at the figures, not the bar length.
- **Tasks line:** processes, threads, and how many are running.
- **Load average and Uptime:** same as the `uptime` command.
- **State column (`S`):** `R` running, `S` sleeping, `D` waiting on disk, `Z` zombie. Lots of `D` suggests an I/O problem.

#### Setup worth doing once (F2)

1. **Display options:** turn on detailed CPU time so `iowait` and `steal` show in the bars.
2. **Meters:** add Disk IO and Network IO to the header if your version offers them.
3. **Columns:** add the I/O rate columns to see which process is hammering the disk.
4. Settings save automatically to `~/.config/htop/htoprc`.

#### Command-line options

```bash
htop -u splunk
```

Start showing only the `splunk` user's processes.

```bash
htop -p 1234,5678
```

Watch specific PIDs only.

```bash
htop -t
```

Start in tree view.

```bash
htop -d 10
```

Refresh every 1 second (the value is in tenths of a second).

```bash
htop -s PERCENT_MEM
```

Start sorted by memory (`htop --sort-key help` lists the column names).

### pidstat

Per-process statistics *over an interval* — like a per-PID `mpstat`. Splits user vs system CPU time and can track I/O per process, which `top` doesn't do well.

```bash
pidstat [-d] [-p <PID>] <INTERVAL>
pidstat 1
pidstat -d 1
```

- **\<INTERVAL\>** — seconds between reports (this is what makes it a rate, not a snapshot).
- **-p \<PID\>** — restrict to one process (`-p ALL` for everything).
- **-d** — report per-process disk I/O (read/write KB/s) instead of CPU.
- Watch: the `%system` column — a process heavy in system time is spending it in the kernel (syscalls, I/O), a clue to drill with tracing.

**In the field — which thread of splunkd is burning CPU?**

```bash
pidstat -t -p $(pgrep -o splunkd) 1
```

### pmap

Lists a process's memory mappings with per-segment usage — where its RSS actually lives (heap, stack, each mapped library or file).

```bash
pmap -x <PID>
pmap -x 18733
```

- **\<PID\>** — the target process.
- **-x** — extended columns, including RSS and dirty pages per mapping. Use `-X` for even more detail.
- Watch: an unexpectedly large anonymous mapping (heap growth / a leak) or a surprising mapped file.

**Reading the output** — excerpt from `pmap -x` on a bash shell process `3542`:

```text
3542:   bash
Address           Kbytes     RSS   Dirty Mode  Mapping
00005d90fbbdb000     192     192       0 r---- bash             <- headers / read-only data
00005d90fbc0b000     956     956       0 r-x-- bash             <- code
00005d90fbd33000      36      36      36 rw--- bash             <- writable globals
00005d9100839000    1712    1668    1668 rw---   [ anon ]       <- heap (malloc) — grows if leaking
0000729a16e00000    5588     532       0 r---- locale-archive   <- big mapping, little resident (lazy paging)
0000729a17428000    1572    1440       0 r-x-- libc.so.6        <- shared library code
0000729a17747000      28      28       0 r--s- gconv-modules.cache  <- s = shared mapping
00007ffe22f2b000     132     112     112 rw---   [ stack ]
---------------- ------- ------- -------
total kB           11536    6020    1964
```

| Col | Meaning |
| --- | --- |
| `Kbytes` | Mapping size — virtual, reserved not necessarily used (sums to ps `VSZ`) |
| `RSS` | Portion actually in RAM (sums to ps `RSS`) |
| `Dirty` | Pages the process modified — can't be re-read from disk; ≈ its unique cost |
| `Mode` | `r`/`w`/`x`; 4th char `s` = shared, `-` = private (copy-on-write) |
| `Mapping` | Backing file, `[ anon ]` (heap, `.bss`, TLS), or `[ stack ]` |

- Each binary/library appears as 4–5 segments: read-only headers, `r-x` code, relocated read-only data, `rw` globals.
- Small `[ anon ]` regions near `ld-linux` are the kernel's vvar/vdso pages; `ffffffffff600000` is the legacy vsyscall page. `pmap -X` names them.
- **RSS overcounts:** clean library pages (libc code) are shared by every process but counted in each one's RSS. `Dirty` ≈ what killing the process frees; for a fair split use **PSS** (`pmap -X`, `/proc/<PID>/smaps_rollup`, `smem`).
- Verdict on the sample: a tiny shell — ~6 MB visible, ~2 MB its own, mostly heap.

**In the field — is a process's memory growing? Sample its total every 5s:**

```bash
watch -n5 'pmap -x <PID> | tail -1'
```

Leak = `RSS`/`Dirty` climbing without plateauing, concentrated in `[ anon ]`.

**In the field — biggest mappings, and the real cost in one look:**

```bash
pmap -x <PID> | sort -k3 -n | tail -5
grep -E '^(Rss|Pss|Private_Dirty)' /proc/<PID>/smaps_rollup
```

## Profilers

*Data: event-based (sampling) · Role: observability · Cost: predictable.* Profilers take periodic samples — typically the instruction pointer or stack at a fixed rate — to paint a coarse picture of where time goes. Overhead is controllable because you choose the rate; the classic choice is 99 Hz, offset from round numbers to avoid sampling in lockstep with periodic activity.

### perf (profiling mode)

The standard Linux profiler. Samples CPU stacks system-wide or per-process and is the usual front end for turning a CPU problem into a flame graph. (perf also traces — see the tracer section.)

```bash
perf record -F <HERTZ> -a -g -- sleep <SECONDS>
perf record -F 99 -a -g -- sleep 30
perf report
```

- **-F \<HERTZ\>** — sample frequency per second (99 is the convention).
- **-a** — all CPUs, system-wide (drop it and add `-p <PID>` to target one process).
- **-g** — capture call-graph (stack traces), not just the leaf — essential for flame graphs.
- **-- sleep \<SECONDS\>** — the command perf wraps to bound the capture; `sleep 30` just means “profile for 30 s.”
- `perf report` reads the resulting `perf.data`; pipe stacks through a flame-graph generator for the visual.

**Permissions** — as non-root on Ubuntu you'll get *Failure to open any events*: `kernel.perf_event_paranoid` defaults to **4** (Ubuntu/Debian-only level = no perf for non-root). What non-root can do by level:

| Level | Non-root can profile |
| --- | --- |
| 4 | Nothing (Ubuntu default) |
| 2 | Own processes, user space only (upstream default) |
| 1 | Own processes, user + kernel |
| 0 | System-wide (`-a`) |
| -1 | Almost everything |

- **Simplest:** `sudo perf record ...`
- **Lab/VM:** `sudo sysctl kernel.perf_event_paranoid=1` (or `-1`), plus `kernel.kptr_restrict=0` for kernel symbol names. Persist in `/etc/sysctl.d/`.
- **Production team:** grant `cap_perfmon,cap_sys_ptrace,cap_syslog` to the real perf binary (under `/usr/lib/linux-tools/`, not the `/usr/bin/perf` wrapper) for a `perf_users` group — no root, setting stays locked.
- *Missing support for build id in kernel mmap events* is a harmless warning (`--no-buildid-mmap` hides it).
- **VMs:** hardware events (`cpu/cycles`) need PMCs passed through (VMware: *Virtualize CPU performance counters*). Without them use `-e cpu-clock` — fine for flame graphs, no cache/IPC data. Check with `perf list hw` (empty = none). On a Windows host with Hyper-V active (WSL2, Docker Desktop, Core Isolation), VMware can't pass PMCs through at all.

**In the field — is a live process stalling on memory?** (needs PMCs; else *The cycles event is not supported*)

```bash
sudo perf stat -e cycles,instructions,cache-misses -p <PID> -- sleep 10
```

Read `insn per cycle` (IPC): > ~1 = doing useful work; < ~1 with high `cache-misses` = stalling on memory. Without `-- sleep N` it counts until Ctrl-C. No PMCs? Software events still work: `-e task-clock,context-switches,cpu-migrations,page-faults`.

**Reading the record output:**

```text
[ perf record: Woken up 3 times to write data ]                  <- buffer flushed to disk 3x, routine
[ perf record: Captured and wrote 1.831 MB perf.data (5940 samples) ]
```

5940 = 99 Hz × 2 CPUs × 30 s — `-a` samples every CPU whether busy or not. On an idle box most samples are `swapper` (the kernel idle task, PID 0) — run a load (`stress --cpu 2 --timeout 15 &`) to see something meaningful.

**Ownership gotcha:** under `sudo`, recent perf hands `perf.data` back to the invoking user, so `sudo perf report` then fails with *not owned by current user or root*. Add `-f` **after** the subcommand: `sudo perf report -f` (not `perf -f report`).

**Flame graph:**

```bash
git clone https://github.com/brendangregg/FlameGraph        # one-off; needs perl
sudo perf script -f | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > cpu.svg
xdg-open cpu.svg                                            # open in a browser — it's interactive
```

`perf script` dumps every sample's stack → `stackcollapse-perf.pl` merges identical stacks into one line + count → `flamegraph.pl` draws them. If run as separate steps (`> out.perf`, `> out.folded`), *No stack counts found* means an earlier step didn't produce its file — check each with `ls -l` / `head`.

Reading it: each box is a function, the one above it is what it called; **width = share of CPU samples** (wide = where the time goes); left-to-right is alphabetical, not time; colours are random. Click to zoom, Ctrl+F to search. Look for wide plateaus at the top — the functions actually burning CPU. Java JIT frames show as `[unknown]` without `-XX:+PreserveFramePointer` + perf-map-agent (or use JFR).

**In the field — profile just the hottest splunkd for 30s:**

```bash
perf record -F 99 -p $(pgrep -n splunkd) -g -- sleep 30
```

### profile (BCC)

A BPF-based CPU profiler that frequency-counts stacks *in the kernel* and emits folded output ready for a flame graph — lighter-weight than perf's record-to-file approach.

```bash
profile [-F <HERTZ>] [-p <PID>] [<DURATION>]
profile -F 99 30
```

- **-F \<HERTZ\>** — sample rate (default 49).
- **-p \<PID\>** — restrict to one process.
- **\<DURATION\>** — seconds to profile.
- Needs a BPF-capable kernel; part of the bcc-tools package.

**In the field — 10s folded CPU profile of one PID, ready for a flame graph:**

```bash
profile -F 99 -p <PID> -f 10 > out.folded
```

### gprof

The GNU profiler for code you compile yourself. The program must be built with profiling instrumentation, so it's a developer tool, not a production one.

```bash
gcc -pg -o <PROG> <PROG>.c  # compile with profiling
./<PROG>                    # run; writes gmon.out
gprof <PROG> gmon.out
```

- **-pg** — the compiler flag that inserts the profiling hooks.
- **gmon.out** — the data file the instrumented run drops in the working directory.
- Reports a flat profile and a call graph with time per function.

### cachegrind

Part of the Valgrind toolkit; simulates the program to profile CPU-cache and branch behaviour. Very detailed, but runs the target far slower — analysis, not production.

```bash
valgrind --tool=cachegrind ./<PROG>
kcachegrind cachegrind.out.<PID>
```

- **--tool=cachegrind** — selects the cache simulator.
- **cachegrind.out.\<PID\>** — the output file, visualised with `kcachegrind`.
- **\<PROG\>** — any program + args (`./` = in the current dir, usually your own fresh build). Use `ls` or `/bin/bash`, not `./bash`.
- **--cache-sim=yes** — needed for miss stats; since Valgrind 3.21 only instruction counts (`I refs`) are collected by default. Text report: `cg_annotate cachegrind.out.<PID>`.
- **No summary printed?** The target `exec`'d another program (e.g. `bash -c 'cmd'` execs `cmd`) and valgrind doesn't follow by default — profile the program directly or add `--trace-children=yes`.

**Real-world uses** — a developer tool, rarely an ops one: tuning hot code to be cache-friendly (line-level misses), deterministic regression benchmarks in CI (simulated, so identical counts every run — SQLite measures performance this way), comparing two versions of code (`cg_diff`), and getting cache data where hardware counters are absent (VMs). Can't attach to a running PID and runs 20–100× slower, so never on production — for a live process use `perf stat` (see perf section). Rule of thumb: own the source → cachegrind; diagnosing something running → perf.

### Java Flight Recorder (JFR)

A language-level profiler built into the JVM that sees Java context (methods, allocations, GC) that a system profiler can't. The pattern recurs: managed runtimes often need their own profiler.

```bash
java -XX:+FlightRecorder -XX:StartFlightRecording=duration=<SECONDS>s,filename=<FILE>.jfr -jar <APP>.jar
```

- **duration=\<SECONDS\>s** — how long to record (note the trailing `s` for seconds).
- **filename=\<FILE\>.jfr** — where the recording lands; open it in JDK Mission Control.
- Directly relevant anywhere you run JVM workloads.

**In the field — attach to an already-running JVM without restarting it:**

```bash
jcmd <PID> JFR.start duration=60s filename=/tmp/rec.jfr
```

## System-wide tracers

*Scope: system-wide · Data: event-based (tracing) · Role: observability · Cost: scales with event rate.* Tracers capture *every* event of interest. They see what counters and sampling miss, but overhead grows with the event rate — under ~10k events/sec is negligible, scheduler-rate events (100k+/sec) can hurt. Each tool's instrumentation type is tagged below.

### tcpdump

Captures network packets off an interface for detailed protocol analysis. Uses libpcap; carries CPU and storage cost, so filter tightly.

```bash
tcpdump -i <IFACE> -w <FILE>.pcap <EXPRESSION>
tcpdump -i eth0 -w capture.pcap 'tcp port 8089'
```

- **-i \<IFACE\>** — interface to capture on (`eth0`, or `any`).
- **-w \<FILE\>.pcap** — write raw packets to a file (analyse later in Wireshark); omit to print live.
- **\<EXPRESSION\>** — a BPF filter limiting what's captured (`tcp port 8089` for Splunk management traffic, say). Filtering at capture is what keeps overhead sane.

**In the field — capture 100 forwarder→indexer S2S packets then stop:**

```bash
tcpdump -i eth0 -c 100 'tcp port 9997'
```

### execsnoop *(static or dynamic, via BCC/bpftrace)*

Prints every new process as it's created, by instrumenting the exec path. Catches short-lived processes that snapshot tools like top miss entirely.

```bash
execsnoop [-t] [-n <NAME>]
execsnoop -t
```

- **-t** — include a timestamp column.
- **-n \<NAME\>** — only show processes whose name matches \<NAME\>.
- Great for spotting a cron job or script storm quietly stealing CPU.

### biosnoop *(dynamic, via BCC/bpftrace)*

Traces each block-device I/O with the issuing process, size, and latency — the per-I/O disk visibility that was historically hard to get.

```bash
biosnoop
```

- No required parameters; prints a live line per I/O (timestamp, process, device, sector, bytes, latency ms).
- Watch the latency column for outliers a `%util` average would hide.

**In the field — show only block I/O slower than 10ms (the latency column is last):**

```bash
biosnoop | awk 'NR==1 || $NF > 10'
```

### perf trace

A perf subcommand that traces system calls (and other events) system-wide, a lower-overhead `strace` alternative.

```bash
perf trace -e <EVENT>
perf trace -e block:block_rq_issue
```

- **-e \<EVENT\>** — the event(s) to trace: a syscall, or a tracepoint like `block:block_rq_issue`. Omit to trace all syscalls.
- Supports `--filter '<EXPR>'` to match on event arguments (e.g. `bytes > 65536`).

**In the field — which syscalls dominate splunkd over 10s? (summary, not every line)**

```bash
perf trace -s -p $(pgrep -o splunkd) -- sleep 10
```

### Ftrace *(static + dynamic)*

The kernel's built-in tracer — a multi-tool for function tracing and tracepoints, with no extra dependencies, ideal on locked-down or minimal systems. Usually driven via front-end scripts (the perf-tools collection) rather than raw tracefs.

```bash
# raw interface lives under:
/sys/kernel/debug/tracing/
# e.g. enable a tracepoint:
echo 1 > events/block/block_rq_issue/enable
```

- Driven by writing to files under **tracefs** (`/sys/kernel/debug/tracing`). Reach for the `perf-tools` wrappers (`funccount`, `iolatency`, `iosnoop`) rather than hand-editing these in anger.

**In the field — count TCP send-path kernel calls over 10s (perf-tools wrapper):**

```bash
funccount -d 10 'tcp_send*'
```

### BCC *(static + dynamic)*

A BPF tracing toolkit — a library plus 100+ polished, production-grade tools (execsnoop, biosnoop, runqlat, ext4slower, cachestat…). Reach for BCC when you want a ready-made, battle-tested answer.

```bash
/usr/share/bcc/tools/<TOOLNAME> [<ARGS>]
/usr/share/bcc/tools/runqlat 10 1
```

- **\<TOOLNAME\>** — the specific tool; each has its own flags and its own man page.
- Needs a BPF-capable kernel and the bcc-tools package.

**In the field — catch ext4 operations slower than 10ms:**

```bash
ext4slower 10
```

### bpftrace *(static + dynamic)*

A high-level language for custom BPF one-liners and short scripts — when no existing tool answers your exact question. Concise enough to write at the keyboard mid-incident.

```bash
bpftrace -e '<PROBE> { <ACTION> }'
bpftrace -e 'kprobe:do_nanosleep { printf("sleep by: %s\n", comm); }'
bpftrace -l '<PATTERN>'
```

- **-e '\<PROGRAM\>'** — the inline program: a \<PROBE\> (what to instrument — `kprobe:`, `tracepoint:`, `uprobe:`, `usdt:`) and an \<ACTION\> in braces.
- **-l '\<PATTERN\>'** — list probes matching a glob (e.g. `-l 'tracepoint:block:*'`) — how you discover what's instrumentable.
- `comm`, `pid`, `arg0..argN` are built-ins available inside the action.

**In the field — count syscalls by process to see who's hammering the kernel:**

```bash
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
```

## Per-process tracers

*Scope: per-process · Data: event-based (tracing) · Role: observability · Cost: high.* These stop and start the target to inspect it, so overhead is severe — fine on a test box, risky in production. Where a system-wide tracer with a PID filter (`perf`, `bpftrace -p`) can do the job, prefer it.

### strace

Traces the system calls a process makes, with arguments and return values — superb for seeing exactly what a program asks the kernel to do, and why a call fails. But it's breakpoint-based (ptrace) and can slow the target by 100× or more.

```bash
strace -f -e trace=<SYSCALLS> -p <PID>
strace -f -e trace=openat,read -p 18733
strace -c -p <PID>
```

- **-p \<PID\>** — attach to a running process (or put a command at the end to launch and trace it).
- **-f** — follow child processes/threads (forks), not just the parent.
- **-e trace=\<SYSCALLS\>** — limit to named syscalls (comma-separated), cutting both noise and overhead.
- **-c** — don't print each call; instead summarise counts and time per syscall at the end (a cheap way to see where syscall time goes).
- Reach for it to diagnose a specific failure (which `open` returns `ENOENT`?), not to run against a hot production process.

**In the field — why can't splunkd open a file? Catch the failing opens:**

```bash
strace -f -e trace=openat -p <PID> 2>&1 | grep ENOENT
```

### gdb

A source-level debugger. In a performance context it's an occasional tool — grabbing a stack from a stuck process, or reading a variable no stat exposes — rather than a tracer you'd leave running.

```bash
gdb -p <PID>
(gdb) thread apply all bt
(gdb) detach
```

- **-p \<PID\>** — attach to a running process (it stops while attached — be careful in production).
- **thread apply all bt** — print a backtrace for every thread (the “where is it stuck?” command).
- **detach** — let the process resume. Forgetting this leaves it frozen.
- Also scriptable with `-batch -ex '<CMD>'` for a single non-interactive grab.

**In the field — grab all thread stacks from a hung process, non-interactively:**

```bash
gdb -p <PID> -batch -ex 'thread apply all bt' 2>/dev/null
```

## Monitoring tools

*Scope: system-wide · Data: counters · Role: observability + history.* Monitoring records statistics *continuously* so the data exists when you need it later. This is what turns “80% busy” into “busy, and climbing all week.”

### sar

The System Activity Reporter: counter-based, collects on a cron schedule, and keeps a rolling archive you can replay. An essential tool that solves many issues on its own.

```bash
sar -<OPTION> [-n <GROUP>]
sar -u -n TCP,ETCP           # read today's archive
sar -n TCP <INTERVAL> <COUNT>    # live mode
sar -n TCP 1 5
```

- **-\<OPTION\>** — the statistic group: `-u` CPU, `-r` memory, `-d` disks, `-b` I/O, `-q` run-queue.
- **-n \<GROUP\>** — network sub-group: `DEV` (interfaces), `EDEV` (errors), `TCP`, `ETCP`.
- **\<INTERVAL\> \<COUNT\>** — given these, sar reports live instead of reading the archive (works even if collection is off).
- **-A** — dump everything.
- Enable collection first: set `ENABLED="true"` in `/etc/default/sysstat`, restart the service, and consider `-S ALL` with a tighter cron cadence. History you didn't collect is gone.

**In the field — replay a past day's CPU to find when it spiked (\<DD\> = day of month):**

```bash
sar -u -f /var/log/sysstat/sa<DD>
```

### sadf

The companion that reformats sar's archive into portable formats for tooling and dashboards.

```bash
sadf -<FORMAT> -- <SAR_OPTIONS>
sadf -j -- -n TCP      # JSON
sadf -d -- -n TCP      # CSV
```

- **-\<FORMAT\>** — output format: `-j` JSON (parse with `jq`), `-d` CSV (for databases), `-g` SVG.
- **-- \<SAR_OPTIONS\>** — everything after `--` is passed through to sar to select the stats.

**In the field — export a day's CPU history to CSV for a spreadsheet:**

```bash
sadf -d /var/log/sysstat/sa<DD> -- -u > cpu.csv
```

### SNMP

The traditional network-monitoring protocol. Many devices and OSes expose basic metrics over it by default, so no agent install is needed — but it hasn't kept pace with modern application metrics, and most shops have moved to agents.

- Not a command you run so much as a data source polled by a monitoring server. Useful mainly for network gear and appliances that expose nothing else.

### Agents: PCP, Prometheus, collectd

*Scope: system-wide + application · Role: monitoring.* Modern monitoring runs an agent (exporter/plugin) on each host that reads kernel and application metrics and ships them to a central store with a dashboard.

- **Performance Co-Pilot (PCP)** — dozens of PMDAs (metric agents), including BPF-based ones; strong for whole-system archived analysis.
- **Prometheus** — pull-based; dozens of exporters (node\_exporter for host metrics, plus databases, HTTP, storage). The common choice with Grafana for dashboards.
- **collectd** — a lightweight daemon with many plugins; often feeds a time-series database.
- The key insight: the system statistics these expose are the *same* kernel counters the CLI tools read — learning the tools makes you fluent in the dashboards, even if you never type the commands.

## Static configuration-at-rest tools

*Role: static (not observability) · Data: config, not activity.* These examine the system *at rest* — what hardware exists, how it's configured, what limits are set — rather than what it's doing under load. Run them early: a surprising number of “performance problems” are really a misconfiguration, and these are free to rule out. Most need no workload and take few or no parameters.

| Tool | What it checks | Typical command |
| --- | --- | --- |
| `lscpu` | CPU model, core/thread count, cache sizes, NUMA layout | `lscpu` |
| `lsblk` | Block devices and their mount tree | `lsblk` |
| `lspci` | PCI devices (NICs, controllers, accelerators) | `lspci` |
| `lsscsi` | SCSI/SATA devices | `lsscsi` |
| `smartctl` | Disk health / SMART attributes | `smartctl -a /dev/<DEVICE>` |
| `blockdev` | Block device sizes and read-ahead | `blockdev --report` |
| `swapon` | Swap devices in use | `swapon --show` |
| `df` | File-system capacity | `df -h` |
| `ethtool` | NIC settings: link speed, duplex, offloads, ring sizes | `ethtool <IFACE>` |
| `numactl` | NUMA policy and node layout | `numactl -H` |
| `lstopo` | Full hardware topology (cores, caches, NUMA, devices) | `lstopo` |
| `dmidecode` | BIOS/DMI: board, memory slots, firmware | `dmidecode -t <TYPE>` |
| `lshw` | Full hardware inventory | `lshw -short` |
| `ldd` | Shared libraries a binary links against | `ldd /path/to/<BINARY>` |
| `sysctl` | Kernel tunables (read and set) | `sysctl -a` |
| `mdadm` | Software-RAID array state | `mdadm --detail /dev/md<N>` |
| `turbostat` | Live CPU frequency, idle states, power (needs MSRs) | `turbostat` |

A few parameter notes:

- **`smartctl -a /dev/<DEVICE>`** — `-a` prints all SMART data; \<DEVICE\> is e.g. `/dev/sda` or `/dev/nvme0`. Look for reallocated sectors and error logs.
- **`ethtool <IFACE>`** — \<IFACE\> is the interface (`eth0`). Add `-S <IFACE>` for driver stats (drops, errors), `-g <IFACE>` for ring-buffer sizes.
- **`dmidecode -t <TYPE>`** — \<TYPE\> narrows the dump: `memory`, `bios`, `processor`, `system`.
- **`sysctl -w <KEY>=<VALUE>`** — sets a tunable live; `sysctl <KEY>` just reads it. Persist real changes in `/etc/sysctl.d/`, and version-control them — point-in-time tunables that nobody understands later are a trap.
- **`turbostat`** needs MSR access, so it may not work in a cloud guest — see the specialist-utilities section.

**In the field — the config checks worth running during an incident:**

```bash
smartctl -l error /dev/sda               # read a suspect disk's error log
ethtool -S eth0 | grep -iE 'drop|err'    # NIC drops and errors
sysctl net.core.somaxconn vm.swappiness  # check specific tunables
```

## Specialist & observability-source utilities

*Role: observability · mixed scope.* A handful of tools that tap specific kernel sources — netlink, delay accounting, per-NIC counters, PMCs/MSRs — worth knowing when the general tools don't reach.

### ss

Socket statistics via the netlink binary interface — faster and richer than the legacy netstat. The go-to for connection state and per-socket TCP detail.

```bash
ss -tiepn
ss -ti state established
```

- **-t** — TCP sockets only.
- **-i** — internal TCP info (rtt, cwnd, retransmits) — the performance-relevant part.
- **-e** — extended socket details; **-p** — owning process; **-n** — numeric (don't resolve ports).
- Watch per-socket `retrans` and `rtt` when chasing network latency between, say, a forwarder and an indexer.

**In the field — count sockets stuck in TIME-WAIT (connection churn):**

```bash
ss -tan state time-wait | wc -l
```

### nicstat

Per-network-interface throughput and utilisation — the network equivalent of iostat, filling a gap the default tools leave.

```bash
nicstat <INTERVAL> <COUNT>
nicstat 1 5
```

- **\<INTERVAL\> \<COUNT\>** — report every \<INTERVAL\> seconds, \<COUNT\> times.
- Watch `%Util` against the link's known max bandwidth — the cleanest read on an interface approaching saturation.

**In the field — watch one interface against its link speed during a throughput complaint:**

```bash
nicstat -i eth0 1
```

### tiptop

A top-like live view powered by PMCs — shows per-process IPC (instructions per cycle) and cache-miss rates, i.e. CPU *efficiency*, not just busyness.

```bash
tiptop
```

- No required parameters. Needs PMC access — so often unavailable in cloud guests (see below).
- Watch IPC: a process burning CPU at low IPC is stalling (often on memory/cache), not doing useful work.

**In the field — log CPU efficiency non-interactively every 5s:**

```bash
tiptop -b -d 5
```

### getdelays

Reads per-task delay accounting — time spent *waiting* on the CPU run-queue, block I/O, swapping, or memory reclaim. Directly answers “is it slow because it's working, or because it's waiting?”

```bash
getdelays -d -p <PID>
```

- **-d** — print delay-accounting statistics.
- **-p \<PID\>** — the target task.
- Reports count and total/average delay per category; high CPU delay = scheduler contention, high IO delay = blocked on disk.

**In the field — is the busy splunkd waiting on the run-queue or on disk?**

```bash
getdelays -d -p $(pgrep -o splunkd)
```

### rdmsr / wrmsr (msr-tools)

Read and write model-specific registers — the low-level source behind PMCs, plus clock, temperature, and power data. Niche, but the accurate source for some CPU metrics.

```bash
rdmsr [-p <CPU>] <REGISTER>
rdmsr -p 0 0x10
```

- **-p \<CPU\>** — which CPU's register to read.
- **\<REGISTER\>** — the MSR address in hex.
- `wrmsr` writes — powerful and dangerous; mostly you'll only read. Needs the `msr` kernel module and privilege, and typically fails in cloud guests where MSR access is disabled.

**Cloud caveat for this whole section:** tiptop, rdmsr, and turbostat all depend on PMC/MSR access that many cloud hypervisors withhold from guests. Check per instance type — some AWS bare-metal and larger Nitro sizes expose them, most smaller guests don't.

## The tracing frameworks compared

Four of the system-wide tracers above are really *frameworks* — big enough that choosing between them is its own decision. Two more round out the landscape. All draw on the same event sources (tracepoints, kprobes, uprobes, USDT, PMCs); they differ in ergonomics and what they're best at.

| Framework | Best for | Reach for it when |
| --- | --- | --- |
| `perf` | CPU profiling and PMC analysis | You need a flame graph or hardware-counter data |
| `Ftrace` | Kernel code-path tracing, no dependencies | The box is locked down/minimal and you're digging in the kernel |
| `BCC` | Ready-made, production-grade BPF tools | You want a proven answer fast (biosnoop, ext4slower, runqlat…) |
| `bpftrace` | Custom one-liners and short scripts | No existing tool fits your exact question |
| `SystemTap` | Scripted tracing with tapset libraries | You're in an environment built around it (now with a BPF backend) |
| `LTTng` | High-volume black-box recording | You need to capture many events for later offline analysis |

**A sane default split** (the author's own): `perf` for CPU analysis, `Ftrace` for kernel code digging, and `BCC`/`bpftrace` for everything else — memory, file systems, disks, networking, and application tracing.

**How they relate to the earlier tools:** many of the individual tracers in this guide (execsnoop, biosnoop, runqlat) ship as part of BCC *and* as bpftrace one-liners *and* as Ftrace-based scripts in the perf-tools collection. Same idea, different engine — so having one framework working on every box matters more than which one.

---

That's every tool the chapter names, tagged and drilled. Pair this with the incident worksheet (drive the investigation) and the troubleshooting guide (the ordered method and per-resource checklists) and you have the full kit: the method, the reference, and the record.
