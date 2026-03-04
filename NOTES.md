# `slg` Silk/toolchain notes (upstream-focused)

This file captures Silk/toolchain limitations encountered while building `slg`.
It intentionally stays focused on actionable upstream feedback; `slg`-specific
design/usage docs live in `slg/README.md` and `slg/man/slg.1`.

Last re-audit: 2026-03-03 (silk 0.2.0, abi 0.2.0, commit b95225b543b8).

## Minimal repros (`slg/.audit_tmp`)

Run:

```bash
cd slg
../silk/build/bin/silk check .audit_tmp/*.slk
```

Still failing in this snapshot:
- `import std::...;` does not introduce a short module alias (`error[E2028]`): `slg/.audit_tmp/alias_call.slk`
- Integer-literal `match` patterns fail to parse (`error[E0001]`): `slg/.audit_tmp/match_int_literal.slk`, `slg/.audit_tmp/match_stmt_value.slk`
- Statement-form `match (x) { ... }` on ordinary values is rejected (`error[E2002]` / `error[E0001]`): `slg/.audit_tmp/match_stmt_wildcard.slk`, `slg/.audit_tmp/match_stmt_value2.slk`
- Optional chained calls (e.g., `opt?.method()`) are rejected (`error[E2002]`): `slg/.audit_tmp/opt_chained_call.slk`

## Backend / lowering issues observed in `slg`

- Exported structs with fields typed as enums imported from another module can fail lowering or produce confusing type errors (for example: expected `ColorMode`, found `ColorMode`).
  - `slg` workaround: store such fields as `int` codes in exported structs and decode in the consumer.
  - Reference: `slg/src/slg/cli.slk` (`Config.color_mode` comment).
- Coroutine lowering + `std::sync` handles: dropping some `std::sync` handle types at the end of an `async fn` that uses `yield` can crash (SIGSEGV / double-free) in this snapshot.
  - `slg` workaround: clear `handle` fields before scope exit (intentionally leak the underlying resources).
  - Reference: `slg/src/slg/engine.slk` (`disarm_parallel_sync_drops`, channel handle clear in `run_parallel`).
- `yield` lowering rejects some expression forms (`error[E4001]`) that are otherwise type-correct.
  - Examples seen while building `slg`: `yield match (...) { ... }`, and joining tasks returning structs in a loop.
  - `slg` workaround: bind the `match` result to a local before `yield`; return a pointer (`u64`) instead of a struct from worker tasks.
  - Reference: `slg/src/slg/engine.slk` (worker join loop and `WorkerResultPtr`).

## Ergonomics gaps (language)

- Varargs (`...args`) are currently lowered as `len` plus fixed fields (`a0..a31`), so indexing requires manual unrolling.
  - `slg` helper: `slg/src/slg/varargs.slk` (`vararg_string_at`).
  - Upstream ask: make varargs indexable/iterable (or provide a std helper).
