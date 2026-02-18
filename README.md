# `slg` — Silk Line Grep

`slg` is a fast recursive file + pattern search tool written in Silk
(think: a tiny, mmap-first `rg`/`ag`-style utility).

## Build & run

```bash
cd slg
silk build
install -b build/bin/slg ~/.local/bin/slg
slg --help
```

To view the manual page:

```bash
man ./man/slg.1
```

## Usage

Search file contents:

```bash
slg [options] <pattern> [path ...]
```

List files that would be searched (no content search):

```bash
slg --files [options] [path ...]
```

Important parsing rule (for speed + simplicity):
- Options must appear before `<pattern>`. Any arguments after `<pattern>` are treated as paths.

Exit status:
- `0` match found (or `--files` completed successfully)
- `1` no matches
- `2` error

## Common examples

```bash
# Regex search (default)
slg "T.DO" .

# Fixed-string search (fast path)
slg -F "TODO:" .

# Case-insensitive (ASCII-only for -F)
slg -i "error" src

# Show only files with matches
slg -l "TODO" .

# Print a file header before matches
slg --heading "TODO" .

# Stop after N matching lines per file
slg -m 3 "TODO" .

# Include hidden files/dirs (dotfiles)
slg --hidden "TODO" .

# Follow symlinks (may loop)
slg --follow "TODO" .

# Limit recursion depth (0 = only the root)
slg --max-depth 2 "TODO" .

# Parallelism control:
#   --jobs 0 = auto (default), --jobs 1 = single-threaded, --no-parallel = --jobs 1
#   --max-workers 0 = no cap (default; currently clamped to 8), --max-workers 1..8 caps worker tasks (values > 8 clamp to 8)
#   Note: in this snapshot, effective --jobs is clamped to 9 (1 orchestrator + 8 workers)
#   --parallel-files enables parallel traversal for --files when jobs > 1
slg --jobs 1 "TODO" .
slg --jobs 0 --max-workers 0 "TODO" .
slg --files --parallel-files --jobs 0 .

# Print stats to stderr
slg --stats "TODO" .

# Quiet mode (exit immediately on first match)
slg -q "TODO" .
```

## Pattern semantics

- By default, `<pattern>` is a regular expression compiled by Silk’s runtime regex engine.
- With `-F/--fixed-string`, `<pattern>` is treated as a literal byte substring (fast path).
- With `-i/--ignore-case`:
  - regex mode: delegated to the runtime regex engine
  - fixed-string mode: ASCII-only case folding

## Binary / text detection

By default, `slg` skips files that look binary (contain a NUL byte in the first 1024 bytes).
Use `--text` to force searching those files anyway.

## Ignores

By default, `slg` skips a small set of common “heavy” directories:
`node_modules`, `build`, `dist`, `target`, `tmp`, `.cache`.

Use `--no-ignore-common` to include them.

By default, `slg` also reads per-directory ignore files:
`.gitignore`, `.ignore`, `.agignore`.

Use `--no-ignore-files` to disable ignore file reading.

## Color

`slg` highlights matches and headings when color is enabled.

- `--color auto|always|never` (default: `auto`)
- `--no-color` (same as `--color never`)
- `NO_COLOR=1` disables color in `--color auto` mode


## Tests

Unit + end-to-end tests use Silk Test syntax and run via:

```bash
cd slg
silk test
```

## License

MIT
