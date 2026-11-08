# LPIC-1 101 – System Architecture – Study Notes

**What the course covers**
- Identifying hardware peripherals
- Using virtual filesystems + kernel modules to find / add / remove device drivers
- The boot process (BIOS/UEFI → GRUB → kernel → init)
- The 3 process managers: **SysVinit**, **Upstart**, **systemd**
- Troubleshooting boot, changing system state (runlevels), process management

---

## 1. Linux virtual filesystems and hardware peripherals

### Key ideas
- On boot (or when hardware changes) the kernel writes **virtual / pseudo files** describing devices + drivers.
- They're plain text files held in **RAM, not on disk** → lost at shutdown, rebuilt on next boot.
- Three pseudo filesystems to know:

| Dir | What it is | Notes |
|---|---|---|
| `/proc` | "process" – part of the **sysctl** system | Older design. CPU, memory, modules, devices |
| `/sys` | **sysfs** (fs = filesystem) | Newer, better structured – lots of files are **symlinks** to the devices |
| `/dev` | Device files, created at boot | One file per hardware device – used for mounting/controlling devices |

- Most useful part of `/sys` is **`/sys/class`** – devices grouped by **type (class)**.

### Snippets

#### `/sys`

`ls` – browse the device classes:

```bash
ls /sys/class                       # all device classes
ls /sys/class/printer/lp0           # a printer (lp0)
ls /sys/class/block                 # block devices (disks/partitions)
```

`cat` – read the details:

```bash
cat /sys/class/printer/lp0/device/id
cat /sys/class/block/sda/sda1/size        # partition size (in 512-byte sectors)
cat /sys/class/block/sda/sda1/partition   # partition number
```

#### `/proc`

```bash
ls /proc/sys/dev
```

```bash
cat /proc/sys/dev/cdrom/info        # CD/DVD drive name (sr0), speed etc.
cat /proc/devices                   # block + character devices
cat /proc/meminfo                   # ➕ memory details
```

```bash
less /proc/cpuinfo                  # CPU – each core listed separately
```

#### `/dev`

```bash
ls /dev                             # device files (sda, sda1, sr0, tty…)
ls -l /dev/disk/by-uuid             # ➕ symlinks from UUIDs to real devices
```

### Remember
- Device + driver info → under **`/sys`** (esp. `/sys/class`) and **`/proc/sys/dev`**
- **`/dev`** → the device files you actually use to control/access the hardware

---

## 2. Using Linux command line tools and modules to manage hardware

### What is a PCI device?

A **PCI device** is any hardware connected through the **PCI bus** – the high-speed connections on the motherboard that link internal components to the CPU and memory.

- **PCI** = **P**eripheral **C**omponent **I**nterconnect (1990s standard).
- Modern machines use **PCI Express (PCIe)** – faster, different slots, but the OS treats it the same way → `lspci` shows both.

What usually counts as a PCI device (the **internal** stuff):

| Device | Example |
|---|---|
| Graphics card / GPU | the Radeon video controller in the video |
| Network cards | Ethernet + Wi-Fi |
| NVMe SSDs | plug straight into PCIe lanes |
| Sound cards | incl. onboard audio |
| Storage controllers | SATA / RAID controllers the hard drives hang off |
| USB controllers | the chip that runs the USB ports |
| Chipset bits | host bridges, PCI bridges… |

- Some are **cards in slots** (graphics card); many are **built into the motherboard** (onboard network, audio) but still sit on the PCI bus → `lspci` lists devices even on a laptop with no slots.

**PCI vs USB**
- **PCI** = internal, high-speed, wired straight into the system → `lspci`
- **USB** = external plug-in stuff (keyboard, mouse, sticks) → `lsusb`
- The **USB controller** itself is a PCI device; the things plugged into it are USB devices.

**Why Linux cares** – every PCI device has a **vendor ID + device ID** (like a model number). The kernel uses them to pick the right **module** (driver).

⚠ Troubleshooting tip: if a device appears in `lspci` but has **no "Kernel driver in use"** line in `lspci -k`, Linux can see the hardware but has **no module loaded** for it – first thing to check when a network card or GPU isn't working.

➕ In a VM, `lspci` shows the **virtual hardware** the hypervisor fakes (e.g. "Intel PRO/1000" NIC, "VirtualBox Graphics Adapter").

### Snippets

`lspci` – PCI devices:

```bash
lspci                  # list PCI devices (video card, network cards…)
```

```bash
lspci -vvxxx           # very verbose + hex dump – for missing/misbehaving devices
```

```bash
lspci -k               # ➕ which kernel driver/module each device uses
```

```bash
lspci -nn              # ➕ also show [vendor:device] IDs, e.g. [8086:15b8]
```

`lsusb` – USB devices:

```bash
lsusb                  # USB devices (keyboard, mouse, drives, hubs)
```

```bash
lsusb -t               # ➕ USB tree view
```

`lsmod` – loaded kernel modules:

```bash
lsmod                  # currently loaded kernel modules
```

```bash
cat /proc/modules      # same info, raw – lsmod is just a nice format of this
```

`lshw` – everything at once:

```bash
sudo lshw              # ALL hardware specs at once
```

```bash
sudo lshw -short       # ➕ compact one-line-per-device version
```

➕ Other handy ones:

```bash
lsblk                  # ➕ disks + partitions as a tree
```

```bash
lscpu                  # ➕ CPU summary (nicer than /proc/cpuinfo)
```

### Key ideas
- **ls** in these commands = "list".
- **Kernel** = core of the OS – manages processes + system resources.
- **Module** = a driver that can be **added/removed without touching the kernel** (and usually without a reboot).
- **D-Bus** = message bus system that lets **desktop** (not server) processes talk to each other and coordinate. Just be aware of it.

### Process managers – history
| Manager | When | Key point |
|---|---|---|
| **init (SysVinit)** | the original | Like air traffic control – but **synchronous** (one thing at a time) → slow |
| **Upstart** | ~2006, from Ubuntu | **Asynchronous** replacement for init |
| **systemd** | Won the debate; Ubuntu switched in **15.04** | Now the standard almost everywhere |

---

## 3. Understanding and managing Linux system modules

### Key ideas
- A device only works if its **module is loaded into the kernel**.
- Most modules are installed by default. Available (not necessarily loaded) modules live in **`/lib/modules/<kernel-version>/`**.
- `/lib/modules` has **one directory per installed kernel** – old kernels + their modules are kept in case an upgrade breaks.
- Module files end in **`.ko`** (kernel object). ➕ On new distros they're often compressed: `.ko.zst` / `.ko.xz`.
- `modprobe -l` (list all available modules) has been **removed** on many systems → use `find` instead.
- Manually added modules are **ignored by kernel upgrades** – you'll need to add them again after an upgrade.

### Snippets

`uname` – system / kernel info:

```bash
uname                  # prints the system: Linux
uname -r               # current kernel release, e.g. 6.8.0-45-generic
uname -a               # ➕ everything (kernel, hostname, arch…)
```

`ls` – installed kernels:

```bash
ls /lib/modules        # one dir per installed kernel
```

`find` – list ALL available modules for the running kernel (replacement for `modprobe -l`):

```bash
find /lib/modules/$(uname -r) -type f -name "*.ko*"
```

> `$(uname -r)` = **command substitution** – runs `uname -r` and drops the result into the path. Combining commands like this is the power of Linux.

#### Remove and reload a module (the `lp` printer module demo)

1. Is it loaded?

```bash
lsmod | grep lp
```

2. Remove it:

```bash
sudo modprobe -r lp
```

3. Check it's gone:

```bash
lsmod | grep lp
```

4. Load it again (name only – no `.ko`):

```bash
sudo modprobe lp
```

5. Check it's back:

```bash
lsmod | grep lp
```

#### Alternatives

`insmod` – load a module (needs the **FULL path**, no dependency handling):

```bash
sudo insmod /path/to/module.ko
```

`rmmod` – remove a module (like `modprobe -r`):

```bash
sudo rmmod lp
```

➕ `modinfo` – info about a module (description, parameters, file):

```bash
modinfo lp
```

| Command | Does | Handles dependencies? |
|---|---|---|
| `modprobe <name>` | load | ✅ yes |
| `modprobe -r <name>` | remove | ✅ yes |
| `insmod <file.ko>` | load | ❌ no |
| `rmmod <name>` | remove | ❌ no |

### udev – dynamic device management
- **udev** = the system tool that **manages devices** and their names.
- Settings changed with **`.rules`** files in three directories, read in this order (earlier = higher precedence):

```
/etc/udev/rules.d/     # admin's own rules  – highest priority
/run/udev/rules.d/     # runtime – /run is wiped on reboot → changes for THIS session only
/lib/udev/rules.d/     # defaults shipped by packages
```

- Rules files start with a **number** → run in **number order** (`40-…` runs before `70-…`). Rename it to e.g. `90-…` to make it run later.
- `70-persistent-net.rules` = maps network card **MAC addresses → names** (eth0, eth1).
  Replacing a failed card can give it a new name (eth3), breaking services that expect eth0 → edit this rule to fix it.

```bash
ls /etc/udev/rules.d
```

```bash
cat /etc/udev/rules.d/70-persistent-net.rules
```

```bash
udevadm monitor        # ➕ watch udev events live (plug in a USB stick)
```

➕ Modern systems use **predictable names** instead (`enp0s3`, `wlp2s0`) and often no `70-persistent-net.rules`. See the names with:

```bash
ip link
```

### Device naming

| Device | Name |
|---|---|
| Network interfaces | `eth0`, `eth1` … (some systems `em0`, `em1`; ➕ modern `enp0s3`) |
| Hard drives / SSDs | `sda`, `sdb` … ➕ NVMe: `nvme0n1` |
| Partitions | `sda1`, `sda2`, `sda3` … ➕ NVMe: `nvme0n1p1` |
| Floppy drives | `fd0`, `fd1` |
| CD/DVD drives | `sr0`, `sr1` |

All managed by **udev**, with entries in **`/dev`**.

#### Which name did my new USB drive get?

`dmesg` – kernel messages:

```bash
dmesg | tail -20       # newest messages at the bottom
```

```bash
sudo dmesg -w          # ➕ follow live, then plug the drive in
```

`lsusb` – the drive + hubs:

```bash
lsusb
```

➕ `lsblk` – shows the new disk (e.g. `sdb`) and its partitions:

```bash
lsblk
```

---

## 4. Accessing environment data on AWS instances

### Key ideas
- On AWS, the hardware is **Amazon's responsibility** – so most of the above matters less.
- AWS gives system info through **Instance Metadata**, from **inside the instance**, at the special IP **`169.254.169.254`**.

```bash
curl http://169.254.169.254/latest/meta-data/                   # list categories
curl http://169.254.169.254/latest/meta-data/instance-type
curl http://169.254.169.254/latest/meta-data/local-ipv4
```

➕ Newer instances enforce **IMDSv2**, which needs a token first. Run these two together:

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/
```

---

## 5. Understanding the Linux boot process

### Why it matters
"PC won't boot" can mean lots of things – knowing the stages tells you **where** it failed:
- No display at all → probably **hardware** (or a loose cable)
- Stuck later → **bootloader config** vs **corrupted filesystem** need very different fixes

### The boot sequence

```
Power on
  │
  ▼
BIOS (or UEFI) firmware ── checks hardware, finds the boot drive
  │
  ▼
MBR (first 512 bytes of the drive) ── partition table + how to start the OS
  │
  ▼
GRUB stage 1 (in MBR) ──► stage 1.5 (optional) ──► GRUB stage 2
  │                                                    │
  │                     menu shown (if configured), loads kernel
  ▼                                                    ▼
Kernel + initrd / initramfs (temporary root filesystem in RAM)
  │
  ▼
Kernel mounts the real root filesystem
  │
  ▼
/sbin/init  →  PID 1   (on systemd it's a symlink to systemd)
  │
  ▼
Default runlevel / target is started
```

### Key facts for the exam
- **BIOS** = Basic Input/Output System – firmware on the motherboard.
- **MBR** = Master Boot Record = **first 512 bytes** of the boot drive. Created when Linux was installed.
- **GRUB** = **GR**and **U**nified **B**ootloader.
- **initrd / initramfs** = temporary root filesystem the kernel uses until the real one is up.
- **init** always gets **PID 1**. `/sbin/init` still exists on Upstart/systemd distros (on systemd → symlink).
- Runlevels: **0–6** (7 modes, but 0 = shutdown, so really 6 "running" ones).
- Each runlevel defined by files in **`/etc/rc0.d`**, **`/etc/rc1.d`** … **`/etc/rc6.d`**.
- **UEFI** = Unified Extensible Firmware Interface – the replacement for BIOS. Not needed for the exam, but you'll meet it (and it's what makes dual-booting with Windows awkward).

### Snippets

`ls` – kernels (`vmlinuz-*`) and initrd images:

```bash
ls /boot
```

➕ `ps` – what is PID 1?

```bash
ps -p 1
```

➕ `ls -l` – on systemd this shows `/sbin/init -> /lib/systemd/systemd`:

```bash
ls -l /sbin/init
```

➕ Did I boot with UEFI or BIOS?

```bash
[ -d /sys/firmware/efi ] && echo UEFI || echo BIOS
```

---

## 6. Understanding and managing the GRUB bootloader

### Getting into the GRUB menu
- Hold / tap **right Shift** during early boot (➕ on UEFI machines it's usually **Esc**).
- If that fails: cut the boot short (hold power button 5 sec) → GRUB menu usually appears next boot.

### The menu

| Option / key | Does |
|---|---|
| Default entry | Normal boot, latest kernel |
| **Advanced options** | Boot **older kernels** (great when an upgrade breaks) + **recovery mode** per kernel |
| Recovery mode | Menu of tools to rescue a damaged install |
| Memory test | Tests RAM |
| **`e`** | **Edit** the boot parameters of the highlighted entry |
| **`c`** (or Ctrl-c) | Limited GRUB **command line** – `Esc` to go back |

Kernel files live in **`/boot`**.

### Editing boot parameters (`e`)
Find the line starting with **`linux`**:

```
linux /boot/vmlinuz-6.8.0-45-generic root=UUID=xxxx-xxxx ro quiet splash
```

- `vmlinuz-…` = the kernel to boot
- `root=` = identifier of the drive holding the root filesystem
- Both can be changed – but changes **can make the machine unbootable**.

**Emergency root shell** – add to the end of the `linux` line, then boot (Ctrl-x / F10):

```
rw init=/bin/bash
```

→ Root shell with **write access to the whole filesystem** – used to fix files / reset passwords.
⚠ Anyone at the keyboard can do this → consider a **BIOS (or GRUB) password**.

### ➕ Making GRUB changes permanent

1. Edit the settings (e.g. `GRUB_TIMEOUT=5`, `GRUB_TIMEOUT_STYLE=menu`):

```bash
sudo nano /etc/default/grub
```

2. Rebuild the GRUB config – Debian/Ubuntu:

```bash
sudo update-grub
```

…or Fedora/RHEL:

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

---

## 7. Diagnosing boot problems in Linux

### Where to look
- Almost everything leaves a record in **`/var/log`**.
- Can't boot? The logs are still reachable:
  - **Recovery mode** from GRUB, or
  - Boot a **Live USB/DVD** (any distro, or SystemRescueCD / Puppy Linux), mount the main drive, read its logs (only if the disk isn't encrypted).

### Boot-related logs

| Log | |
|---|---|
| `/var/log/kern.log` | kernel messages |
| `/var/log/boot.log` | boot messages |
| `/var/log/dmesg` / `dmesg` command | kernel ring buffer – hardware + drivers |

### grep – "your new best friend"
One boot = hundreds of lines → filter with **grep** using a **pipe** `|` (Shift + `\`).

The course demo – only lines containing that string:

```bash
cat /var/log/dmesg | grep "power/level"
```

➕ More `grep` tricks:

```bash
dmesg | grep -i "error"                      # -i = ignore case
```

```bash
dmesg | grep -i -C 3 "usb"                   # 3 lines of context around each hit
```

```bash
grep -i "fail" /var/log/boot.log             # grep can read the file directly
```

### less – read it in full context

```bash
less /var/log/dmesg
```

Keys inside `less`:

| Key | Does |
|---|---|
| `/power/level` | search forward for the string |
| `n` / `N` | next / previous match |
| `q` | quit |

### ➕ journalctl – on systemd machines the journal holds it all

```bash
journalctl -b              # this boot
```

```bash
journalctl -b -1           # previous boot (the one that failed)
```

```bash
journalctl -p err -b       # only errors from this boot
```

Still stuck? → search engine, your distro's forum, **ServerFault.com**.

### AWS
- EC2 Ubuntu instances still have `boot.log`, `dmesg`, `kern.log`.
- Console: **Instances → right-click instance → Instance Settings → Get System Log** (➕ newer console: *Actions → Monitor and troubleshoot → Get system log*).
- Then check AWS docs: **"Troubleshooting Instances with Failed Status Checks"**.

---

## 8. Controlling Linux system runlevels

### Key ideas
- **Runlevel** = which OS resources + access are available at a time.
- Runlevel **0** = shut down. Runlevel **3** = full access (multi-user).

➕ Standard runlevel table (exam favourite):

| Runlevel | Meaning | systemd target |
|---|---|---|
| 0 | Halt / power off | `poweroff.target` |
| 1 (S, single) | Single-user / maintenance | `rescue.target` |
| 2 | Multi-user (Debian/Ubuntu: full, incl. GUI) | `multi-user.target` |
| 3 | Multi-user, text mode + network (Red Hat) | `multi-user.target` |
| 4 | Unused / custom | `multi-user.target` |
| 5 | Multi-user + GUI (Red Hat) | `graphical.target` |
| 6 | Reboot | `reboot.target` |

### Where the default runlevel is set

| System | File | Entry |
|---|---|---|
| **SysVinit** | `/etc/inittab` | `id:3:initdefault:` → runlevel 3 |
| **Upstart** | `/etc/init/rc-sysinit.conf` | `env DEFAULT_RUNLEVEL=2` |
| ➕ **systemd** | `/etc/systemd/system/default.target` (symlink) | `systemctl get-default` |

⚠ Most distros **no longer use `inittab`** – lots of old docs online still talk about it.

### Service scripts
- `/etc/init.d/` → **SysVinit-style** scripts
- `/etc/init/` → **Upstart** job files

➕ `runlevel` – previous + current runlevel, e.g. `N 5`:

```bash
runlevel
```

➕ `ls` – start/kill scripts for a runlevel (`S##` = start, `K##` = kill, run in number order):

```bash
ls /etc/rc3.d
```

➕ Old-style way to control a service:

```bash
sudo /etc/init.d/ssh status
```

➕ `service` – wrapper that works on SysV / Upstart / systemd:

```bash
sudo service ssh restart
```

---

## 9. How systemd controls the Linux environment

### Key ideas
- Demo done on **Fedora** – the first major distro to adopt systemd.
- systemd focuses on the **state of individual processes**, not on whole-system runlevels.
- It works with **Units** – plain text files describing a **service, device, socket or mount point**.

| Location | Who uses it |
|---|---|
| `/etc/systemd/system/` | **You** (admin) – manual units/overrides, take priority |
| `/usr/lib/systemd/system/` | **Packages** install their units here (➕ `/lib/systemd/system/` on Debian/Ubuntu) |

Example: `tmp.mount` → mounts **`/tmp`** as **tmpfs** (in RAM) for short-term data.

```bash
cat /usr/lib/systemd/system/tmp.mount
```

### systemd-analyze – diagnosing slow boots

```bash
systemd-analyze                  # total time: firmware / loader / kernel / userspace
```

```bash
systemd-analyze blame            # time per unit (ms) – slowest first
```

```bash
systemd-analyze critical-chain   # ➕ the chain of units that held up boot
```

### systemctl – the main tool

List units:

```bash
systemctl list-units                         # all active units
```

```bash
systemctl list-units | grep httpd            # narrow it down
```

#### The Apache (`httpd`) demo

1. Is it running?

```bash
systemctl status httpd.service
```

2. Start it at every boot (doesn't start it now):

```bash
sudo systemctl enable httpd.service
```

3. Start it right now:

```bash
sudo systemctl start httpd.service
```

4. Confirm:

```bash
systemctl list-units | grep httpd
```

Stop it:

```bash
sudo systemctl stop httpd.service
```

#### ➕ Extras worth knowing

```bash
sudo systemctl enable --now httpd            # enable + start in one go
```

```bash
sudo systemctl restart httpd                 # stop + start
```

```bash
sudo systemctl disable httpd                 # don't start at boot
```

```bash
systemctl is-enabled httpd
```

```bash
systemctl list-units --type=service --state=failed   # what's broken?
```

- Apache = **`httpd`** on Fedora/RHEL, **`apache2`** on Debian/Ubuntu.
- Services are referred to with **`.service`** on the end (optional for services).
- ➕ **enable ≠ start**: enable = at boot; start = now.

#### ➕ Runlevels the systemd way

```bash
systemctl get-default                          # current default target
```

```bash
sudo systemctl set-default multi-user.target   # boot to text mode
```

```bash
sudo systemctl isolate rescue.target           # switch now (like telinit 1)
```

---

## 10. Changing runlevels from GRUB and the command line (all process managers)

### Single-user mode
- From GRUB: press **`e`**, add **`single`** (or `1`) to the end of the kernel (`linux`) line → boots single-user. Used for **emergency recovery**.
- From a running system:

`telinit`:

```bash
sudo telinit 1                 # go to single-user mode
```

```bash
sudo telinit 0                 # power off
```

```bash
sudo telinit 6                 # ➕ reboot
```

### shutdown

```bash
sudo shutdown -h now           # halt now   (-h = halt)
```

```bash
sudo shutdown -h 2             # halt in 2 minutes – all users get a warning (Ctrl-c to cancel if still in the foreground)
```

```bash
sudo shutdown -r 5             # reboot in 5 minutes  (-r = reboot)
```

```bash
sudo shutdown -c               # ➕ cancel a scheduled shutdown from any terminal
```

```bash
sudo shutdown -h 22:00 "Maintenance tonight"   # ➕ at a set time, with a message
```

### Telling other users – be a good admin
- `shutdown` / `telinit` **automatically warn** every logged-in user (if they're looking at their terminal!).
- **`wall`** ("write all") = send a message to **every logged-in user's terminal**.

`wall`:

```bash
cat message.txt | wall                    # send a text file to everyone
```

```bash
wall "Server reboot in 10 minutes"        # ➕ quick one-liner
```

➕ `who` – who's logged in (who will see it):

```bash
who
```

---

## Quick revision

| Topic | Commands / paths |
|---|---|
| Hardware | `lspci` · `lspci -vvxxx` · `lspci -k` · `lsusb` · `lsmod` · `sudo lshw` · `dmesg` |
| Virtual files | `/proc/cpuinfo` · `/proc/devices` · `/proc/modules` · `/sys/class` · `/dev` |
| Modules | `uname -r` · `find /lib/modules/$(uname -r) -type f -name "*.ko*"` · `modprobe <mod>` · `modprobe -r <mod>` · `insmod` · `rmmod` |
| udev rules (priority order) | `/etc/udev/rules.d` → `/run/udev/rules.d` → `/lib/udev/rules.d` |
| Boot / logs | `/boot` · `/var/log/kern.log` · `/var/log/boot.log` · `dmesg \| grep "text"` |
| GRUB | right Shift = menu · `e` = edit · `c` = command line · add `single` or `rw init=/bin/bash` |
| Runlevel files | `/etc/inittab` (`id:3:initdefault:`) · `/etc/init/rc-sysinit.conf` (`DEFAULT_RUNLEVEL=2`) |
| Runlevel commands | `telinit 0\|1\|6` · `shutdown -h now` · `shutdown -r 5` · `cat file \| wall` |
| systemd | `systemd-analyze` · `systemd-analyze blame` · `systemctl list-units / start / stop / enable / status <unit>.service` |
| systemd unit dirs | `/etc/systemd/system` · `/usr/lib/systemd/system` |

### Exam-style check questions
1. Which two directories hold kernel-generated device/driver info? → `/sys` (sysfs) and `/proc`
2. What is `lsmod` a formatted view of? → `/proc/modules`
3. Command to load a module **with** its dependencies? → `modprobe`
4. How big is the MBR? → 512 bytes
5. Which PID does init get? → 1
6. In which file did SysVinit set the default runlevel? → `/etc/inittab`
7. Which udev rules directory only lasts until reboot? → `/run/udev/rules.d`
8. Show how long each unit took at boot? → `systemd-analyze blame`
9. Reboot in 5 minutes? → `shutdown -r 5`
10. Message every logged-in user? → `wall`
11. What does PCI stand for, and which command lists PCI devices? → Peripheral Component Interconnect, `lspci`
12. A device shows in `lspci` but has no "Kernel driver in use" in `lspci -k` – what's wrong? → no module (driver) loaded for it
