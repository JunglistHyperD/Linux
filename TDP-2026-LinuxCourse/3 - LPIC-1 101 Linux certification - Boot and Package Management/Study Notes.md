# LPIC-1 101 – Boot and Package Management – Study Notes

**What the course covers**
- Designing disk partitions + filesystems for an install
- The Logical Volume Manager (**LVM**)
- Installing + configuring the **GRUB** boot manager
- **Shared libraries** – how programs share code
- Package management: **dpkg / APT** (Debian family) and **RPM / Yum** (Red Hat family)

> ➕ = not in the course, added for real-world use. ⚠ = correction or caveat to what the course says.

---

## 1. Designing Linux disk partitions

### Key ideas
- Good disk design balances two opposites: **separation** (users can't see each other's files or system config) and **accessibility** (they can still work).
- **Partition** = a portion of a disk logically walled off from the rest. To software, each partition looks like a separate disk.
  - e.g. a 1 TB disk → 4 **primary** partitions of 250 GB each (MBR disks allow max 4 primary; ➕ GPT disks allow 128).
- The reverse also works: make several physical disks look like **one** via **LVM** (section 2) or **mount points**.

### The top-level directories, in four groups

| Group | Directories |
|---|---|
| User data | `/home` |
| System | `/etc`, `/var`, `/lib` … |
| Virtual / pseudo | `/dev`, `/proc` (see section 2 notes) |
| Boot | `/boot` – a category of its own |

### Typical partition layout

| Partition | Size | Why |
|---|---|---|
| `/` (root) | The main Linux system is happy in **< 4 GB** (➕ allow 20–30 GB today) | Everything not split off elsewhere |
| `/boot` | ~**500 MB** | Kernels + initrd images |
| **swap** | **2 × RAM** for desktops, **= RAM** for servers (course rule of thumb) | Virtual memory when RAM runs out |

Worth splitting off as well:
- **`/home`** – if the system files get corrupted, user data survives. Copy `/home` to the rebuilt machine; you only need to recreate the accounts and passwords.
- **`/var`** – logs live in `/var/log` and can grow huge (log files bigger than 100 GB have been seen; or a forgotten test container keeps logging). A separate `/var` stops that from filling up the whole system. Also useful on shared dev/test servers.

⚠ **Plan sizes up front.** Partitions *can* be resized, but every resize risks failure, and failure here usually means **losing all data on the disk**.

➕ The course's swap rule is from the era of small RAM. Today a few GB (or a swap file) is usually enough unless you need hibernation (which needs at least the size of RAM).

### Reading a real layout (the GParted demo)
**GParted** = GUI front end for the command-line tool **`parted`**. It's the one GUI tool the course uses. Demo disk `sdb`:

| Partition | Size | Mounted on | Notes |
|---|---|---|---|
| `sdb1` | 500 MB | `/boot/efi` | **boot flag** set, formatted **FAT32** (so Windows/firmware can read it) |
| `sdb2` | ~500 GB | `/` | root |
| `sdb3` | – | – | `linux-swap` |

⚠ The course calls `sdb1` the partition "with the Master Boot Record". A FAT32 partition mounted on `/boot/efi` is actually the **EFI System Partition (ESP)** used by UEFI. The **MBR** is the first 512 bytes of the *disk*, not a partition.

Each partition also appears as a device file in `/dev` (`/dev/sdb1` …).

### Snippets

`lsblk` – all block devices + partitions as a tree:

```bash
lsblk
```

`fdisk -l` – much more detail (partition table type, sectors, types):

```bash
sudo fdisk -l
```

➕ Sizes and free space of mounted filesystems:

```bash
df -h
```

#### Mounting a USB drive (quick preview – mounting is covered properly in a later course)

1. Make a directory to mount it on:

```bash
cd /tmp
mkdir drive
```

2. Mount the drive's first partition there:

```bash
sudo mount /dev/sdc1 drive
```

3. Its contents now appear under `/tmp/drive`:

```bash
cd drive
ls
```

➕ When finished, step out of the directory and unmount it:

```bash
cd /tmp
sudo umount drive
```

### Remember
- Normal install = **3 partitions**: `/`, `/boot`, **swap**. Split off `/home` and/or `/var` for safety.
- `lsblk` = quick overview; `sudo fdisk -l` = full detail.

---

## 2. The Logical Volume Manager (LVM)

### Key ideas
- **LVM** treats all the physical disks as **one pool** and lets you resize and rearrange the pieces easily and safely.
- For the exam you **don't need to configure LVM**, just know the basic terms and tools (**LVM2**).

| Term | Means |
|---|---|
| **PV** – Physical Volume | A real disk or partition handed to LVM |
| **VG** – Volume Group | A pool made of one or more PVs |
| **LV** – Logical Volume | A virtual "partition" carved out of a VG – you put a filesystem on this |

```text
 /dev/sdb2   /dev/sdb3        <- PVs
     \         /
      \       /
       my_vg                  <- VG (pooled space)
      /     \
   my_lv    (free)            <- LVs (formatted + mounted like partitions)
```

### Snippets

➕ First mark the partitions as PVs (the course skips this step):

```bash
sudo pvcreate /dev/sdb2 /dev/sdb3
```

`vgcreate` – create a volume group from one or more partitions:

```bash
sudo vgcreate my_vg /dev/sdb2 /dev/sdb3
```

`lvcreate` – carve a 250 GB logical volume called `my_lv` out of `my_vg`:

```bash
sudo lvcreate -L 250G -n my_lv my_vg
```

- `-L` = size, `-n` = name, then the VG to take the space from.
- ⚠ The course says `/l` and `/n`. Linux options use a **dash**: `-L` and `-n`.

`lvscan` – confirm it worked:

```bash
sudo lvscan
```

➕ Summary views of each layer:

```bash
sudo pvs
sudo vgs
sudo lvs
```

➕ The LV appears as a device you can format and mount:

```bash
sudo mkfs.ext4 /dev/my_vg/my_lv
```

### Remember
- **PV → VG → LV**: disks go into a pool, and volumes come out of the pool.
- `vgcreate` · `lvcreate` · `lvscan`

---

## 3. Installing and managing the GRUB boot manager

> Using the GRUB **menu** (Shift/Esc, `e` to edit, `single`, `rw init=/bin/bash`) is in the **section 2 notes**. This section covers **installing** GRUB and changing its **configuration**.

### Key ideas
- **GRUB Legacy** (old) vs **GRUB 2** (current). Files and commands vary between distros and even between releases.
- ⚠ "GRUB2" can mean **GRUB version 2** *or* **GRUB stage 2** of the boot process. Context tells you which.
- Why install or update GRUB on a working machine?
  1. An old, rarely rebooted server may run a GRUB version with **known bugs**.
  2. Far more common: **recovering a broken GRUB** (Windows overwrote it, an update went wrong).

### Is GRUB installed?

Fedora (note: the package is called `grub2`):

```bash
rpm -q grub2
```

➕ Debian/Ubuntu equivalent:

```bash
dpkg -l | grep grub
```

### Installing GRUB

Fedora:

```bash
sudo grub2-install /dev/sda
```

Debian/Ubuntu:

```bash
sudo grub-install /dev/sda
```

- The target is the **disk** (`/dev/sda`), not a partition (`/dev/sda1`). Often that's all you need.

### The config files

| File | Role | Edit it? |
|---|---|---|
| `/boot/grub/grub.cfg` (Ubuntu) · `/boot/grub2/grub.cfg` (Fedora) · `menu.lst` (Legacy, SUSE / older Debian) | The script GRUB actually reads at boot | ❌ **No** – it's generated, so edits get overwritten |
| `/etc/default/grub` | **Settings**: menu look, default OS, timeout | ✅ Yes |
| `/etc/grub.d/` | **Templates**: one script per OS / menu entry. Add files here for custom entries | ✅ Yes |

Key settings in `/etc/default/grub`:

| Setting | Controls |
|---|---|
| `GRUB_DEFAULT` | Which OS boots by default if none is chosen before the timeout |
| `GRUB_TIMEOUT` | Seconds before the default boots |
| ➕ `GRUB_CMDLINE_LINUX_DEFAULT` | Kernel parameters, e.g. `"quiet splash"` |

(Menu colours and other settings can also be changed.)

### Applying changes – regenerate `grub.cfg`

Fedora:

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

Debian/Ubuntu:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

…or the shortcut:

```bash
sudo update-grub
```

- `update-grub2` is a **symlink** to `update-grub`, which is just a script that runs `grub-mkconfig`. All three do the same job.

```bash
ls -l /usr/sbin/update-grub*
```

### ➕ Boot-Repair (not on the exam, but it can save your skin)
- Free tool that **repairs a broken boot process** automatically.
- Can't boot? Either make a **Boot-Repair USB** and boot from it, or boot a **live Linux session**, install Boot-Repair, and let it fix the installed system.

### Remember
- Install: `grub-install` / `grub2-install` + **disk**.
- Edit `/etc/default/grub` + `/etc/grub.d/`, **never** `grub.cfg` directly. Then run `grub-mkconfig -o …` / `update-grub`.
- `GRUB_DEFAULT` = default OS, `GRUB_TIMEOUT` = delay.

---

## 4. Working with Linux shared libraries

### Key ideas
- **Library** = ready-made code that programs call instead of reinventing the wheel (e.g. talking to the filesystem). Programs end up smaller and faster, and there are more of them.

| | Static | Dynamic (shared) |
|---|---|---|
| File ending | **`.a`** | **`.so`** (shared object) |
| How it's used | Copied **into** the program | Loaded from the library file **each time the program runs** |
| Pros | Program is **self-contained**, no external files needed. Great for **system recovery tools** | **Leaner**: one copy on disk/RAM shared by many programs |
| Cons | Bigger programs | Library must stay installed and findable |

⚠ The course says static libraries are read "as the program is installed". Strictly, it happens when the program is **built (linked)**.

- Package managers (apt, yum) track a package's **dependencies** (the libraries it needs) and install them automatically. You deal with libraries by hand only for software from **outside the repos**.

### Library file names

```text
libQtGui.so.4
│  │     │  └── version
│  │     └───── so = dynamic (a = static)
│  └─────────── unique library name
└────────────── lib = it's a library
```

- Most libraries live in or under **`/lib`** (some in category subdirectories) and `/usr/lib`.

### Where the system finds libraries

| File | What it is |
|---|---|
| `/etc/ld.so.cache` | **Binary** list of available libraries – what the loader actually reads. Not human-readable |
| `/etc/ld.so.conf` | Human-readable config. On Ubuntu/Fedora it just **includes** `/etc/ld.so.conf.d/` |
| `/etc/ld.so.conf.d/*.conf` | Each file lists **directories** that hold libraries (e.g. `/usr/lib/...`) |

⚠ The course says `ld.so.conf` "mirrors" the cache. It's actually the other way round: `ldconfig` **reads** the `.conf` files and **builds** the cache from them.

### Snippets

`ldd` – which libraries a program (or a library) depends on, and where they are:

```bash
ldd /bin/ls
```

- On a binary it also shows whether it's **dynamically linked** ("not a dynamic executable" = static).

`ldconfig -p` – print all libraries in the cache (often 1,700+), so filter with grep:

```bash
ldconfig -p | grep libQtGui
```

➕ Peek at the config chain:

```bash
cat /etc/ld.so.conf
ls /etc/ld.so.conf.d/
```

#### Adding your own library

Option 1 – **permanent**: create a `.conf` file pointing at the directory, then rebuild the cache:

```bash
echo "/opt/mylib/lib" | sudo tee /etc/ld.so.conf.d/my_lib_name.conf
```

```bash
sudo ldconfig -v               # rebuild the cache; -v = verbose (show what it's doing)
```

Option 2 – **this shell only**: add the directory to `LD_LIBRARY_PATH`:

```bash
export LD_LIBRARY_PATH=/opt/mylib/lib:$LD_LIBRARY_PATH
```

### Remember
- `.so` = dynamic, `.a` = static.
- `ldd` = dependencies of a program/library · `ldconfig -p` = list all · `ldconfig` = rebuild the list.
- New library → file in `/etc/ld.so.conf.d/` + `sudo ldconfig`, **or** `LD_LIBRARY_PATH`.

---

## 5. Debian package management – dpkg and APT

### Key ideas
- The two big package systems:

| System | Package file | Distros | Low-level tool | Front ends |
|---|---|---|---|---|
| **dpkg** | `.deb` | Debian, Ubuntu, Linux Mint | `dpkg` | `apt-get`, `apt-cache`, aptitude, Synaptic |
| **RPM** ("R" was originally Red Hat) | `.rpm` | Red Hat, Fedora, openSUSE | `rpm` | `yum` (➕ now `dnf`) |

- A `.deb` **won't install** on an RPM system and vice versa. Most software is available in both formats, though.
- Repos are **curated**, so the software is functional and malware-free, and dependencies are handled for you.
- **dpkg = local `.deb` files only. APT = online repositories** (and it resolves dependencies).

### Package file names

```text
my_package_1.2.3_i386.deb
│          │     │    └── Debian package
│          │     └─────── architecture (i386 = 32-bit, amd64 = 64-bit)
│          └───────────── version
└──────────────────────── package name
```

### dpkg

Its defaults are in **`/etc/dpkg/dpkg.cfg`** (here just package-signature and log settings).

| Command | Does |
|---|---|
| `sudo dpkg -i my_package_1.2.3_i386.deb` | **Install** a downloaded `.deb` (full file name) |
| `sudo dpkg --unpack my_package_1.2.3_i386.deb` | Unpack **without configuring** (for a custom setup) |
| `sudo dpkg -r my_package` | **Remove** (short name), keep config files |
| `sudo dpkg -P my_package` | **Purge**: remove files **and** settings (`--purge`) |
| `sudo dpkg-reconfigure my_package` | Re-run the configuration of an installed package |
| `sudo debconf-show my_package` | **Show** a package's configuration |

➕ Everyday extras:

```bash
dpkg -l | grep ssh            # list installed packages (filter)
```

```bash
dpkg -L openssh-server        # files a package installed
```

```bash
dpkg -S /usr/sbin/sshd        # which package owns this file?
```

### APT – repositories

Repos are listed in **`/etc/apt/sources.list`**. Each line looks like this:

```text
deb      http://archive.ubuntu.com/ubuntu  trusty  main universe multiverse
deb-src  http://archive.ubuntu.com/ubuntu  trusty  main universe multiverse
│        │                                 │       └── categories (components)
│        │                                 └────────── release (trusty = Ubuntu 14.04)
│        └──────────────────────────────────────────── repo URL
└───────────────────────────────────────────────────── deb = binary packages, deb-src = source code
```

- To add a trusted third-party repo, edit this file. **Get the syntax exactly right** or it won't work.
- ➕ Modern systems put extra repos in **`/etc/apt/sources.list.d/`**. Ubuntu 24.04 uses `ubuntu.sources` there, in a newer multi-line format.

### apt-get

| Command | Does |
|---|---|
| `sudo apt-get update` | Refresh the package **lists** from the repos (installs nothing) |
| `sudo apt-get install <pkg>` | Install a package + its dependencies |
| `sudo apt-get upgrade` | Upgrade **all** installed packages |
| `sudo apt-get dist-upgrade` | Upgrade, **also** adding/removing packages when dependencies change |
| `sudo apt-get remove <pkg>` | Remove a package |
| `apt-get install -s <pkg>` | **Simulate**: show what would happen, install nothing |

➕ Extras:

```bash
sudo apt-get purge <pkg>       # remove + config files
```

```bash
sudo apt-get autoremove        # remove dependencies nothing needs any more
```

```bash
apt list --installed           # newer 'apt' front end – friendlier output
```

➕ Typical routine: `sudo apt-get update && sudo apt-get upgrade`.

### apt-cache – look things up

| Command | Does |
|---|---|
| `apt-cache stats` | Statistics about the package cache |
| `apt-cache depends <pkg>` | A package's dependencies |
| `apt-cache unmet <pkg>` | Unmet dependencies of a package. With **no package** = the whole system (long!) |
| ➕ `apt-cache search <word>` | Find packages by keyword |
| ➕ `apt-cache show <pkg>` | Full details of a package |

⚠ The course describes `stats` as info "about a specified package". It actually gives overall cache statistics and takes no package name.

### Other front ends
- **aptitude** – **terminal menu** interface over APT. The exam expects you to know it exists. `?` = help, `q` = quit.
- **Synaptic** – **desktop GUI**. Does most of what apt-get does, and is good for **research** (search packages, see files + versions).

### Remember
- `dpkg -i / --unpack / -r / -P` · `dpkg-reconfigure` · `debconf-show`
- Repos: `/etc/apt/sources.list` (`deb` / `deb-src`, URL, release, categories)
- `apt-get update / install / upgrade / dist-upgrade / remove / install -s`
- `apt-cache stats / depends / unmet`

---

## 6. Red Hat package management – RPM and Yum

> The course's sysadmin story: package management was the **#1 skill** asked for in their first Linux job. It underpins everything else, and being comfortable with apt + yum is a good sign of general experience. The LPI rates it highly too.

### Package file names

```text
apacheds-2.0.0-M20-x86_64.rpm
│        │          │      └── RPM package
│        │          └───────── architecture (x86_64 = 64-bit)
│        └──────────────────── version
└───────────────────────────── package name (ApacheDS = an LDAP + Kerberos server)
```

### rpm – local package files

| Command | Does |
|---|---|
| `sudo rpm -i <file>.rpm` | **Install** |
| `sudo rpm -U <file>.rpm` | **Upgrade** (or install if missing). ⚠ `-U` = **U**pgrade, not uninstall! |
| `rpm -q <pkg>` | **Query**: is it installed? (prints the version) |
| `rpm -i --test <file>.rpm` | **Test** the install / check dependencies without installing |
| `sudo rpm -e <pkg>` | **Erase** (remove) |
| `sudo rpm -e --nodeps <pkg>` | Remove **skipping the dependency check** |
| `rpm -vK <file>.rpm` | **Checksum / signature** check. Compare with what the package source publishes |
| `rpm -V <pkg>` | **Verify** an installed package's files against the RPM database |

- The course demo: `rpm -i --test` caught a **64-bit** package on a **32-bit** machine before anything was installed.
- ⚠ The course says `--nodeps` "respects other packages' dependencies". It's the **opposite**: it **ignores** them and removes the package even if others need it. Use with care.

➕ Handy queries:

```bash
rpm -qa | grep ssh             # all installed packages (filter)
```

```bash
rpm -ql openssh-server         # files a package installed
```

```bash
rpm -qf /usr/sbin/sshd         # which package owns this file?
```

```bash
rpm -qpR <file>.rpm            # dependencies of an uninstalled .rpm
```

#### Extracting files from an `.rpm` without installing
RPMs are **cpio** archives inside. `rpm2cpio` converts the package, and `cpio` unpacks it:

```bash
rpm2cpio apacheds-2.0.0-M20-x86_64.rpm | cpio -id
```

- `-i` = extract (restore) the archive, `-d` = create leading directories as needed. ➕ Add `-v` to list files as they come out.

### Yum – the front end (like apt-get)

Repo config:
- **`/etc/yum.repos.d/`** – **one `.repo` file per repo**, unlike apt's single `sources.list`.
- **`/etc/yum.conf`** – main config. Repo info can also go here.

| Command | Does |
|---|---|
| `sudo yum install xboard` | Install (xboard = a chess game) |
| `sudo yum remove xboard` | Remove |
| `sudo yum update [pkg]` | Update one package, or everything if none given |
| `yum search <word>` | Search for a package |
| `yum list` | All available packages. Long, so pipe to grep |
| `yum list installed` | Installed packages only |
| `yumdownloader --resolve <pkg>` | **Download** without installing. `--resolve` also grabs the dependencies |

➕ **dnf** replaced yum on Fedora 22+, RHEL 8+ and Amazon Linux 2023. The commands are almost identical (`dnf install`, `dnf search` …), and `yum` usually still works as an alias.

### Remember
- `rpm -i / -U / -q / -e / -K / -V` · `rpm -i --test`
- `rpm2cpio <file>.rpm | cpio -id` to extract
- Repos: `/etc/yum.repos.d/` + `/etc/yum.conf`
- `yum install / remove / update / search / list / list installed` · `yumdownloader`

---

## 7. Package management on AWS

- **Amazon Linux** instances use **RPM + yum** (➕ Amazon Linux 2023: `dnf`).
- `/etc/yum.repos.d/` points to Amazon-managed repos, e.g. **`amzn-main`**.
- **Ubuntu** instances use **dpkg + APT** as usual.
- Software can also be provisioned **by script** via deployment services like **Elastic Beanstalk** (➕ or EC2 **user data** scripts at launch).

```bash
ls /etc/yum.repos.d/
```

---

## Quick revision

| Topic | Commands / paths |
|---|---|
| Partitions | `/` · `/boot` (~500 MB) · swap (2×RAM desktop / =RAM server) · split `/home`, `/var` · `lsblk` · `sudo fdisk -l` · GParted (GUI for `parted`) |
| Mounting | `sudo mount /dev/sdc1 /tmp/drive` · ➕ `umount` |
| LVM | PV → VG → LV · `vgcreate my_vg /dev/sdb2 /dev/sdb3` · `lvcreate -L 250G -n my_lv my_vg` · `lvscan` |
| GRUB install | `grub-install /dev/sda` · `grub2-install /dev/sda` (Fedora) · check: `rpm -q grub2` |
| GRUB config | edit `/etc/default/grub` + `/etc/grub.d/` → `grub-mkconfig -o /boot/grub/grub.cfg` (Fedora: `grub2-mkconfig -o /boot/grub2/grub.cfg`) · `update-grub` · Legacy: `menu.lst` |
| Libraries | `.so` dynamic / `.a` static · `ldd <prog>` · `ldconfig -p \| grep <lib>` · `ldconfig -v` · `/etc/ld.so.cache` · `/etc/ld.so.conf(.d)` · `LD_LIBRARY_PATH` |
| dpkg | `/etc/dpkg/dpkg.cfg` · `dpkg -i` · `--unpack` · `-r` · `-P` · `dpkg-reconfigure` · `debconf-show` |
| APT | `/etc/apt/sources.list` · `apt-get update / install / upgrade / dist-upgrade / remove / install -s` · `apt-cache stats / depends / unmet` · aptitude · Synaptic |
| rpm | `rpm -i` · `-U` · `-q` · `-i --test` · `-e` · `-e --nodeps` · `-vK` · `-V` · `rpm2cpio x.rpm \| cpio -id` |
| Yum | `/etc/yum.repos.d/` · `/etc/yum.conf` · `yum install / remove / update / search / list / list installed` · `yumdownloader --resolve` |

### Exam-style check questions
1. Which three partitions does a normal install create? → `/` (root), `/boot`, swap
2. Why put `/var` on its own partition? → runaway logs can't fill the whole system
3. Command for full partition details of every disk? → `sudo fdisk -l`
4. In LVM, what are PV, VG and LV? → physical volume, volume group (pool of PVs), logical volume (carved from a VG)
5. Create a 250 GB LV named `my_lv` from `my_vg`? → `lvcreate -L 250G -n my_lv my_vg`
6. Why shouldn't you edit `grub.cfg` directly? → it's generated by `grub-mkconfig` from `/etc/default/grub` + `/etc/grub.d/`, so edits are overwritten
7. Which setting picks the default OS in GRUB? → `GRUB_DEFAULT`
8. What does `update-grub` actually run? → `grub-mkconfig`
9. File ending of a dynamic vs a static library? → `.so` vs `.a`
10. Show the libraries a program depends on? → `ldd <program>`
11. After adding a `.conf` in `/etc/ld.so.conf.d/`, what must you run? → `ldconfig`
12. Install a local `.deb`? Purge a package? → `dpkg -i <file>.deb` · `dpkg -P <pkg>`
13. Where does APT get its repo list? → `/etc/apt/sources.list`
14. Difference between `apt-get upgrade` and `dist-upgrade`? → `dist-upgrade` also adds/removes packages when dependencies change
15. Which `rpm` option upgrades a package? → `-U`
16. Check an `.rpm`'s dependencies without installing? → `rpm -i --test <file>.rpm` (➕ or `rpm -qpR`)
17. Extract files from an `.rpm` without installing? → `rpm2cpio <file>.rpm | cpio -id`
18. Where are yum's repo files? → `/etc/yum.repos.d/` (one file per repo) and `/etc/yum.conf`
19. Download a package and its dependencies without installing (RPM world)? → `yumdownloader --resolve <pkg>`
