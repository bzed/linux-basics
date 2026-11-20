# Processes: checking, finding and stopping them

Read this before using `pgrep`, `pkill`, `ps | grep` or `killall`, or whenever you need to know
whether something is still running.

## Contents

- The self-match trap with `pgrep -f`
- How pgrep matches
- Ways to avoid the self-match
- Stopping processes
- Resources, ports and network

## The self-match trap with `pgrep -f`

The classic agent bug: "is my server still running?" → `pgrep -f myserver` → always "yes",
even after the server died.

`pgrep` itself never matches itself (procps-ng, see `man pgrep` → NOTES). But your command
doesn't run alone. The harness wraps it in a shell, e.g. `bash -c '... pgrep -f myserver ...'`
or `zsh -c "... eval '...pgrep -f myserver...'"`. With `-f`, pgrep matches against full command
lines, and that **wrapper shell's command line contains your pattern**, so it matches its own
parent and reports success. The same goes for `ps aux | grep myserver`, where `grep` matches
itself.

## How pgrep matches

From `man pgrep`, procps-ng 4.x:

- The pattern is an **extended regular expression**, matched as a substring (unanchored) unless
  you pass `-x`.
- Without `-f`, it matches only the **process name**, which is truncated to **15 characters**
  (`/proc/PID/stat`). `pgrep my-long-server-name` never matches; use `-f` or the first 15
  characters.
- With `-f`, it matches the **full command line** (`/proc/PID/cmdline`), including arguments.
  That's where the self-match trap is.
- Exit status: `0` = something matched, `1` = nothing matched, `2` = syntax error, `3` = fatal
  error. Don't treat every non-zero status as "not running": `2` means your command was wrong.
- Zombie (defunct) processes are reported too. "Matched" doesn't guarantee "working".
- Useful options: `-a` (show the command line, always use it while checking what you'll match),
  `-x` (exact match), `-u user` (only that user's processes), `-n`/`-o` (newest/oldest), `-c`
  (count), `-P ppid` (children of a process), `--quiet` (no output, only the exit status).

## Ways to avoid the self-match, best first

1. **Track the PID instead of searching for a name.** When you start the process yourself,
   record its PID (`$!`, or your harness's background-task ID) and check that specific process:
   `kill -0 "$pid" 2>/dev/null && echo running`. `kill -0` sends no signal. It only tests whether
   the process exists and you may signal it. `ps -p "$pid" -o pid=,stat=,cmd=` also shows whether
   it's a zombie. With a pidfile: `pgrep -F file.pid`. This is the only method that can't confuse
   two processes.
2. **Match the process name, not the command line**, when the name is distinctive:
   `pgrep -x nginx` (no `-f`). The wrapper shell is called `bash`/`zsh`, so it can't match.
3. **`-A` / `--ignore-ancestors`** excludes pgrep's own parent chain, which covers exactly the
   wrapper shells: `pgrep -A -af 'myserver --port 8080'`. It exists only in newer procps-ng (4.x,
   e.g. Debian 13+). Check with `pgrep --help | grep -q -- --ignore-ancestors`. Older systems
   reject the option (exit `2`).
4. **The bracket trick** for `-f` when nothing else works: `pgrep -af '[m]yserver'`. The regex
   `[m]yserver` matches the text `myserver` in the real process's command line, but the
   wrapper's command line contains the literal text `[m]yserver`, which that regex does not
   match. This works only if your pattern appears in the wrapper **solely** in bracketed form:
   if the same command also says `myserver` plainly anywhere (a `cd myserver`, an `echo`, a log
   path), the wrapper matches again. **Always single-quote it**: unquoted, `[m]yserver` is a
   glob. zsh then aborts with `no matches found`, and bash may silently replace it with a
   matching file name in the current directory.
5. Same idea for ps: `ps aux | grep '[m]yserver'`, or better `ps -C myserver -o pid=,cmd=`
   (match by name, no grep at all).

## Stopping processes

- Stop a process with `kill PID` (SIGTERM) first, and use `kill -9` only if it ignores that.
- `pkill name` or `killall name` can hit other people's processes. Never `pkill -f pattern`
  without first running `pgrep -af pattern` with the same pattern and reading what it lists.
  The self-match there means killing your own wrapper shell, and broad patterns hit unrelated
  processes.

## Resources, ports and network

- Use `top -b -n1`, not `top`. `htop` and similar tools are interactive and hang.
- Resources: `nproc`, `free -h`, `df -h`, `du -sh dir`. Check free space before large builds or
  downloads. How many CPUs and how much memory you may use: see `resources.md`.
- Ports and sockets: `ss -ltnp`. Process names are only shown as root. `netstat` is deprecated
  and often missing.
- Network config: `ip addr`, `ip route`. `ifconfig` and `route` are deprecated and often
  missing.
- Long-running jobs should use the harness's background mechanism rather than a bare `&`, so
  that their output and exit status aren't lost.
