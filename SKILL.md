---
name: linux-basics
description: Core working knowledge for running commands on Linux systems — identifying the distro and environment, privileges and sudo, package managers, services (systemd), filesystem layout, temp/build locations, processes, permissions and safe shell habits. Load this skill whenever you are about to execute shell commands, scripts, builds, installs or system inspection on a Linux host, even for tasks that seem simple or when the user never mentions "Linux" — if the platform is linux and you are going to run something, this applies.
---

# Linux Basics

<!-- DRAFT SKELETON: section headings are placeholders; content to be filled in. -->

Practical knowledge for an agent executing things on a Linux machine. The goal is to
act like an experienced admin: find out what system you are on before assuming,
prefer the native tooling of that system, and avoid surprising or destructive side effects.

## 1. Orient yourself first

TODO: distro / version detection (`/etc/os-release`), kernel (`uname -r`), init system,
container/VM/WSL detection, shell in use, available tools (`command -v`).

## 2. Privileges

TODO: running as root vs. user, `sudo -n` checks, never assume passwordless sudo,
user/group membership.

## 3. Packages

TODO: map distro family → package manager (apt/dpkg, dnf/rpm, pacman, zypper, apk),
querying vs. installing, which package provides a file.

## 4. Services and logs

TODO: systemd (`systemctl`, `journalctl`), user units, non-systemd fallbacks.

## 5. Filesystem layout and locations

TODO: FHS basics, XDG dirs, where temp/build files belong (`/tmp`, `/var/tmp`, `mktemp -d`
— never under `$HOME`), config locations.

## 6. Processes, resources, networking

TODO: ps/pgrep/kill, resource checks (df, free, nproc), ports (`ss`), background jobs.

## 7. Safe shell habits

TODO: quoting, `set -euo pipefail` in scripts, look before `rm -rf`/overwrite,
GNU vs. BSD tool differences.

## Reference files

- `references/` — deeper per-topic material (to be added; read only when the topic comes up).
- `scripts/` — helper scripts (e.g. a system-orientation probe), to be added.
