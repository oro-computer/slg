# `slg` — Silk Line Grep

`slg` is a fast recursive file + pattern search tool written in Silk
(think: a tiny, hybrid read+`mmap` `rg`/`ag`-style utility).

Implementation note:
- Regular files are probed with a small read and, when <= 64KiB, read into a reusable scratch buffer.
- Larger files fall back to `mmap`/`munmap`.

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

Argument parsing:
- Options may appear before or after `<pattern>` and `[path ...]`.
- Use `--` to end option parsing (useful for patterns or paths starting with `-`).

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
slg -Q "TODO:" .

# Case-insensitive (ASCII-only for -F)
slg -i "error" src

# Smart-case (like rg/ag): enables -i only when the pattern has no ASCII uppercase
slg -S "todo" src

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
#   --jobs 0|auto = auto (default; capped to 9), --jobs 1 = single-threaded, --no-parallel = --jobs 1
#   --threads is an alias for --jobs
#   --max-workers 0 = no cap (default; capped by --jobs-1), --max-workers >= 1 caps worker tasks (clamped to 1023)
#   --split-depth 0|auto = auto (default), --split-depth 1..8 bounds how deep the orchestrator splits directory subtrees into jobs
#   --queue-cap 0|auto = auto (default), --queue-cap >= 1 sets the bounded job queue capacity
#   --target-jobs 0|auto = auto (default), --target-jobs >= 1 stops splitting deeper once this many subtrees have been scheduled (enqueued + in-thread)
#   --max-jobs-total 0|auto = auto (default), --max-jobs-total >= 1 caps the total number of enqueued jobs across the run
#   --file-batch 0|auto = auto (default; currently 256), --file-batch >= 1 sets files per file-batch job (flat, file-heavy dirs)
#   --no-file-jobs disables file-batch jobs, --file-jobs re-enables them (default: on)
#   Note: in this snapshot, --jobs is clamped to 1024 (hard safety limit)
#   --parallel-files enables parallel traversal for --files when jobs > 1
slg --jobs 1 "TODO" .
slg --jobs 0 --max-workers 0 "TODO" .
slg --jobs 0 --split-depth 2 "TODO" .
slg --jobs 0 --queue-cap 512 --max-jobs-total 8192 "TODO" .
slg --files --parallel-files --jobs 0 .

# Print stats to stderr
slg --stats "TODO" .

# Quiet mode (exit immediately on first match)
slg -q "TODO" .

# Suppress line numbers (default output includes them)
slg --no-line-number -F "TODO" .

# Column output requires line numbers
slg --column "TODO" .
```

## Pattern semantics

- By default, `<pattern>` is a regular expression compiled by Silk’s runtime regex engine.
- With `-F/--fixed-string` (aliases: `-Q/--literal`, `--fixed-strings`), `<pattern>` is treated as a literal byte substring (fast path).
- With `-i/--ignore-case`:
  - regex mode: delegated to the runtime regex engine
  - fixed-string mode: ASCII-only case folding
- With `-S/--smart-case`: if `-i/--ignore-case` is not already set and `<pattern>` contains no ASCII uppercase, `slg` enables ignore-case matching.

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

Use `--no-ignore` to disable all ignore sources (common dirs, VCS dirs, ignore files).
Use `--ignore` to re-enable ignore behavior (default).

To re-enable individual ignore sources after `--no-ignore`:
- `--ignore-common`
- `--ignore-vcs`
- `--ignore-files`

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
