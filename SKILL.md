---
name: linux-basics
description: Core working knowledge for running commands on Linux systems — identifying the distro and environment, privileges and sudo, package managers, services (systemd), filesystem layout, temp/build locations, processes, permissions and safe shell habits. Debian-first, but covers other distros. Load this skill whenever you are about to execute shell commands, scripts, builds, installs or system inspection on a Linux host, even for tasks that seem simple or when the user never mentions "Linux" — if the platform is linux and you are going to run something, this applies.
---

# Linux Basics

Practical knowledge for an agent running commands on a Linux machine. Most of what
follows exists because agents keep getting it wrong: they assume a system instead of
checking it, start commands that wait for input that never comes, or have side effects
nobody asked for. Debian is the main target; differences on other distros are noted
where they matter.

Your shell is non-interactive. Nobody can answer a password prompt, scroll a pager or
close an editor, so any command that waits for a human just hangs until it times out.
Keep that in mind throughout.

## 1. Orient yourself first

Don't guess the distro from the presence of `apt` or from habit. One cheap probe
saves a wrong command:

```sh
. /etc/os-release; echo "$ID $VERSION_ID ($ID_LIKE)"   # debian 13, ubuntu 24.04, fedora 41 ...
uname -r                                              # kernel
systemd-detect-virt 2>/dev/null                       # none, kvm, docker, wsl ...
command -v systemctl apt-get dnf pacman apk           # what actually exists
id                                                    # who am I, which groups
```

- Reading the Debian version: `/etc/debian_version` holds a number (`13.2`) on a released stable version. If it holds a codename instead, the system runs a version that hasn't been released yet. If it contains `/sid` (e.g. `forky/sid`), the system is Debian **testing or unstable**: the file is identical on both, and `VERSION_ID` is missing from `/etc/os-release`. Tell testing and unstable apart by the configured Debian archives: `apt-cache policy | grep -oE 'o=Debian,a=[a-z-]+' | sort -u` (no root needed; it ignores third-party repos). Developer machines often enable several archives at once (stable, testing, unstable, experimental), with pinning deciding what gets installed. In that case don't pick one; say it is mixed. Report it as "Debian testing/unstable (forky/sid)" rather than guessing a release number. Packages and behaviour there can be newer than any documentation you know, so check versions instead of assuming.
- Ubuntu is Debian-like but not Debian: it has different package names, snaps, and PPAs. Treat it as its own system.
- In containers there is usually no systemd, no `sudo`, and only a minimal set of tools. Check before using them.
- Check for a tool with `command -v tool`, not `which` (which isn't always installed and behaves differently between implementations).

## 2. Privileges

- **Never become root without explicit permission from the user.** That covers `sudo` (including "harmless" probes like `sudo -n true`), `su`, `doas`, `pkexec`, `run0`, `machinectl shell root@`, and any setuid helper or trick that gets the same effect. This includes `sudo` tucked into a larger diagnostic one-liner "just to check": that is still running sudo. Root can damage the whole system and bypasses every safeguard the user relies on, so the user decides when it's used, not you. Permission applies to the specific task it was given for. It is not a standing licence.
- When a task seems to need root, stop and say so: name the exact command, explain why it needs root, and offer alternatives that work without it (a user-local install, a venv, a container, a different port or path). If the user agrees and a password is needed, use the harness's secret mechanism if there is one. Never type or store the password in the conversation.
- Most work doesn't need root anyway: user-local installs, files you own, `git`, builds. Running a build as root leaves root-owned files the user can't delete later.
- Once you have permission: `sudo echo x > /etc/file` fails because the redirection happens in your unprivileged shell. Use `echo x | sudo tee /etc/file >/dev/null` (add `-a` to append). A plain `sudo` that asks for a password hangs a non-interactive shell.
- On Debian, `/usr/sbin` and `/sbin` are often **not** in a normal user's `PATH`. When `adduser`, `iptables`, `modprobe` or `ldconfig` come back "not found", they are usually installed. Many of them run fine as a normal user with the full path (`/usr/sbin/...`) for read-only queries. If one genuinely needs root, ask first.
- If you are already root (containers, CI), that is the environment you were given, not something you escalated to. `sudo` may not exist there. Drop it rather than installing it. Still be careful: you have no safety net.

## 3. Packages

| Family | Install | Query installed | Which package owns a file | Search |
|---|---|---|---|---|
| Debian/Ubuntu | `apt-get install -y pkg` | `dpkg -l pkg` | `dpkg -S /path` | `apt-cache search`, `apt-file search` |
| Fedora/RHEL | `dnf install -y pkg` | `rpm -q pkg` | `rpm -qf /path` | `dnf provides '*/file'` |
| Arch | `pacman -S --noconfirm pkg` | `pacman -Q pkg` | `pacman -Qo /path` | `pacman -F file` |
| Alpine | `apk add pkg` | `apk info -e pkg` | `apk info --who-owns /path` | `apk search` |

Debian specifics:
- Use `apt-get`/`apt-cache` in scripts and automation. `apt` is meant for humans, its output format isn't stable, and it warns when you pipe it.
- Set `DEBIAN_FRONTEND=noninteractive` when installing, or debconf may stop and wait for an answer to a configuration question.
- Run `apt-get update` before the first install on a fresh system or container. Otherwise you get "Unable to locate package".
- Installing system packages needs root, so the rule in section 2 applies: ask first, and list the exact packages. Checking what is installed (`dpkg -l`, `apt-cache policy`) needs no root and is always fine.
- Package names follow conventions: `python3-foo`, `libfoo-dev` for headers, `build-essential` for a compiler toolchain. Some software only exists in the `contrib`/`non-free` components or in backports.
- `pip install` into the system Python fails with `externally-managed-environment` (PEP 668). Don't use `--break-system-packages`. Use `apt-get install python3-foo`, a venv (`python3 -m venv`), `pipx`, or `uv`.
- When a command is missing, find the package that provides it: `apt-file search bin/foo` (needs `apt-file` and `apt-file update`), or the `command-not-found` hint if it's installed.

## 4. Services and logs

- With systemd: `systemctl status unit --no-pager`, `journalctl -u unit -b --no-pager -n 100`. Without `--no-pager` these commands open a pager and hang.
- User services run under `systemctl --user` and `journalctl --user`, and are separate from the system ones.
- Containers and WSL often run without systemd (PID 1 isn't `systemd`, check with `ps -p 1 -o comm=`). There, `systemctl` fails. Run the daemon directly, or use the container's own mechanism.
- Starting, stopping, enabling or restarting system services affects the whole machine and needs root, so ask first (section 2). Reading status and logs usually doesn't need root. Don't reach for `sudo journalctl`.

### Who can read which logs

Check what your user can read **before** concluding that logs are missing:

```sh
id -nG                                   # look for adm, systemd-journal (or wheel on Fedora/Arch)
id -u                                    # your UID
grep -E '^(SYS_)?UID_(MIN|MAX)' /etc/login.defs   # regular-user UID range, usually 1000-60000
```

- **Full system journal** (all services, kernel, other users): you need to be root or in the `systemd-journal` group. On Debian, `adm` also works because the journal directory has an ACL granting `adm` read access (`getfacl /var/log/journal/*`). Fedora/Arch usually grant it to `wheel` instead.
- **Without those groups**, you only see your own user journal. journald writes a separate `user-<UID>.journal` file (made readable to you through an ACL) only for UIDs in the regular-user range (`UID_MIN`–`UID_MAX`, usually ≥1000), so you can read that file. Messages from system services and system UIDs (<1000) go into the system journal, which you can't read.
- `journalctl` doesn't fail when you lack access. It prints a hint like "you are currently not seeing messages from other users and the system" (easy to miss) and shows only what you may read. Empty or suspiciously thin output for a system unit usually means **no permission**, not **no logs**. Say that to the user rather than reporting "no errors found".
- The user journal also needs persistent or split storage: with `Storage=volatile` or `SplitMode=none` in `/etc/systemd/journald.conf` (or a drop-in in `journald.conf.d/`), there may be no per-user file at all.
- Classic text logs in `/var/log` (`syslog`, `auth.log`, `kern.log`, if rsyslog is installed) are usually group `adm` on Debian, so the same group membership applies. Newer Debian installs may not have rsyslog at all, so there is no `/var/log/syslog` and the journal is the only source.
- If you can't read what you need, tell the user which group is missing and suggest they run the command themselves or grant access (adding a user to `adm` needs root and a fresh login). Don't escalate on your own.

## 5. Filesystem layout and locations

- Temp, scratch and build folders go in `/tmp` or `/var/tmp` (use `mktemp -d /tmp/name.XXXXXX`), **never** under `$HOME`. The home directory is the user's space, and leftover build trees there are clutter they didn't ask for.
- On Debian 13 (trixie) and later, `/tmp` is a tmpfs by default: it lives in RAM (and swap) and is wiped on reboot. Check with `findmnt -n -o FSTYPE,SIZE,AVAIL /tmp`. If it is a tmpfs, don't overfill it. Whatever you write there takes memory away from every process on the machine. A full `/tmp` makes unrelated programs fail and can push the system into swapping or the OOM killer. Before you put anything large in `/tmp` (builds, source trees, downloads, extracted archives, test data), compare its expected size with `AVAIL` and with `free -h`, and leave plenty of headroom. When in doubt, or when it will be more than a few hundred MB, use `/var/tmp` (on disk) instead. Also put anything that must survive a reboot in `/var/tmp`. Delete large temp files as soon as you no longer need them.
- Neither location is permanent storage. systemd-tmpfiles cleans **both** `/tmp` and `/var/tmp` on a timer, deleting files that haven't been used for a set age. The defaults are in `/usr/lib/tmpfiles.d/tmp.conf` (`/lib/tmpfiles.d/tmp.conf` on older systems), usually 10 days for `/tmp` and 30 days for `/var/tmp`. An admin can override them with a file of the same name in `/etc/tmpfiles.d/`, so check both if the lifetime matters:
  ```sh
  grep -h '^[^#]' /usr/lib/tmpfiles.d/tmp.conf /etc/tmpfiles.d/tmp.conf 2>/dev/null
  ```
  Results the user wants to keep go to a durable location they choose, not a temp folder.
- Clean up temp folders you created when you're done, unless the user wants the results.
- Config lives in `/etc` (system-wide) and `~/.config` (per user, `$XDG_CONFIG_HOME`). On Debian, prefer drop-in files (`/etc/foo.d/*.conf`, `systemctl edit`) over editing files the package ships: package upgrades may overwrite those or ask about them interactively.
- Before overwriting or deleting a path, look at it (`ls -la`, `file`, `head`).

## 6. Processes, resources, networking

- Use `top -b -n1`, not `top`. `htop` and similar tools are interactive and hang.
- Stop a process with `kill PID` (SIGTERM) first, and use `kill -9` only if it ignores that. `pkill name` or `killall name` can hit other people's processes. Prefer `pgrep -a` to see what matches before killing.
- Resources: `nproc`, `free -h`, `df -h`, `du -sh dir`. Check free space before large builds or downloads.
- Ports and sockets: `ss -ltnp`. Process names are only shown as root. `netstat` is deprecated and often missing.
- Network config: `ip addr`, `ip route`. `ifconfig` and `route` are deprecated and often missing.
- Long-running jobs should use the harness's background mechanism rather than a bare `&`, so that their output and exit status aren't lost.

## 7. Safe shell habits

- **Don't trigger pagers or editors.** Use `git --no-pager ...` or `GIT_PAGER=cat`, `PAGER=cat`, `SYSTEMD_PAGER=cat`, `--no-pager`. Set `GIT_EDITOR=true` for commands that would open an editor (`git commit` without `-m`, `git rebase --continue`).
- **`/bin/sh` is `dash` on Debian/Ubuntu**, not bash. Scripts with `#!/bin/sh` must avoid bashisms (`[[ ]]`, arrays, `source`, `function`, `{1..5}`). Either write POSIX sh or use `#!/bin/bash`.
- In bash scripts, use `set -euo pipefail`. Quote every expansion: `"$var"`, `"$@"`.
- Guard destructive commands: `rm -rf "${dir:?}/"` fails instead of deleting `/` when `dir` is empty. Never `rm -rf` a path built from unchecked input.
- When parsing command output, set `LC_ALL=C`. Otherwise localized messages, number formats and sort order break your parsing.
- Linux userland is GNU (`sed -i`, `grep -P`, `date -d` work). Alpine and minimal containers use BusyBox, where many GNU options are missing. macOS/BSD habits like `sed -i ''` break on GNU.
- File permissions: don't fix permission problems with `chmod 777` or `chmod -R`. Find out who should own the file and why access is denied (`ls -l`, `id`, `namei -l /path`).

## Reference files

- `references/`: deeper per-topic material (none yet). Read a file only when its topic comes up.
- `scripts/`: helper scripts, e.g. a system-orientation probe (none yet).
