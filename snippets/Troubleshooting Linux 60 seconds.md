
- Load averages to identify if load is increasing or decreasing (compare 1-, 5-, and 15-minute averages).

```
uptime
```

-  Kernel errors including OOM events.
```
dmesg -T | tail -n 50
```

- System-wide statistics: run queue length, swapping, overall CPU usage.
```
vmstat -SM 1 20
```

- Per-CPU balance: a single busy CPU can indicate poor thread scaling.
```
mpstat -P ALL 1
```

- Per-process CPU usage: identify unexpected CPU consumers, and user/system CPU time for each process.
```
pidstat 1
```

-  Disk I/O statistics: IOPS and throughput, average wait time, percent busy.
```
iostat -sxz 1
```

- Memory usage including the file system cache.
```
free -m
```

-  Network device I/O: packets and throughput.
```
sar -n DEV 1
```

- TCP statistics: connection rates, retransmits.
```
sar -n TCP,ETCP 1
```

- Check overview.
```
top
```