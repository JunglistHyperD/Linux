# Linux Performance & Splunk-on-EC2 Cheat Sheets

## 1. The 60-second checklist

Run these ten in order when a box feels slow; each takes a few seconds and together they show which resource is stressed. Ctrl+C stops anything that repeats. If a command is missing, install the toolset: `sudo apt install sysstat` covers `mpstat`, `pidstat`, `iostat` and `sar`.

### 1. uptime: is load rising or falling?

Shows load averages for the last 1, 5 and 15 minutes. Compare them to your core count (`nproc`). 1-min above 15-min means load is rising.

```
uptime
```

### 2. dmesg: any kernel errors?

The kernel's own log. Look for OOM kills ("Out of memory: Killed process"), disk I/O errors and filesystems remounted read-only. `-T` gives readable timestamps.

```
dmesg -T | tail -n 50
```

Might need `sudo` ?

### 3. vmstat: system-wide view

One line per second, 20 samples. Ignore the first line (averages since boot). `r` above core count means CPU queueing, `b` above 0 means tasks blocked on I/O, non-zero `si`/`so` means swapping, high `wa` means disk wait, high `st` means the hypervisor is taking your CPU.

```
vmstat -SM 1 20
```

### 4. mpstat: per-core balance

Shows each core separately. One core near 100% with the rest idle points to a single-threaded bottleneck. Also check `%iowait`, `%steal` and `%sys`.

```
mpstat -P ALL 1
```

### 5. pidstat: which process uses the CPU?

Per-process CPU, split into `%usr` (own code) and `%system` (kernel work on its behalf). Rolling output, so you see who is busy right now. Unlike `top`, it scrolls and can be pasted into a ticket.

```
pidstat 1
```

### 6. iostat: disk load and latency

`-s` short format, `-x` extended stats, `-z` hides idle devices. Watch `await` (ms per request: sustained high values mean a slow or capped disk) and `%util`. On NVMe and EBS, `%util` near 100 can overstate saturation because they serve requests in parallel, so trust `await` more.

```
iostat -sxz 1
```

### 7. free: memory and cache

Read the `available` column, not `free`. Linux fills spare RAM with file cache and hands it back on demand. Low `available` plus swap use means real memory pressure.

```
free -m
```

### 8. sar -n DEV: network throughput

Packets and kB/s per interface. Ignore `lo`. Compare `rxkB/s` and `txkB/s` against what the instance type allows. On EC2, limits are enforced outside the OS, so a capped link can still look quiet here.

```
sar -n DEV 1
```

### 9. sar -n TCP,ETCP: connections and retransmits

`active/s` and `passive/s` are outgoing and incoming connection rates. `retrans/s` is the key one: sustained retransmits relative to `oseg/s` mean packet loss or a congested path.

```
sar -n TCP,ETCP 1
```

## 4. Splunk on EC2 Ubuntu: on-the-box troubleshooting

Start with the 60-second checklist to find the stressed resource, then use the symptom sections below to find the Splunk-side cause. Paths assume the defaults: `SPLUNK_HOME=/opt/splunk`, service user `splunk`, systemd unit `Splunkd`. Adjust if yours differ. Splunk CLI commands may prompt for credentials or need `-auth user:pass`.

### Step 0: is Splunk up, and what is it saying?

```
sudo systemctl status Splunkd
sudo -u splunk /opt/splunk/bin/splunk status
sudo tail -n 100 /opt/splunk/var/log/splunk/splunkd.log
```

Service state, Splunk's own view of its processes, then the last 100 log lines. The next two commands pull out only the problems:

```
sudo grep -E "ERROR|WARN" /opt/splunk/var/log/splunk/splunkd.log | tail -n 50
sudo journalctl -u Splunkd -n 50 --no-pager
```

### Splunk won't start

Check ports, disk, ownership and config validity, in that order.

```
sudo ss -tlnp | grep -E "8089|8000|9997|8088"
df -h /opt/splunk && df -i /opt/splunk
sudo find /opt/splunk ! -user splunk | head
sudo -u splunk /opt/splunk/bin/splunk btool check
```

- A listener already on 8089, 8000, 9997 or 8088 means a port clash or a stuck old process.
- Full disk or no free inodes stops startup.
- Files owned by root appear when Splunk was once started as root. Fix with `sudo chown -R splunk:splunk /opt/splunk`.
- `btool check` reports syntax errors in `.conf` files.

### Disk filling, indexing paused

Splunk pauses indexing when free space falls below `minFreeSpace` (default 5000 MB, in `server.conf`).

```
df -h
sudo du -sh /opt/splunk/var/lib/splunk/* | sort -h | tail
sudo du -sh /opt/splunk/var/run/splunk/dispatch
```

Big index directories point to retention or volume sizing. A large `dispatch` directory means search artefacts piling up. Also check the instance's EBS volume size against growth.

### Indexing lag or blocked queues

On the box, look at queue state and disk latency together.

```
sudo grep "blocked=true" /opt/splunk/var/log/splunk/metrics.log | tail -n 20
iostat -sxz 1
```

Blocked parsing, aggregation or indexing queues plus high `await` in `iostat` means the disk is the limit. On EC2 that is usually the EBS volume's IOPS or throughput cap, which the OS cannot see; confirm in CloudWatch.

### Slow searches

```
mpstat -P ALL 1
ps -eo pid,user,rss,pcpu,etime,args --sort=-rss | head -n 15
```

A few cores pegged while others idle is normal for heavy concurrent searches, since each search mostly runs on one core. Long-running, high-RSS `splunkd` search processes show the offenders. Scheduled searches all firing on the hour is the classic cause.

### Data not arriving from forwarders

On the receiver:

```
sudo ss -tlnp | grep 9997
sudo grep -i "blocked" /opt/splunk/var/log/splunk/splunkd.log | tail
```

On the forwarder:

```
/opt/splunkforwarder/bin/splunk list forward-server
nc -vz <indexer-ip> 9997
```

No listener means receiving isn't enabled. A failed `nc` means a security group, NACL or routing problem. Add `sar -n TCP,ETCP 1` on either side to check retransmits.

### HEC problems

```
curl -k https://localhost:8088/services/collector/health
sudo ss -tlnp | grep 8088
```

A healthy endpoint returns `HEC is healthy`. Use `http://` if SSL is off for HEC. No listener means HEC is disabled or Splunk isn't running. Rejected events usually mean a bad token or disabled index; the reason is in `splunkd.log`.

### Splunk killed or restarting unexpectedly

```
sudo dmesg -T | grep -i -E "out of memory|oom|killed process"
free -m
```

A kill line naming `splunkd` means the OOM killer. Cause is usually memory-hungry searches or an undersized instance type. `Killed process` with no Splunk line points elsewhere.

### Limits and kernel settings Splunk complains about

Splunk logs warnings at startup about these; check them directly:

```
cat /proc/$(pgrep -o splunkd)/limits | grep -i -E "open files|processes"
cat /sys/kernel/mm/transparent_hugepage/enabled
```

Splunk recommends an open-files limit of 64000 or more, and transparent huge pages set to `never`. Limits for systemd services are set in the unit file (`LimitNOFILE`), not in `ulimit` shell settings.

### Clock skew

Bad timestamps and cluster flapping often trace back to time drift.

```
timedatectl
chronyc tracking
```

The clock should show as synchronised, with a small offset.

### Clustering checks

```
/opt/splunk/bin/splunk show cluster-status
/opt/splunk/bin/splunk show shcluster-status
```

The first runs on the cluster manager, the second on a search head cluster member. After a peer restarts, expect a burst of disk and network activity while buckets replicate and fix up; that is normal, so compare with your baseline before chasing it.

### Limits that live outside the OS (EC2)

If the box looks healthy but Splunk is slow, AWS may be throttling it.

| Symptom on the box | Check in AWS |
| --- | --- |
| High `await`, disk cannot keep up | CloudWatch EBS volume queue length, throughput and IOPS against the volume's provisioned limits; `BurstBalance` on gp2 |
| High `%steal`, nothing busy in `top` | `CPUCreditBalance` on burstable instances (t3, t3a) |
| Network retransmits or drops | ENA allowance counters on the instance (command below) |

```
ethtool -S ens5 | grep -i allowance
```

Non-zero and rising `*_allowance_exceeded` counters mean AWS is delaying or dropping packets. Your interface name may not be `ens5`; check with `ip a`.

## 5. The 60-second checklist on Windows 10

Windows has no single `uptime` or `vmstat`, but the built-in performance counters (`Get-Counter`) cover the same ground. Run these in PowerShell as Administrator. `-SampleInterval 1 -MaxSamples N` takes one reading a second, N times, and Ctrl+C stops anything running. Counter names below are the English ones.

### 1. Uptime and CPU queue (Linux: uptime)

There is no load average. Boot time shows how long the machine has been up, and the processor queue length is the closest equivalent: tasks waiting for a CPU. Sustained above 2 per core means CPU contention.

```
(Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
Get-Counter '\System\Processor Queue Length' -SampleInterval 1 -MaxSamples 5
```

Check core count with `$env:NUMBER_OF_PROCESSORS`.

### 2. Event Log errors (Linux: dmesg)

The System log is the kernel-and-drivers log. This lists critical, error and warning events from the last hour. Look for disk and NTFS errors, unexpected shutdowns (Kernel-Power 41), and driver failures.

```
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2,3; StartTime=(Get-Date).AddHours(-1)} -MaxEvents 50 | Format-Table TimeCreated, ProviderName, Id, Message -Wrap
```

The out-of-memory equivalent is the Resource Exhaustion Detector, which logs when Windows runs low on memory and names the biggest consumers:

```
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Microsoft-Windows-Resource-Exhaustion-Detector'} -MaxEvents 10
```

### 3. System-wide statistics (Linux: vmstat)

One combined reading: CPU queue, CPU use, free memory, paging and disk queue. High `Pages/sec` with low `Available MBytes` means the machine is paging heavily. A disk queue persistently above 2 means the disk is the bottleneck.

```
Get-Counter '\System\Processor Queue Length','\Processor(_Total)\% Processor Time','\Memory\Available MBytes','\Memory\Pages/sec','\PhysicalDisk(_Total)\Current Disk Queue Length' -SampleInterval 1 -MaxSamples 20
```

### 4. Per-CPU balance (Linux: mpstat)

Shows each logical processor separately. One core pinned near 100% with the rest idle points to a single-threaded bottleneck. `_Total` is the average across all cores.

```
Get-Counter '\Processor(*)\% Processor Time' -SampleInterval 1 -MaxSamples 5 | ForEach-Object { $_.CounterSamples | Format-Table InstanceName, @{n='CPU%';e={[math]::Round($_.CookedValue,1)}} -AutoSize }
```

### 5. Per-process CPU (Linux: pidstat)

Top ten processes by current CPU, as a share of the whole machine (the counter is per core, so it is divided by the core count). `Get-Process ... CPU` is misleading here because it is total seconds used since start, not current load.

```
(Get-Counter '\Process(*)\% Processor Time').CounterSamples | Where-Object { $_.InstanceName -notin '_total','idle' } | Sort-Object CookedValue -Descending | Select-Object -First 10 InstanceName, @{n='CPU%';e={[math]::Round($_.CookedValue / $env:NUMBER_OF_PROCESSORS,1)}}
```

For the user versus kernel split, swap in `\Process(*)\% User Time` and `\Process(*)\% Privileged Time`.

### 6. Disk I/O (Linux: iostat)

Transfers per second (IOPS), queue length, and latency per read and write. Latency values are in seconds, so 0.010 is 10 ms. Sustained above roughly 20 to 25 ms is poor, and SSDs should sit far lower.

```
Get-Counter '\PhysicalDisk(*)\Disk Transfers/sec','\PhysicalDisk(*)\Avg. Disk sec/Read','\PhysicalDisk(*)\Avg. Disk sec/Write','\PhysicalDisk(*)\Current Disk Queue Length','\PhysicalDisk(*)\% Idle Time' -SampleInterval 1 -MaxSamples 10
```

### 7. Memory (Linux: free -m)

Total and free RAM in MB. Windows also keeps reclaimable cache (the standby list), so `Available MBytes` is the figure that matches Linux's `available` column. Low available memory plus heavy paging means real pressure.

```
Get-CimInstance Win32_OperatingSystem | Select-Object @{n='TotalMB';e={[int]($_.TotalVisibleMemorySize/1KB)}}, @{n='FreeMB';e={[int]($_.FreePhysicalMemory/1KB)}}
Get-Counter '\Memory\Available MBytes','\Memory\% Committed Bytes In Use' -SampleInterval 1 -MaxSamples 5
```

### 8. Network throughput (Linux: sar -n DEV)

Bytes and packets per second per adapter. Ignore the loopback and virtual adapters unless you are looking at them. `Output Queue Length` above 2 suggests the adapter can't keep up.

```
Get-Counter '\Network Interface(*)\Bytes Total/sec','\Network Interface(*)\Packets/sec','\Network Interface(*)\Output Queue Length' -SampleInterval 1 -MaxSamples 10
```

### 9. TCP statistics (Linux: sar -n TCP,ETCP)

Segments retransmitted per second against segments sent. Sustained retransmits relative to sent segments mean packet loss or congestion. Connection failures and resets show unhealthy connections.

```
Get-Counter '\TCPv4\Segments Sent/sec','\TCPv4\Segments Retransmitted/sec','\TCPv4\Connections Established','\TCPv4\Connection Failures','\TCPv4\Connections Reset' -SampleInterval 1 -MaxSamples 10
```

For cumulative totals since boot:

```
netstat -s -p tcp
```

### 10. Overview (Linux: top)

Task Manager is the equivalent. Press Ctrl+Shift+Esc, then use the Processes tab (sort by CPU, Memory, Disk, Network) and the Performance tab. For more detail, open Resource Monitor:

```
resmon
```

Its CPU, Memory, Disk and Network tabs show per-process activity, including disk files and network connections. In the terminal, a quick top-ten by memory:

```
Get-Process | Sort-Object WS -Descending | Select-Object -First 10 Name, Id, @{n='WS_MB';e={[int]($_.WS/1MB)}}
```

If you want an `htop`-style view, Sysinternals Process Explorer is the usual choice; it is a separate download from Microsoft.
