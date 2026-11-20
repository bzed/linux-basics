# Searching with ripgrep (`rg`)

Read this before searching code or files with `rg`, and before concluding from an `rg` search
that something doesn't exist. Based on `man rg` (ripgrep 15.x).

## Contents

- What rg skips by default, and why results can be missing
- Turning the filtering off
- Everyday options
- Machine-readable output: `--json`
- Exit status
- Pitfalls

## What rg skips by default, and why results can be missing

`rg PATTERN [PATH…]` searches recursively, starting in the current directory. Unlike `grep -r`,
it does **four kinds of automatic filtering**:

1. **Ignore rules**: files and directories matched by `.gitignore` (in the repository and its
   parent directories up to the repository root), `.git/info/exclude`, the global git excludes
   file (`core.excludesFile`, default `~/.config/git/ignore`), and `.ignore` / `.rgignore`
   files. Git ignore rules only apply inside a git repository, i.e. when a `.git` exists.
2. **Hidden files and directories**: anything whose name starts with `.` (`.env`, `.github/`,
   `.config/`, `.gitlab-ci.yml`).
3. **Binary files**: once a NUL byte is seen, the file is treated as binary and searching it
   stops.
4. **Symbolic links** are not followed.

So **"no matches" does not mean "not there"**. Typical things it silently skips: build output
(`build/`, `target/`, `dist/`), `node_modules/`, vendored dependencies, generated files, CI
configuration in `.github/`, dotfiles, and anything reached through a symlink. Also note: a file
that is tracked by git but matches a `.gitignore` rule is still skipped (`git grep` searches it).

Find out what was skipped and why:

```sh
rg --files | wc -l          # files rg would search
rg --files -uuu | wc -l     # all files
rg --debug PATTERN 2>&1 | rg 'ignoring'   # each skipped path with the rule that caused it
```

## Turning the filtering off

Use the narrowest switch that answers your question:

| Option | Effect |
|---|---|
| `-.` / `--hidden` | also search hidden files and directories (includes `.git/`, add `-g '!.git'`) |
| `--no-ignore` | ignore `.gitignore`/`.ignore`/`.rgignore`/global/exclude rules |
| `--no-ignore-vcs` | ignore only version-control rules (`.gitignore`), still respect `.ignore`/`.rgignore` |
| `--no-ignore-parent` | don't apply ignore files from parent directories |
| `-L` / `--follow` | follow symbolic links |
| `--binary` | keep searching binary files after a NUL byte (prints a warning, not binary data) |
| `-a` / `--text` | treat binary files as text (may print binary garbage) |
| `-u` | = `--no-ignore` |
| `-uu` | = `--no-ignore --hidden` |
| `-uuu` | = `--no-ignore --hidden --binary`: searches everything except symlinks |

A path given explicitly on the command line is always searched, regardless of ignore and hidden
rules: `rg PATTERN .env build/out.log`.

When a search comes back empty and the thing might be in a hidden, ignored or generated file,
repeat it with `-uu` (or `-uuu`) before reporting that it doesn't exist. Say which variant you
used.

## Everyday options

| Option | Meaning |
|---|---|
| `-F` / `--fixed-strings` | literal string, not a regex (use for code with `(`, `[`, `.`, `$`, `*`) |
| `-e PATTERN` | pattern, also one starting with `-`. Repeat for several patterns |
| `-i` / `-S` / `-s` | ignore case / smart case (case-insensitive unless the pattern has uppercase) / case-sensitive |
| `-w` / `-x` | match whole words / whole lines |
| `-v` | invert match |
| `-t TYPE` / `-T TYPE` | only / not files of a type (`-t py`, `-t rust`, `-T js`); list types with `--type-list` |
| `-g GLOB` | include (`-g '*.toml'`) or exclude (`-g '!tests/**'`) paths; overrides ignore rules. A directory needs `'dir/**'`, not `dir` |
| `--iglob GLOB` | case-insensitive glob |
| `-l` / `--files-without-match` | only file names with / without matches |
| `-c` / `--count-matches` | matching lines / matches per file |
| `-o` | print only the matched part |
| `-n` / `-N` | with / without line numbers (rg adds them only on a terminal) |
| `-H` / `-I` | with / without file name |
| `-A N` / `-B N` / `-C N` | context lines after / before / around |
| `-m N` | at most N matching lines per file |
| `-U` | multiline: patterns may match across lines (`--multiline-dotall` lets `.` match newlines) |
| `-P` | PCRE2 engine: look-around and backreferences (the default engine has neither) |
| `-r REPL` | replace matches **in the output only**. rg never modifies files |
| `--files` | list the files that would be searched (no pattern) |
| `-d N` / `--max-filesize 10M` | limit directory depth / skip large files |
| `--one-file-system` | don't cross mount points |
| `-M N` | omit lines longer than N bytes (useful against minified files) |
| `--sort path` | stable, sorted output (disables parallelism) |
| `-j N` | threads; default is chosen automatically, so set it on busy or shared machines (see `resources.md`) |
| `-z` | also search compressed files (gz, bz2, xz, zstd, …) |
| `--no-config` | ignore `RIPGREP_CONFIG_PATH`, so a user config can't change behaviour |

The pattern syntax is Rust regex, close to ERE (`grep -E`) plus Perl classes like `\d`, `\w`,
`\b`. No look-around and no backreferences without `-P`.

## Machine-readable output: `--json`

`--json` prints **JSON Lines**, one object per line. This is the most reliable way to process
results in a program, because file names and lines with `:`, spaces or odd bytes can't break
the parsing.

```sh
rg --json -F 'TODO' src/
```

Message types (`"type"`): `begin` (a file with matches starts), `match`, `context` (with `-A`/
`-B`/`-C`), `end` (per-file statistics), `summary` (totals, always at the end). A match looks
like:

```json
{"type":"match","data":{"path":{"text":"a.txt"},"lines":{"text":"needle here\n"},"line_number":1,"absolute_offset":0,"submatches":[{"match":{"text":"needle"},"start":0,"end":6}]}}
```

- Text fields are objects with either `"text"` (valid UTF-8) or `"bytes"` (base64 for anything
  else). Handle both.
- `submatches[].start`/`end` are byte offsets within the line.
- Extract with jq: `rg --json PATTERN | jq -r 'select(.type=="match") | "\(.data.path.text):\(.data.line_number): \(.data.lines.text)"'`.
- `--json` can't be combined with `--files`, `-l`, `--files-without-match`, `-c` or
  `--count-matches`. Options like `-o`, `-r`, `-M`, `--heading` have no effect with it.
- Without JSON, `--vimgrep` (`file:line:column:text`) or `-n -H --no-heading` give stable
  one-match-per-line output. `--null` (`-0`) ends file names with NUL for `xargs -0`.

## Exit status

- `0`: at least one match (and no error).
- `1`: no match, no error.
- `2`: an error, e.g. invalid regex or an unreadable file. This happens even if other files
  matched, unless `-q` is used and something matched.

Under `set -e`, a search without matches aborts the script: use `rg … || true`, or check for
`1` explicitly. Don't treat `2` as "not found".

## Pitfalls

- **stdin detection**: when stdin is a readable pipe or file and no path is given, rg searches
  stdin instead of the directory. In agent harnesses stdin can be an open pipe, so the search
  hangs or finds nothing. **Always give a path** (`rg PATTERN .` or `rg PATTERN src/`).
- Quote patterns and globs in single quotes. In zsh, `rg -g *.py foo` fails with
  `no matches found`; in bash it may expand to file names.
- Output on a pipe differs from a terminal (no colours, no headings, file name on every line).
  Rely on explicit flags (`-n`, `-H`, `--no-heading`, `--color=never`) when parsing.
- A user config file (`RIPGREP_CONFIG_PATH`) can add flags silently. Add `--no-config` when
  results must be reproducible.
- `rg` may not be installed (`command -v rg`). Then fall back to `grep -rn` (which has none of
  the filtering) or `git grep -n` inside a repository.
