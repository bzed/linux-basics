# Services and logs: systemd, journal access, classic log files

Read this when you inspect services or read logs, and especially before concluding that
"there are no errors".

## Contents
- systemctl and journalctl basics
- Who can read which logs
- Classic text logs
- When access is missing

## systemctl and journalctl basics

- `systemctl status unit --no-pager`, `journalctl -u unit -b --no-pager -n 100`. Without
  `--no-pager` these commands open a pager and hang.
- User services run under `systemctl --user` and `journalctl --user`, and are separate from the
  system ones.
- Containers and WSL often run without systemd (PID 1 isn't `systemd`, check with
  `ps -p 1 -o comm=`). There, `systemctl` fails. Run the daemon directly, or use the container's
  own mechanism.
- Starting, stopping, enabling or restarting system services affects the whole machine and needs
  root, so ask first. Reading status and logs usually doesn't need root. Don't reach for
  `sudo journalctl`.

## Who can read which logs

Check what your user can read **before** concluding that logs are missing:

```sh
id -nG                                   # look for adm, systemd-journal (or wheel on Fedora/Arch)
id -u                                    # your UID
grep -E '^(SYS_)?UID_(MIN|MAX)' /etc/login.defs   # regular-user UID range, usually 1000-60000
```

- **Full system journal** (all services, kernel, other users): you need to be root or in the
  `systemd-journal` group. On Debian, `adm` also works because the journal directory has an ACL
  granting `adm` read access (`getfacl /var/log/journal/*`). Fedora/Arch usually grant it to
  `wheel` instead.
- **Without those groups**, you only see your own user journal. journald writes a separate
  `user-<UID>.journal` file (made readable to you through an ACL) only for UIDs in the
  regular-user range (`UID_MIN`–`UID_MAX`, usually ≥1000), so you can read that file. Messages
  from system services and system UIDs (<1000) go into the system journal, which you can't read.
- `journalctl` doesn't fail when you lack access. It prints a hint like "you are currently not
  seeing messages from other users and the system" (easy to miss) and shows only what you may
  read. Empty or suspiciously thin output for a system unit usually means **no permission**, not
  **no logs**. Say that to the user rather than reporting "no errors found".
- The user journal also needs persistent or split storage: with `Storage=volatile` or
  `SplitMode=none` in `/etc/systemd/journald.conf` (or a drop-in in `journald.conf.d/`), there
  may be no per-user file at all.

## Classic text logs

Classic text logs in `/var/log` (`syslog`, `auth.log`, `kern.log`, if rsyslog is installed) are
usually group `adm` on Debian, so the same group membership applies. Newer Debian installs may
not have rsyslog at all, so there is no `/var/log/syslog` and the journal is the only source.

## When access is missing

Tell the user which group is missing and suggest they run the command themselves or grant
access (adding a user to `adm` needs root and a fresh login). Don't escalate on your own.
