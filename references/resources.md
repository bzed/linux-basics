# CPU and memory: don't take over the machine

Read this before builds, test suites, compression, encoding or anything else that can run in
parallel or use a lot of memory.

## Contents
- Why it matters
- Check the machine first
- How many jobs to use
- Tools that use all cores by default
- Memory
- Lowering priority and hard limits

## Why it matters

The machine you run on is usually somebody's workstation: a desktop session, an editor, a
browser, maybe a video call. A job that occupies every CPU core makes all of that stutter or
freeze. A job that eats all memory pushes the system into swapping and then into the OOM killer,
which may kill the user's applications instead of your job. So **never use all CPU cores, or
anything close to all memory, unless the user explicitly granted it** (e.g. "this is a throwaway
build VM, use everything"). That permission applies to that machine and task, not in general.

## Check the machine first

```sh
nproc                                   # CPUs available to you (respects affinity/containers)
cat /proc/loadavg                       # load average; compare the first number with nproc
free -h                                 # look at "available", not "free"
systemctl get-default 2>/dev/null       # graphical.target → a desktop machine
echo "${WAYLAND_DISPLAY:-}${DISPLAY:-}" # non-empty → you're inside a graphical session
```

A desktop session, a high load average or other users' processes (`top -b -n1 | head -20`)
mean less room for you. Servers and CI runners may have other workloads too: ask if unsure.

## How many jobs to use

- Default to **at most half of `nproc`**, and never more than `nproc - 2`, on a machine people
  use interactively. Example: `make -j"$(( $(nproc) / 2 ))"`. A 24-core workstation then builds
  with 12 jobs, which is still fast and leaves the desktop responsive.
- Use the full core count only when the user said so. Then still prefer `nice` (see below).
- Don't run several heavy jobs at the same time (two parallel builds, a build plus a parallel
  test suite). Each one sizes itself to the whole machine.

## Tools that use all cores by default

Many tools are either single-threaded by default (fine) or grab every core (not fine).
Always pass an explicit limit to these:

| Tool | Default | Limit it with |
|---|---|---|
| `make -j` (no number) | **unlimited** jobs | `make -j"$N"`. Never a bare `-j` |
| `cmake --build . -j` (no number) | build tool default, unlimited with make | `cmake --build . -j "$N"` or `CMAKE_BUILD_PARALLEL_LEVEL=$N` |
| `ninja` | `nproc + 2` | `ninja -j "$N"` |
| `cargo build` / `cargo test` | all CPUs | `cargo build -j "$N"`, `CARGO_BUILD_JOBS=$N`; tests: `-- --test-threads=$N` |
| `go build` / `go test` | all CPUs | `go build -p "$N"`, `GOMAXPROCS=$N` |
| `pytest -n auto` (xdist) | all CPUs | `pytest -n "$N"` |
| `xz -T0`, `zstd -T0` | all CPUs | `-T"$N"` |
| `dpkg-buildpackage`, `debuild` with `DEB_BUILD_OPTIONS=parallel=…`, `sbuild` with `-j` | as given | pick `N`, don't pass `$(nproc)` |
| `npm`/`node`, `webpack`, `jest` | often all CPUs | `jest --maxWorkers=$N`, tool-specific |
| `ffmpeg`, `parallel`, `xargs -P 0` | all CPUs / unlimited | `-threads $N`, `parallel -j $N`, `xargs -P $N` |

Set `N` once (`N=$(( $(nproc) / 2 ))`, minimum 1) and reuse it.

## Memory

- Parallel builds multiply memory use: each C++ or Rust compile or link job can take 1–2 GB or
  more. With `-j12` that's easily 20 GB. Keep `N × memory per job` well below `available` in
  `free -h`, and lower `N` for heavy C++/Rust/LTO builds.
- Don't load huge files into memory at once (reading a multi-GB log into a variable, `sort` of
  a giant file without `-S`/`-T`, Python reading a whole dataset). Stream or chunk instead.
- Remember that tmpfs (`/tmp` on Debian 13+) is memory too. See `filesystem.md`.
- If the OOM killer struck, `journalctl -k --no-pager | grep -i 'out of memory'` shows it
  (needs journal access, see `logs.md`).

## Lowering priority and hard limits

- Run heavy jobs at low priority so interactive work wins: `nice -n 19 ionice -c 3 make -j"$N"`.
  This needs no root. (`ionice -c 3` is "idle" I/O class.)
- For a hard cap without root, use a transient systemd user scope (works when the user's
  systemd delegates the `cpu` and `memory` controllers, as Debian does by default):

  ```sh
  systemd-run --user --scope --quiet -p CPUQuota=400% -p MemoryMax=8G -- make -j4
  ```

  `CPUQuota=400%` means at most four cores' worth of CPU time. When `MemoryMax` is hit, the job
  gets OOM-killed instead of the user's desktop. Check
  `cat /sys/fs/cgroup/user.slice/user-$(id -u).slice/user@$(id -u).service/cgroup.controllers`
  to see which controllers are delegated.
- Containers and CI runners often already have a cgroup limit, and `nproc` may then show more
  CPUs than you're really allowed. `cat /sys/fs/cgroup/cpu.max` shows the quota.
