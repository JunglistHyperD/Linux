

Core dirs in Linux 

- `home`  Where the Users stuff lives
- etc ???
- var
- usr
- bin
- `dev` - Devices
- sys



---



# LXC 

```
sudo apt-get update
sudo apt-get install lxc
```

(required )INstal lthe lxc templates
```
sudo apt-get install lxc-templates
```


```
lxc-create -t ubuntu -n newPC   £ template and name of the container
```


```
lxc-ls     # List all the containers
```

Start `newpc` and allow Container detatchment (`-d`)
```
lxc-start -n newpc -d 
```


```
lxc-ls  --fancy   # List all the containers and their local Ip detasil , for ssh accees
```



---

## LXC – from the transcript (lab I couldn't access) + extras

> LXC = Linux Containers. A container is a lightweight "virtual machine" that shares the host's kernel.
> Builds in a couple of minutes, reboots in seconds, and is **disposable** – perfect sandbox for breaking things while learning.
> Nothing done inside the container affects the host.

### ⚠ Gotchas with my commands above

- **Every `lxc-*` command needs `sudo`** (transcript: "all LXC commands you issue from the hypervisor require administrator rights").
  Without it `lxc-ls` just shows an **empty list** – containers are not gone, you just can't see root's ones.
- **Names are case-sensitive** – `newPC` and `newpc` are two different containers. Pick one and stick to it.
- Comments in bash start with `#` – a `£` would be passed to the command as an argument.
- `-t ubuntu` builds the filesystem with `debootstrap` → `sudo apt install debootstrap` if create fails. First build is slow, later ones use a cache.
- `apt` says *"Could not get lock … unattended-upgr"* → background updates are running. Wait (`watch -n 0.5 'pgrep -a unattended'`), never kill it.

### The lab workflow (as in the transcript)

```bash
sudo apt-get update
sudo apt-get install lxc lxc-templates     # templates needed for -t ubuntu

sudo lxc-create -t ubuntu -n newpc          # -t template, -n name  (2-3 minutes)
sudo lxc-ls                                 # list containers
sudo lxc-start -n newpc -d                  # start, -d = detached (shell stays free)
sudo lxc-ls --fancy                         # status + IP address of running containers
```

SSH in (host needs an ssh client; the ubuntu template already puts `openssh-server` in the container):

```bash
sudo apt-get install openssh-server         # if ssh isn't on your machine
ssh ubuntu@10.0.3.xxx                       # IP from lxc-ls --fancy
```

- Default login for the **ubuntu template**: user `ubuntu` / password `ubuntu`
- Change it straight away if the container is used for anything serious: `passwd`

Containers only have the **base system** – add the basics yourself (inside the container):

```bash
sudo apt update
sudo apt install nano vim curl wget net-tools iputils-ping less man-db
```

Destroy a broken / unneeded container:

```bash
sudo lxc-destroy -n newpc                   # add -f to force it if it's still running
```

### Fixing a container from the outside

Everything lives under `/var/lib/lxc/<name>/` (needs root):

```bash
sudo -i                                     # root shell
cd /var/lib/lxc
ls                                          # one directory per container
cd newpc
ls                                          # config  fstab  rootfs
cat config                                  # plain-text container settings
cd rootfs                                   # the container's whole filesystem / (works even when stopped)
```

So a broken config file inside the container can be edited from the host, e.g.:

```bash
sudo nano /var/lib/lxc/newpc/rootfs/etc/hosts
```

Forgot the container password? Reset it from the host (container stopped):

```bash
sudo lxc-stop -n newpc
sudo chroot /var/lib/lxc/newpc/rootfs passwd ubuntu
```

---

### Taking it further – the useful extras

#### Get a shell without SSH

```bash
sudo lxc-attach -n newpc                           # root shell straight into the container
sudo lxc-attach -n newpc -- apt install -y nano    # run one command and come back
exit                                               # leave (container keeps running)
```

#### Day-to-day commands

| Command | What it does |
|---|---|
| `sudo lxc-ls -f` | Short for `--fancy` – state, autostart, IPs |
| `sudo lxc-info -n newpc` | State, PID, IP, memory/CPU use |
| `sudo lxc-stop -n newpc` | Clean shutdown (`-k` = kill now) |
| `sudo lxc-start -n newpc -F` | Start in the **foreground** – handy to see why it won't boot |
| `sudo lxc-console -n newpc` | Attach to its console (`Ctrl+a` then `q` to get out) |
| `sudo lxc-checkconfig` | Checks the kernel supports everything LXC needs |
| `sudo lxc-destroy -n newpc -f` | Stop + delete in one go |

> On newer LXC, starting detached is already the default – `-d` does no harm though.

#### Modern way to create: the `download` template

Faster than `-t ubuntu` (pulls a ready-made image instead of building one) and gives other distros too:

```bash
sudo lxc-create -t download -n box1 -- --list                         # see what's available
sudo lxc-create -t download -n box1 -- -d ubuntu -r noble -a amd64     # Ubuntu 24.04
sudo lxc-create -t download -n deb1 -- -d debian -r bookworm -a amd64  # Debian 12
sudo lxc-create -t download -n alp1 -- -d alpine -r 3.20 -a amd64      # tiny Alpine
```

⚠ Download images have **no `ubuntu`/`ubuntu` login** – use `lxc-attach`, then set a password / install `openssh-server` yourself.

#### Clone & snapshot – the "undo button" for a sandbox

Set a container up the way you like, then copy or snapshot it before experimenting.
(Container must be **stopped**.)

```bash
sudo lxc-stop -n newpc

sudo lxc-copy -n newpc -N newpc2            # full clone with a new name

sudo lxc-snapshot -n newpc                  # take snapshot (snap0, snap1, …)
sudo lxc-snapshot -n newpc -L               # list snapshots
sudo lxc-snapshot -n newpc -r snap0         # roll back to snap0
sudo lxc-snapshot -n newpc -d snap0         # delete a snapshot
```

#### Autostart with the host

Add to `/var/lib/lxc/newpc/config`:

```
lxc.start.auto = 1
lxc.start.delay = 5
```

Check with `sudo lxc-ls -f` (AUTOSTART column).

#### Limit resources (so a test can't eat the host)

On the fly (lost on restart):

```bash
sudo lxc-cgroup -n newpc memory.max 512M
```

Permanently – add to the container's `config`:

```
lxc.cgroup2.memory.max = 512M
lxc.cgroup2.cpuset.cpus = 0
```

#### Networking

- LXC creates the bridge **`lxcbr0`**; containers get addresses in **`10.0.3.0/24`** and reach the internet through NAT.
- Container has no IP? Check the network service on the host:

```bash
systemctl status lxc-net
sudo systemctl restart lxc-net
ip addr show lxcbr0
```

- Copy your SSH key in so you stop typing passwords: `ssh-copy-id ubuntu@10.0.3.xxx`

#### Won't start? Get a log

```bash
sudo lxc-start -n newpc -l DEBUG -o ~/newpc.log
less ~/newpc.log
```

---

### LXC vs LXD – don't mix them up

I also installed **LXD** (`snap install lxd` + `sudo lxd init`). Same kernel tech, different tools:

| | **LXC** (this course) | **LXD** |
|---|---|---|
| Install | `sudo apt install lxc` | `sudo snap install lxd` then `sudo lxd init` |
| Commands | `lxc-create`, `lxc-start`, `lxc-ls` … (with a **dash**) | `lxc launch`, `lxc list`, `lxc exec` (**space**, no dash) |
| Bridge | `lxcbr0` | `lxdbr0` |
| Containers stored | `/var/lib/lxc/` | LXD storage pool (ZFS etc.) |

They don't see each other's containers, but both can be installed together.

LXD equivalents, for reference:

```bash
lxc launch ubuntu:24.04 newpc               # create + start
lxc list                                    # list with IPs
lxc exec newpc -- bash                      # shell inside
lxc snapshot newpc snap0                    # snapshot
lxc restore newpc snap0                     # roll back
lxc delete newpc --force                    # destroy
```

### Cheat summary

```bash
sudo lxc-create -t ubuntu -n NAME           # create
sudo lxc-ls --fancy                         # list + IPs
sudo lxc-start -n NAME -d                   # start
sudo lxc-attach -n NAME                     # shell in (or: ssh ubuntu@IP)
sudo lxc-stop -n NAME                       # stop
sudo lxc-destroy -n NAME                    # delete
```