# Distros and packages: detecting the system, apt and the others

Read this before installing or querying packages, or when you need to know exactly which
distribution and release you are on.

## Contents

- Detecting the distribution and release
- Package manager equivalents
- Debian/apt specifics
- Python packages (PEP 668)

## Detecting the distribution and release

```sh
. /etc/os-release; echo "$ID $VERSION_ID ($ID_LIKE)"   # debian 13, ubuntu 24.04, fedora 41 ...
uname -r                                              # kernel
systemd-detect-virt 2>/dev/null                       # none, kvm, docker, wsl ...
command -v systemctl apt-get dnf pacman apk           # what actually exists
```

- Reading the Debian version: `/etc/debian_version` holds a number (`13.2`) on a released stable
  version. If it holds a codename instead, the system runs a version that hasn't been released
  yet. If it contains `/sid` (e.g. `forky/sid`), the system is Debian **testing or unstable**: the
  file is identical on both, and `VERSION_ID` is missing from `/etc/os-release`.
- Tell testing and unstable apart by the configured Debian archives:
  `apt-cache policy | grep -oE 'o=Debian,a=[a-z-]+' | sort -u` (no root needed; it ignores
  third-party repos). Developer machines often enable several archives at once (stable, testing,
  unstable, experimental), with pinning deciding what gets installed. In that case don't pick
  one; say it is mixed. Report it as "Debian testing/unstable (forky/sid)" rather than guessing a
  release number.
- On testing/unstable, packages and behaviour can be newer than any documentation you know, so
  check versions instead of assuming.
- Ubuntu is Debian-like but not Debian: it has different package names, snaps, and PPAs. Treat it
  as its own system.
- In containers there is usually no systemd, no `sudo`, and only a minimal set of tools. Check
  before using them.

## Package manager equivalents

| Family | Install | Query installed | Which package owns a file | Search |
|---|---|---|---|---|
| Debian/Ubuntu | `apt-get install -y pkg` | `dpkg -l pkg` | `dpkg -S /path` | `apt-cache search`, `apt-file search` |
| Fedora/RHEL | `dnf install -y pkg` | `rpm -q pkg` | `rpm -qf /path` | `dnf provides '*/file'` |
| Arch | `pacman -S --noconfirm pkg` | `pacman -Q pkg` | `pacman -Qo /path` | `pacman -F file` |
| Alpine | `apk add pkg` | `apk info -e pkg` | `apk info --who-owns /path` | `apk search` |

Installing needs root: ask the user first and list the exact packages. Querying needs no root.

## Debian/apt specifics

- Use `apt-get`/`apt-cache` in scripts and automation. `apt` is meant for humans, its output
  format isn't stable, and it warns when you pipe it.
- Set `DEBIAN_FRONTEND=noninteractive` when installing, or debconf may stop and wait for an
  answer to a configuration question.
- Run `apt-get update` before the first install on a fresh system or container. Otherwise you get
  "Unable to locate package".
- Package names follow conventions: `python3-foo`, `libfoo-dev` for headers, `build-essential`
  for a compiler toolchain. Some software only exists in the `contrib`/`non-free` components or in
  backports.
- When a command is missing, find the package that provides it: `apt-file search bin/foo` (needs
  `apt-file` and `apt-file update`), or the `command-not-found` hint if it's installed.
- On Debian, `/usr/sbin` and `/sbin` are often **not** in a normal user's `PATH`. When `adduser`,
  `iptables`, `modprobe` or `ldconfig` come back "not found", they are usually installed. Many of
  them run fine as a normal user with the full path (`/usr/sbin/...`) for read-only queries.

## Python packages (PEP 668)

`pip install` into the system Python fails with `externally-managed-environment`. Don't use
`--break-system-packages`. Check first whether the module is already there
(`python3 -c 'import foo'`), then use `apt-get install python3-foo` (with permission), a venv
(`python3 -m venv`), `pipx`, or `uv`.
