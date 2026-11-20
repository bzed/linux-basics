# Shells: which one runs your commands, and how they differ

Read this when the shell running your commands (or the user's login shell) is not bash, or
when you write `#!/bin/sh` scripts.

## Contents

- Detecting the shell
- What to do when it isn't bash
- zsh
- dash / POSIX sh
- fish, ksh, csh and friends

## Detecting the shell

You probably write commands as bash by habit, but the harness often runs them in **the user's
login shell**, which may well not be bash.

```sh
ps -p $$ -o comm=                         # the shell actually running this command
getent passwd "$(id -un)" | cut -d: -f7   # the user's login shell
echo "$SHELL"                             # usually the login shell, not necessarily the current one
ls -l /bin/sh                             # dash on Debian/Ubuntu
```

## What to do when it isn't bash

- **Wrap it in bash** for anything using bash features (arrays, `[[ ]]`, `${var//x/y}`, process
  substitution, `set -o pipefail`, `read -p`, `mapfile`, `shopt`): `bash -c '...'`, or write a
  `#!/bin/bash` script and run it. This is the reliable choice for multi-line logic.
- **Or write for the shell you're in.** Simple commands (one tool, a pipe, `&&`) work the same
  everywhere, *as long as you quote properly*. Know the differences below.

## zsh

Common as a login shell (e.g. Debian developers, macOS).

- Unquoted variables are **not** word-split: with `v="a b"`, `for w in $v` loops once. Bash
  would loop twice. Use `${=v}` in zsh, or an array.
- An unmatched glob is an **error** (`no matches found: [m]yserver`, `*.log`) and the command
  doesn't run at all. Bash would pass the literal text through. Quote every pattern meant for a
  program, not the shell: regexes, `find -name '*.c'`, `pgrep '[f]oo'`, `apt-get install 'libfoo*'`,
  URLs with `?`/`[]`.
- Arrays start at **1** (`${a[1]}` is the first element), not 0.
- `echo` interprets backslash escapes (`\n`, `\t`) by default. Use `printf '%s\n' "$x"` for
  exact output in any shell.
- `=word` at the start of a word expands to the path of a command (`echo =ls` → `/usr/bin/ls`),
  and `**` is a recursive glob.
- `read -p` doesn't mean "prompt" (zsh uses `read "var?prompt"`), `mapfile`/`readarray`/`shopt`
  don't exist, and `$BASH_SOURCE` is empty.
- Config like `setopt`, aliases or functions from the user's `.zshrc` may change behaviour. A
  wrapped `bash -c` avoids that too.

## dash / POSIX sh

`/bin/sh` on Debian/Ubuntu, `sh -c`, `#!/bin/sh` scripts.

- No arrays, no `[[ ]]` (use `[ ]` and quote both sides), no `==` in `[ ]` (use `=`).
- No `{1..5}` (use `seq`), no `function` keyword, no `source` (use `.`).
- No `$'...'`, no `<<<`, and `echo -e` prints `-e`.
- `set -o pipefail` only in recent dash (test it: `dash -c 'set -o pipefail'`; older releases
  reject it).
- Check scripts with `checkbashisms` (package `devscripts`) or `shellcheck -s sh` if they're
  installed.

## fish, ksh, tcsh/csh, nushell, xonsh

Their syntax differs too much to adapt bash one-liners. Always wrap: `bash -c '...'` (or
`sh -c` for simple POSIX commands). In fish, even `$(...)`, `VAR=x cmd` and `&&` differ or only
work in recent versions.
