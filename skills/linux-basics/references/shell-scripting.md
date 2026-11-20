# Safe shell scripting

Read this whenever you write a shell script, or a one-liner long enough to have loops,
conditionals or temp files.

## Contents

- Strict mode: `set -euf -o pipefail`
- Quoting
- Temp files with `mktemp` and `trap`
- No `eval`
- Non-POSIX features
- shellcheck
- A template

## Strict mode: `set -euf -o pipefail`

Start every bash script with:

```bash
#!/bin/bash
set -euf -o pipefail
```

| Option | Effect | Why |
| --- | --- | --- |
| `-e` | exit on the first failing command | a failed `cd` or `mkdir` doesn't silently continue into the wrong place |
| `-u` | using an unset variable is an error | a typo like `rm -rf "$BUILDIR/"` aborts instead of expanding to `rm -rf /` |
| `-f` | no pathname expansion (globbing) | an unquoted `$var` or a pattern can never expand into unexpected file names |
| `-o pipefail` | a pipeline fails if any part fails | `false \| tee log` is a failure, not a success |

Know the side effects so they don't surprise you:

- **`-f` turns off globs everywhere**, including ones you meant: `for f in *.txt` loops once over
  the literal `*.txt`. Use `find … -print0 | xargs -0`, `find … -exec`, or turn globbing on just
  where you need it (`set +f; files=(*.txt); set -f`).
- **`-u`**: refer to optional variables and arguments with a default: `"${1:-}"`,
  `"${DEBUG:-0}"`. `"$@"` with no arguments is fine in modern bash.
- **`-e` is not a safety net everywhere.** It doesn't trigger inside `if`/`while` conditions,
  on the left of `&&`/`||`, or in a function called from such a context. `local x=$(cmd)` hides
  `cmd`'s failure: split it into `local x; x=$(cmd)`. Check important steps explicitly.
- **`pipefail` and early-closing readers**: `cmd | head -1` can fail with exit status 141
  (SIGPIPE) because `head` exits before `cmd` is done. Use `cmd | head -1 || true` when that
  is expected, or let the reading tool limit itself (`grep -m1`, `sed -n 1p` reads everything).
- In POSIX sh/dash, use `set -euf` and test whether `set -o pipefail` is supported (see
  `shells.md`).

## Quoting

- Double-quote **every** expansion: `"$var"`, `"${arr[@]}"`, `"$(cmd)"`, `"$@"`. Unquoted
  expansions are split on whitespace and glob-expanded, which breaks on file names with spaces
  and can turn into unintended arguments.
- Use single quotes for literal strings and patterns: `grep -E 'a|b'`, `find -name '*.c'`.
- Put `--` before arguments that come from variables or file names, so a value starting with
  `-` isn't read as an option: `rm -- "$file"`, `grep -- "$pattern" "$file"`.
- Build commands as arrays, not strings: `args=(-v --color=never); grep "${args[@]}" -- "$p"`.
- Iterate over files safely: `find . -name '*.log' -print0 | while IFS= read -r -d '' f; do …; done`.
  Never `for f in $(ls …)` or `for f in $(find …)`.
- Read lines with `while IFS= read -r line`, not `for line in $(cat file)`.
- Use `printf '%s\n' "$x"` instead of `echo "$x"` for arbitrary data (`echo` interprets `-n`/`-e`
  and, in some shells, backslashes).

## Temp files with `mktemp` and `trap`

Never use fixed names like `/tmp/out.txt`: they collide with parallel runs and other users, and
predictable names in `/tmp` are a security risk (symlink attacks).

```bash
tmpdir=$(mktemp -d "${TMPDIR:-/tmp}/myscript.XXXXXX")
trap 'rm -rf -- "$tmpdir"' EXIT
tmpfile=$(mktemp "$tmpdir/data.XXXXXX")
```

- One temp **directory** per script and one `trap … EXIT` to remove it is simpler and more
  reliable than tracking individual files.
- The `EXIT` trap runs on normal exit and on `set -e` aborts. In bash it also runs after `INT`
  or `TERM` stop the script. For other shells, add `trap 'exit 1' INT TERM` so the EXIT trap
  fires there too.
- Single-quote the trap body so `$tmpdir` is expanded when the trap runs, and quote it inside.
- A second `trap … EXIT` replaces the first. Put all cleanup in one function:
  `cleanup() { rm -rf -- "$tmpdir"; …; }; trap cleanup EXIT`.
- Size and location rules from `filesystem.md` apply: tmpfs `/tmp` is RAM, large data goes to
  `/var/tmp`.

## No `eval`

`eval` re-parses its argument as shell code, so any data in it (file names, user input, command
output) can execute commands. There is almost always a safe alternative:

| Instead of | Use |
| --- | --- |
| `eval "$cmd"` to run a built command | an array: `cmd=(tar -czf "$out" -- "$dir"); "${cmd[@]}"` |
| `eval "echo \$$name"` (indirect variable) | `"${!name}"` (bash) or an associative array |
| `eval "$name=\$value"` (dynamic assignment) | `printf -v "$name" '%s' "$value"` (bash) or `declare -g` |
| `eval "$(some-tool)"` (env setup) | only for trusted tools that document it (e.g. `ssh-agent`); otherwise parse the output |

The same applies to `sh -c "…$var…"`, `bash -c` with interpolated data, and `source` of
generated files: pass data as arguments (`sh -c 'cmd "$1"' _ "$var"`), never by pasting it into
code.

## Non-POSIX features

Bash features (arrays, `[[ ]]`, `${var//a/b}`, `<( )`, `printf -v`, `${!name}`, `mapfile`) need
bash. Use `#!/bin/bash`, and run the script with bash, not `sh script.sh`, which ignores the
shebang and uses dash on Debian. In interactive one-liners, check which shell runs your commands
and wrap bash-specific code in `bash -c` (see `shells.md`). Scripts with `#!/bin/sh` must stay
POSIX.

## shellcheck

Run [shellcheck](https://www.shellcheck.net) on every script you write or change before you run
it or hand it over:

```sh
shellcheck script.sh                 # human-readable
shellcheck -f gcc script.sh          # one line per finding: file:line:col: severity: message [SCxxxx]
shellcheck -s sh script.sh           # check as POSIX sh (also: -s bash, -s dash)
shellcheck -S warning script.sh      # only warnings and errors
```

- Fix findings rather than silencing them. If a warning really doesn't apply, disable it for that
  line only with a reason: `# shellcheck disable=SC2086 # intentional word splitting of $FLAGS`.
- shellcheck uses the shebang to decide the dialect. Without one, pass `-s`.
- If shellcheck isn't installed (`command -v shellcheck`), say so. Installing it (`shellcheck`
  package on Debian) needs root, so ask (`SKILL.md`, section 1).

## A template

```bash
#!/bin/bash
set -euf -o pipefail

usage() { printf 'usage: %s <input-dir> <output-file>\n' "${0##*/}" >&2; exit 2; }
[[ $# -eq 2 ]] || usage
in_dir=$1
out_file=$2

tmpdir=$(mktemp -d "${TMPDIR:-/tmp}/${0##*/}.XXXXXX")
cleanup() { rm -rf -- "$tmpdir"; }
trap cleanup EXIT

find "$in_dir" -type f -name '*.log' -print0 |
  while IFS= read -r -d '' f; do
    grep -h -- 'ERROR' "$f" || true      # grep exits 1 when nothing matches
  done > "$tmpdir/errors.txt"

mv -- "$tmpdir/errors.txt" "$out_file"
```
