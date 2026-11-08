# Linux

A personal, growing set of Linux notes, cheatsheets and scripts — built up over time
from first principles (kernel and boot) through day-to-day usage and into security and
pentesting. The writing is deliberately my own; this index just maps where things live.

## Map

### 01 · Fundamentals
Start here — how Linux actually works, from the hardware up.
- [linux-internals.md](01-fundamentals/linux-internals.md) — kernel/user space, device files, the shell, C, the boot procedure, init/systemd, logging and storage (MBR/GPT/LVM).
- [fundamentals.md](01-fundamentals/fundamentals.md) — core concepts and commands.
- [cheatsheet.md](01-fundamentals/cheatsheet.md) — quick command reference.
- [under-the-hood.md](01-fundamentals/under-the-hood.md) — short deep-dive notes.

### 02 · Security
- [security-in-linux.md](02-security/security-in-linux.md) — hardening, permissions and security concepts.

### 03 · Pentesting
Field references from hands-on work.
- [hacking-snippets-2025.md](03-pentesting/hacking-snippets-2025.md) — large snippet collection.
- [mega-cheatsheet.md](03-pentesting/mega-cheatsheet.md) — combined cheatsheet.

### Snippets
- [useful-aliases.md](snippets/useful-aliases.md) — shell aliases worth keeping.
- [snippets-tbc.md](snippets/snippets-tbc.md) — snippets still to be sorted.

## Scripts
My own shell scripts, grouped by purpose.
- [scripts/recon/](scripts/recon/) — `nmapRecon.sh`, `WebRecon.sh`.
- [scripts/security/](scripts/security/) — `searchCodeForSecrets.sh` (semgrep secret scan).
- [scripts/utils/](scripts/utils/) — `docker-clean.sh`, `Functions.sh` (includes an `extract` helper).
- [scripts/templates/](scripts/templates/) — `Boilerplate-Shell-Script-Template.sh` to start new scripts from.

## Tools
Third-party tools kept separate from my own writing (they carry their own licences).
- [tools/LinEnum/](tools/LinEnum/) — the LinEnum local privilege-escalation enumeration script.

## scratch/
Work-in-progress and test scripts, parked here rather than deleted.
