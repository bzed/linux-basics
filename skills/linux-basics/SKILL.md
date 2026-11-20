---
name: linux-basics
description: Core working knowledge for running commands on Linux systems — identifying the distro and environment, privileges and sudo, package managers, services (systemd), filesystem layout, temp/build locations, CPU and memory limits for builds and parallel jobs, processes, podman containers, permissions, safe shell scripting (strict mode, quoting, mktemp/trap, shellcheck) and searching with ripgrep. Debian-first, but covers other distros. Load this skill whenever you are about to execute shell commands, scripts, builds, installs or system inspection on a Linux host, even for tasks that seem simple or when the user never mentions "Linux" — if the platform is linux and you are going to run something, this applies.
license: MIT
---

# Linux Basics

Practical knowledge for an agent running commands on a Linux machine. Most of what follows
exists because agents keep getting it wrong: they assume a system instead of checking it, start
commands that wait for input that never comes, or have side effects nobody asked for. Debian is
the main target; differences on other distros are noted where they matter.

This file holds the rules. The details live in `references/`. Read a reference file when its
topic comes up; each section below says which one.

Your shell is non-interactive. Nobody can answer a password prompt, scroll a pager or close an
editor, so any command that waits for a human just hangs until it times out.

## 1. Never become root without permission

- **Never become root without explicit permission from the user.** That covers `sudo`
  (including "harmless" probes like `sudo -n true`), `su`, `doas`, `pkexec`, `run0`,
  `machinectl shell root@`, and any setuid helper or trick that gets the same effect. This
  includes `sudo` tucked into a larger diagnostic one-liner "just to check": that is still
  running sudo. Root can damage the whole system and bypasses every safeguard the user relies on,
  so the user decides when it's used, not you. Permission applies to the specific task it was
  given for. It is not a standing licence.
- When a task seems to need root, stop and say so: name the exact command, explain why it needs
  root, and offer alternatives that work without it (a user-local install, a venv, a container, a
  different port or path). If the user agrees and a password is needed, use the harness's secret
  mechanism if there is one. Never type or store the password in the conversation.
- Installing system packages and starting, stopping or restarting system services all need root,
  so they fall under this rule. Querying packages, service status and logs usually does not.
- Most work doesn't need root anyway: user-local installs, files you own, `git`, builds. Running a
  build as root leaves root-owned files the user can't delete later.
- Once you have permission: `sudo echo x > /etc/file` fails because the redirection happens in
  your unprivileged shell. Use `echo x | sudo tee /etc/file >/dev/null`.
- If you are already root (containers, CI), that is the environment you were given, not
  something you escalated to. Drop `sudo` rather than installing it, and be careful: you have no
  safety net.

## 2. Orient yourself first

Don't guess the system from habit. One cheap probe saves a wrong command:

```sh
. /etc/os-release; echo "$ID $VERSION_ID"; cat /etc/debian_version 2>/dev/null
ps -p $$ -o comm=; getent passwd "$(id -un)" | cut -d: -f7   # current shell, login shell
id                                                          # user, groups (adm? systemd-journal?)
```

- **Debian version:** a number in `/etc/debian_version` means a released version. A codename
  means a version that hasn't been released yet, and `…/sid` (e.g. `forky/sid`) means testing or
  unstable. Never guess a release number. → `references/packages.md`
- **Shell:** the harness often runs your commands in the user's login shell, not bash. If it
  isn't bash, wrap anything beyond a simple command in `bash -c '...'`, or write for that
  shell. zsh in particular treats unquoted patterns like `'[f]oo'` or `'*.c'` as file globs and
  errors out, so quote every pattern meant for a program. → `references/shells.md`
- Check for tools with `command -v tool`, not `which`.

## 3. Packages

- Use `apt-get`/`apt-cache` in scripts, not `apt`, and set `DEBIAN_FRONTEND=noninteractive` when
  installing (with permission, see section 1).
- Never `pip install --break-system-packages`. Check whether the module is already installed,
  then use `python3-foo` from apt, a venv, `pipx` or `uv`.
- Other distros, finding which package provides a file, `/usr/sbin` not being in `PATH` on
  Debian. → `references/packages.md`

## 4. Services and logs

- Always pass `--no-pager` to `systemctl` and `journalctl`.
- Before saying "no errors in the logs", check that you can read them: the system journal needs
  root or the `systemd-journal`/`adm` group. Without them, `journalctl` silently shows only your
  own user journal. Thin output usually means "no permission", not "no logs".
  → `references/logs.md`

## 5. Temp and build files

- Temp, scratch and build folders go in `/tmp` or `/var/tmp` (`mktemp -d /tmp/name.XXXXXX`),
  **never** under `$HOME`. Clean them up when you're done.
- `/tmp` is often a tmpfs (in RAM; the default since Debian 13). Don't overfill it: check
  `findmnt -n -o FSTYPE,SIZE,AVAIL /tmp` before writing anything large, and use `/var/tmp` for
  big builds or anything over a few hundred MB.
- Both `/tmp` and `/var/tmp` are cleaned automatically by systemd-tmpfiles. Results to keep go
  somewhere durable. → `references/filesystem.md`

## 6. CPU and memory

- **Never use all CPU cores unless the user explicitly granted it** (e.g. on a development-only
  VM). Using every core makes a desktop system unusable. Default to at most half of `nproc`:
  `make -j"$(( $(nproc) / 2 ))"`, never a bare `make -j`, and never `-j"$(nproc)"`.
- Watch for tools that grab every core by default: `cargo`, `go`, `ninja`, `xz -T0`,
  `pytest -n auto`, `xargs -P 0`. Give them an explicit job count.
- Keep memory in check too: parallel jobs × memory per job must stay well below `available` in
  `free -h`. Don't run several heavy jobs at once.
- Run heavy jobs with `nice -n 19 ionice -c 3`. For a hard cap without root, use
  `systemd-run --user --scope -p CPUQuota=… -p MemoryMax=…`. → `references/resources.md`

## 7. Processes

- **`pgrep -f foo` matches your own wrapper shell**, whose command line contains `foo`, so it
  reports "running" even when nothing is. Prefer tracking the PID (`kill -0 "$pid"`), matching
  by name (`pgrep -x foo`, no `-f`), `pgrep -A -af` (newer procps-ng), or the quoted bracket
  trick `pgrep -af '[f]oo'`.
- Run `pgrep -af pattern` before any `pkill -f pattern`. Use `kill` (SIGTERM) before `kill -9`.
- Use non-interactive forms: `top -b -n1`, not `top`/`htop`. → `references/processes.md`

## 8. Safe shell habits

- **Don't trigger pagers or editors.** Use `git --no-pager` or `GIT_PAGER=cat`, `PAGER=cat`,
  `SYSTEMD_PAGER=cat`, `--no-pager`. Set `GIT_EDITOR=true` for commands that would open an editor.
- `/bin/sh` is `dash` on Debian/Ubuntu. `#!/bin/sh` scripts must be POSIX. Otherwise use
  `#!/bin/bash`. → `references/shells.md`
- **Writing scripts:** start bash scripts with `set -euf -o pipefail` (know its side effects:
  `-f` disables globbing). Quote every expansion (`"$var"`, `"$@"`) and use `--` before
  file-name arguments. Create temp files with `mktemp` and remove them with
  `trap 'rm -rf -- "$tmpdir"' EXIT`. Avoid `eval`; use arrays to build commands. Make sure
  non-POSIX features work in the shell that runs the code. Run `shellcheck` on every script
  before using it. → `references/shell-scripting.md`
- Guard destructive commands: `rm -rf "${dir:?}/"` fails instead of deleting `/` when `dir` is
  empty. Look at a path before overwriting or deleting it.
- When parsing command output, set `LC_ALL=C`.
- Linux userland is GNU. Alpine and minimal containers use BusyBox, where many GNU options are
  missing. macOS/BSD habits like `sed -i ''` break on GNU.
- Don't fix permission problems with `chmod 777` or `chmod -R`. Find out why access is denied
  (`ls -l`, `id`, `namei -l /path`).

## 9. Containers (podman)

- **Mount only what's needed.** Never mount `$HOME`, `/` or system directories. Mount the
  specific project or data directory, read-only (`:ro`) unless the container must write, and
  never the podman/docker socket or credential directories (`~/.ssh`, …).
- **Limit the container.** Set CPU, memory and process limits (`--cpus`, `--memory`,
  `--pids-limit`, `--timeout`), drop capabilities, and never use `--privileged`.
- **Limit networking to what's needed.** `--network=none` when no network is needed, an
  `--internal` network for container-to-container traffic, and published ports bound to
  `127.0.0.1` only. Avoid `--network=host`.
- **Leave nothing behind.** Use `--rm`, label what you create (`--label agent-task=…`), and
  delete your containers, networks, volumes and images when done. Never use `prune` or
  `system reset`: they delete the user's things too.
- Use rootless podman. `sudo podman` needs permission (section 1). → `references/containers.md`

## 10. Searching with `rg`

- `rg` silently **skips** files matched by `.gitignore`/`.ignore`, hidden files and
  directories (`.github/`, `.env`), binary files and symlinks. "No matches" isn't "not there":
  before concluding something doesn't exist, repeat with `-uu` (`--no-ignore --hidden`) or
  `-uuu` (also binary), or name the path explicitly.
- Always give a path (`rg PATTERN .`): without one, rg may search stdin instead and hang.
- Use `-F` for literal strings, `-t TYPE`/`-g GLOB` to narrow, single-quote patterns and globs.
- Add `--color=never`: if the harness runs commands in a pty or a user config sets
  `--color=always`, ANSI escape codes end up in the output and break parsing and edits.
- For processing results, use `--json` (JSON Lines with `begin`/`match`/`end`/`summary`
  messages). Exit status `1` = no match, `2` = error. → `references/ripgrep.md`

## Reference files

| File | Read it when |
| --- | --- |
| `references/packages.md` | detecting distro/release, installing or querying packages, Python packages |
| `references/shells.md` | the shell isn't bash, or you write `#!/bin/sh` scripts |
| `references/logs.md` | inspecting services or reading logs |
| `references/filesystem.md` | creating temp/build dirs, writing large data, editing config |
| `references/resources.md` | builds, tests, compression or anything parallel or memory-hungry |
| `references/processes.md` | checking whether something runs, `pgrep`/`pkill`/`ps`, ports |
| `references/containers.md` | running anything with `podman` (or `docker`) |
| `references/shell-scripting.md` | writing any shell script or non-trivial one-liner |
| `references/ripgrep.md` | searching with `rg`, or when a search finds nothing |
