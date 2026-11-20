# Filesystem: temp and build locations, config files

Read this before creating temp or build directories, writing anything large, or editing
configuration.

## Contents
- Where temp and build files go
- tmpfs `/tmp`: don't overfill it
- Automatic cleanup of `/tmp` and `/var/tmp`
- Config files

## Where temp and build files go

- Temp, scratch and build folders go in `/tmp` or `/var/tmp` (use `mktemp -d /tmp/name.XXXXXX`),
  **never** under `$HOME`. The home directory is the user's space, and leftover build trees there
  are clutter they didn't ask for.
- Clean up temp folders you created when you're done, unless the user wants the results.
- Results the user wants to keep go to a durable location they choose, not a temp folder.
- Before overwriting or deleting a path, look at it (`ls -la`, `file`, `head`).

## tmpfs `/tmp`: don't overfill it

On Debian 13 (trixie) and later, `/tmp` is a tmpfs by default: it lives in RAM (and swap) and is
wiped on reboot. Check with:

```sh
findmnt -n -o FSTYPE,SIZE,AVAIL /tmp
free -h
```

If it is a tmpfs, whatever you write there takes memory away from every process on the machine.
A full `/tmp` makes unrelated programs fail and can push the system into swapping or the OOM
killer.

- Before you put anything large in `/tmp` (builds, source trees, downloads, extracted archives,
  test data), compare its expected size with `AVAIL` and with `free -h`, and leave plenty of
  headroom.
- When in doubt, or when it will be more than a few hundred MB, use `/var/tmp` (on disk)
  instead. Also put anything that must survive a reboot in `/var/tmp`.
- Delete large temp files as soon as you no longer need them.

## Automatic cleanup of `/tmp` and `/var/tmp`

Neither location is permanent storage. systemd-tmpfiles cleans **both** on a timer, deleting files
that haven't been used for a set age. The defaults are in `/usr/lib/tmpfiles.d/tmp.conf`
(`/lib/tmpfiles.d/tmp.conf` on older systems), usually 10 days for `/tmp` and 30 days for
`/var/tmp`. An admin can override them with a file of the same name in `/etc/tmpfiles.d/`, so
check both if the lifetime matters:

```sh
grep -h '^[^#]' /usr/lib/tmpfiles.d/tmp.conf /etc/tmpfiles.d/tmp.conf 2>/dev/null
```

## Config files

Config lives in `/etc` (system-wide) and `~/.config` (per user, `$XDG_CONFIG_HOME`). On Debian,
prefer drop-in files (`/etc/foo.d/*.conf`, `systemctl edit`) over editing files the package
ships: package upgrades may overwrite those or ask about them interactively.
