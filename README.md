# linux-basics

An [Agent Skill](https://agentskills.io) that teaches coding agents (Claude Code, Mistral
Vibe and other skill-aware agents) how to behave when they run commands on a Linux machine.
Debian is the primary target; other distributions are covered where they differ.

Most of the content exists to fix mistakes agents make again and again: assuming a system
instead of checking it, starting commands that wait for input that never comes, taking over
the whole machine, or leaving side effects behind that nobody asked for.

## What it covers

| Topic | Key rules |
|---|---|
| Root | Never `sudo`/`su`/`doas`/`pkexec`/`run0` without the user's explicit permission, not even `sudo -n true` probes |
| Orientation | Detect distro, Debian release (including testing/unstable `…/sid`), login shell and groups before acting |
| Shells | Commands may run in zsh or dash, not bash: wrap in `bash -c` or know the differences (globbing, word splitting, arrays) |
| Packages | `apt-get` over `apt`, `DEBIAN_FRONTEND=noninteractive`, no `pip --break-system-packages` |
| Services and logs | `--no-pager` everywhere; journal access via `adm`/`systemd-journal` or the per-user journal (regular UID range); thin output means "no permission", not "no logs" |
| Temp and build files | `/tmp` or `/var/tmp`, never `$HOME`; don't overfill a tmpfs `/tmp`; both are cleaned by systemd-tmpfiles |
| CPU and memory | Never use all cores unless granted (at most half of `nproc` by default), limit tools that grab every core, keep memory in check, `nice`/`ionice`/`systemd-run --user` caps |
| Processes | `pgrep -f` matches the agent's own wrapper shell: track PIDs, use `-x`, `-A` or the quoted `'[f]oo'` trick |
| Containers | podman: mount only what's needed, set CPU/memory/pids/network limits, no host network, clean up with `--rm` and labels, never `prune` |
| Shell habits | No pagers or editors, POSIX `#!/bin/sh`, `set -euo pipefail`, `${dir:?}` guards, `LC_ALL=C` for parsing |

## Layout

```
linux-basics/
├── SKILL.md            the rules, loaded whenever the skill triggers
├── references/         details, read on demand
│   ├── packages.md     distro/release detection, package managers, apt, PEP 668
│   ├── shells.md       shell detection, zsh / dash / other shells
│   ├── logs.md         systemctl, journalctl, who can read which logs
│   ├── filesystem.md   temp/build locations, tmpfs, tmpfiles cleanup, config files
│   ├── resources.md    CPU and memory limits, per-tool parallelism
│   ├── processes.md    pgrep/pkill done right, stopping processes, ports
│   └── containers.md   podman: volumes, limits, networking, cleanup
└── evals/evals.json    test prompts used while developing the skill
```

`SKILL.md` stays short on purpose: it holds the rules, and each section points to the
reference file with the details.

## Installation

The skill directory must be named `linux-basics` (the `name` in `SKILL.md`). A plain
`git clone` already gives you that; keep the name if you clone somewhere else.

```sh
git clone https://github.com/bzed/linux-basics.git ~/workspace/skills/linux-basics
```

### Claude Code

Personal skill, available in all projects:

```sh
mkdir -p ~/.claude/skills
ln -s ~/workspace/skills/linux-basics ~/.claude/skills/linux-basics
```

Or for a single project only, so everyone working on it gets the skill:

```sh
mkdir -p .claude/skills
git clone https://github.com/bzed/linux-basics.git .claude/skills/linux-basics
```

Claude Code picks up new skills in new sessions. Check with `/skills` (or ask "which skills
do you have?") that `linux-basics` is listed. Update with `git -C ~/workspace/skills/linux-basics pull`.

### Mistral Vibe

```sh
mkdir -p ~/.vibe/skills
ln -s ~/workspace/skills/linux-basics ~/.vibe/skills/linux-basics
```

### Other agents

Any agent that supports the [Agent Skills](https://agentskills.io) format can load the
directory as is. Point the agent's skills directory at it the same way.

## Development

Validate the skill with `agentskills` from the reference library
[skills-ref](https://github.com/anthropics/agentskills) (pass an absolute path; a bare `.`
fails the directory-name check):

```sh
agentskills validate "$PWD"
```

`evals/evals.json` holds the test prompts (installing a Python library, building jq from
source, diagnosing a service from its logs). Run them with and without the skill to see
whether a change actually improves behaviour.

Contributions are welcome, especially new rules backed by mistakes you have seen agents make.

## License

[MIT](LICENSE)
