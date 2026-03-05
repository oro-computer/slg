# `slg` Silk/toolchain notes (upstream-focused)

This file captures Silk/toolchain limitations encountered while building `slg`.
It intentionally stays focused on actionable upstream feedback; `slg`-specific
design/usage docs live in `slg/README.md` and `slg/man/slg.1`.

Last re-audit: 2026-03-05 (silk 0.2.0, abi 0.2.0, commit 382cc9e11f3a).

## Status

No known Silk/toolchain issues affecting `slg` in this workspace snapshot.

## Regression probes (`slg/.audit_tmp`)

Run:

```bash
cd slg
../silk/build/bin/silk check .audit_tmp/*.slk
```
