# `slg` (Silk Line Grep) Notes

This file documents Silk/toolchain expectations encountered while building `slg`, along with the workarounds used in this project.

Last re-audit: 2026-02-08 (Silk (ABI) 0.2.0 in this workspace).

## Silk subset / toolchain limitations (observed)

### 1) Stdlib support is uneven across modules
`slg` intentionally avoids most higher-level stdlib modules (for now) and uses:
- `std::runtime::posix::{fs,io,time,env}` for syscall-heavy functionality
- `std::path` (`std::path::PathBuf`) for path building
- a small local buffering/output layer (to avoid subset/backend gaps in some higher-level stdlib I/O abstractions)

Re-audit result:
- `std::io::println(...)` compiles and runs in this workspace.
- Other stdlib surfaces are not exhaustively re-verified; we still observe subset/backend rejections in parts of the shipped stdlib (for example, qualified struct literals in some modules).

### 2) Optional coalescing (`??`) is parsed but rejected by the backend subset
The optional-coalescing operator `opt ?? fallback` is accepted by the parser in this workspace, but the hosted backend subset rejects it (`error[E4001]`: unsupported expression `Binary`).

Workaround:
- use `match (opt) { Some(v) => v, None => fallback }` instead

### 3) `match` expressions are supported; `match` statements are limited to typed errors
In the current subset:
- `match <scrutinee> { ... }` works as an *expression* (arms are expressions, not blocks).
- The *statement* form `match (<error_call(...)>) { ... }` is supported for typed error handling (`T | ErrorType...`).
- A statement-form `match (x) { ... }` where `x` is an ordinary value is still rejected with `E2002`.

Workarounds:
- use `match` in expression position for ordinary values,
- for typed-error-producing calls, use the typed errors `match` statement,
- otherwise use `if`/`while` statements plus `match` expressions.

### 4) Some string-valued control-flow expressions are not lowered
We observed backend failures when lowering `let s: string = if cond { "a" } else { "b" }`.

Re-audit result (2026-02-08):
- `if` as an expression lowers for scalar types (e.g., `int`),
- but `if` as an expression producing a `string` still fails backend lowering (`error[E4001]`).

Workaround:
- use a `var` and an `if` statement:
  - `var s: string = "..."; if cond { s = "..."; }`

### 5) Module-level `string` constants are not reliably lowered
We observed backend failures when referencing `export const X: string = "...";` from other modules.

Re-audit result (2026-02-08):
- even stdlib string constants can fail when referenced from user code (e.g. `std::path::SEP`).

Workaround:
- expose these as zero-argument functions returning string literals (avoid cross-module `export const X: string = ...`)

### 6) Method calls on `mut` field-access receivers can be unsupported
Example pattern (problematic in the current subset):
- calling a method directly on `mut self.some_field` (or similar `mut <expr>.field`) receivers

Re-audit result (2026-02-08):
- calling a `mut` receiver method through a field can trigger `E2005` (“invalid assignment”) even when the field is logically writable (e.g., `self.inner.inc()` inside `mut self: &Outer`).

Workaround:
- move the field to a local variable, call methods on the local, then write back if needed
- this pattern is used in `slg`’s buffered output layer and in end-to-end test helpers where owned buffers must be dropped safely

### 7) Generics in type positions no longer look restricted (historical)
We previously avoided applying generic types at use sites (e.g., `std::result::Result(T, E)` in signatures).

Re-audit result: generic applications in type positions now work in this workspace (including cross-module type aliases).

In `slg` today:
- some hot-path APIs still use local result enums instead of `std::result::Result(T, E)` for explicit, allocation-free error handling

### 8) Regex ownership + result destructuring can cause SIGABRT / double-free hazards
We observed runtime aborts (SIGABRT / “core dumped”) when using regex mode (the default, i.e. without `-F`).

Root cause (toolchain snapshot):
- constructing a user-defined enum/struct that *stores a `std::regex::RegExp` by value* inside a `match` over
  `std::regex::RegExp.compile(...)` can trigger an ownership bug (even when using `move` in arms).
- separate but related: `std::regex::RegExp` appears shallow-copyable by assignment in this snapshot; patterns that
  "copy, overwrite, then explicitly `.drop()` the copy" can double-free.

Workarounds used by `slg`:
- `slg` uses `Result.unwrap_or(...)` (a consuming helper) instead of destructuring the compile result directly, and constructs the final pattern value outside of the compile-result `match`.
- `Pattern.drop` resets `self.regex = RegExp.empty()` and avoids explicit `RegExp.drop()` calls.

Regression coverage:
- the test suite includes a regex compile + execution test to ensure this path does not abort.

Re-audit result (2026-02-08):
- With the current workarounds in place, we no longer reproduce the abort in our automated tests or in manual invocation.

However, the underlying toolchain bug still reproduces in this workspace when destructuring the regex compile result directly and storing `RegExp` by value inside a user-defined enum/struct payload (this crashes with `Aborted (core dumped)`).

### 9) Exported structs with cross-module enum fields can fail backend lowering
We observed `error[E4001]: unsupported construct in the current backend subset` when:
- a module exports a `struct` type,
- the struct contains a field typed as an `enum` imported from another module, and
- that module also exports functions.

Repro (historical, this workspace snapshot):
- an exported `struct` with a field typed as an `enum` imported from another module triggered a backend lowering failure, even though parsing and type-checking succeeded.

Workaround used by `slg`:
- store the imported-enum field as an `int` code in the exported struct,
- decode it back to the enum type in the consuming module.

Re-audit result (2026-02-08):
- This still reproduces in Silk (ABI) 0.2.0 in this workspace.

## Formal Silk usage

`slg` is a low-level, syscall-heavy program; much of the production code uses raw pointers, external calls, and mmap.
In this toolchain snapshot, adding Formal Silk directives (e.g., `#theory`, `#invariant`) to such functions can trigger `E3005` (“unsupported construct in verified function”).

What we do today:
  - Keep a small, verified Formal Silk smoke test (compile-time only).
  - Write production code in a way that makes future formalization easier (explicit invariants in docs, small functions, centralized OS bindings).

## Concurrency (`async` / `await` / `task` / `yield`)

`slg` uses Silk’s current concurrency subset:

- The entrypoint is `async` and uses `await` to run the selected engine runner.
- The engine supports task-based parallel search (Mode::Search):
  - `--jobs 0` (default) picks a runtime default (`std::task::available_parallelism()`).
  - `--jobs 1` (or `--no-parallel`) forces the single-threaded engine.
  - For `--jobs > 1`, `slg` spawns up to `min(--jobs - 1, 8)` worker tasks for **top-level subdirectories**
    (depth=1) and processes remaining work in the orchestrator thread.
  - Output is serialized with a `std::sync::Mutex` so stdout/stderr don’t interleave.
  - `--quiet` uses a `std::sync::CancellationToken`; hot loops check it periodically to stop other tasks sooner.

Observed toolchain/subset limitations (this snapshot):

- the specific expression form `yield match (...) { ... }` failed backend lowering; `slg` binds the handle first, then `yield`s (`let h = match (...); let r = yield h;`).
- Task handles are non-copyable; storing/extracting requires explicit `move` in some patterns.
- Fixed-size arrays containing `Task(T)` still cannot be constructed ergonomically:
  - array literals attempt to *copy* task handles (`cannot copy a Task/Promise handle`),
  - `move` inside array literals currently yields confusing `use of moved value` errors.
  As a result, `slg` keeps an unrolled, fixed number of handle slots instead of a real worker pool.

Future work:
- Replace depth=1 splitting with a bounded work queue once task handles can be stored in collections and the runtime
  uses a thread pool.

## CLI parsing note

Options must appear before `<pattern>`.
Once `<pattern>` is parsed, remaining positional arguments are treated as paths (this keeps parsing fast and allocation-free).

## Ergonomics feedback (for upstream)

This section is a “wish list” of language + toolchain improvements motivated by
concrete friction points in `slg`’s code. The goal is to make common patterns
shorter and harder to get wrong, especially around `T?` (Option) and
`std::result::Result(...)`.

### 1) Option (`T?`) ergonomics are currently very `match`-heavy
In this toolchain snapshot, the following common “unwrap” syntaxes are **not**
accepted by the parser (tested via `silk check`):

```silk
// Desired:
if let Some(v) = x { ... }         // rejected
let Some(v) = x else { ... };      // rejected
let y = x ?? return 2;             // rejected (and `return` is not an expression)
```

Practical impact in `slg`:
- We frequently write `if opt == None { ... }` and then a second `match(opt)` to
  extract `Some(v)`.
- Tests often have “unreachable” fallback values just to satisfy typing after
  `assert(opt != None)`.
- Optional field access (`opt?.field`) *is* supported and helps for `Struct?`
  cases, but optional chained calls (e.g., `opt?.method()`) are still rejected
  in this subset.

Upstream improvements that would materially reduce code size:
- `if let` / `while let` pattern binding (for `Some(...)`, `Ok(...)`, etc.).
- `let-else` destructuring (`let Some(v) = ... else { ... }`).
- A concise Option/Result propagation mechanism (similar to typed-error `call()?`,
  but for recoverable values like `T?` and `Result(T, E)`).
- Block arms (or a statement form) for `match` expressions so error-printing /
  early-return unwraps don’t require dummy temporaries.

### 2) Missing “bottom” / `unreachable()` expression forces dummy values
Because we can’t express “this branch is impossible” in a type-directed way,
we often write placeholders in `None`/`Err` arms even after a guard/`assert`.

Upstream improvements:
- Provide an `unreachable()`/`panic()` expression with a `never`/bottom type.
- (Advanced) add basic flow-sensitive refinement so `assert(x != None)` lets
  the compiler treat `x` as non-optional in the following block.

### 3) Coalescing + propagation for `T?` and `Result(T, E)` is missing
Because `??` is not available in the current backend subset (see above), we can’t
use it to simplify common “fallback” patterns. Separately, `return` is not an
expression, so even with coalescing it’s hard to express “fallback that returns
early” without boilerplate.

Upstream improvements:
- Typed errors already support `call()?` (propagation of `panic`-style errors),
  but `T?` and `Result(T, E)` do not have an equivalent early-return operator.
- Add an early-return operator for `T?` and `Result(T, E)` (or an explicit
  `try`-style keyword/operator that doesn’t conflict with typed errors).
- Or add a block-form coalesce (`x ?? { ... }`) where the RHS can be a statement
  block that ends in `return`/`break`/etc.

### 4) Stdlib `std::path::PathBuf` is good, but lacks a safe truncate API
`slg` uses `std::path::PathBuf` for path building in hot directory-walk loops.
For performance, we need an O(1) “restore previous length” operation. Today we
implement this by writing `ptr[len]=0` directly in the hot loop.

Upstream improvements:
- Add `PathBuf.truncate_len(new_len)` (or similar) that maintains invariants
  (including the trailing NUL) without requiring field access.

### 5) Mutability noise (`mut`) and receiver limitations increase boilerplate
We still hit patterns where the subset rejects method calls on `mut` field
access receivers (necessitating move-to-local workarounds).

Upstream improvements:
- Expand backend lowering support for method calls on `mut <expr>.field`.
- Consider reducing “mut binding noise” where the compiler can infer that a
  binding must be mutable (or allow `mut` on a narrower scope).

### 6) Tooling/docs gaps: built-in core types are hard to discover
`silk man` resolves docs for `std::strings::String` and many std modules, but
core/builtin types like `string` and `T?` (Option) do not have direct docs.

Upstream improvements:
- Add `silk man` entries (or aliases) for `string`, `Option`/`T?`, and other
  core language types/operators (`??`, `task`, `yield`, etc.).
- Ensure subset-checker help text references resolvable docs; several errors
  currently suggest `silk man silk(7)` but `silk man` does not resolve that
  query in this workspace.

### 7) Async/task ergonomics: storing task handles in collections would remove unrolled code
The current subset rejects arrays/struct fields that contain `Task(T)` (or
`Task(T)?`), which forces `slg` to unroll a fixed number of worker slots.

Upstream improvements:
- Permit `Task(T)` in arrays/vectors/struct fields (or provide a standard join
  abstraction) so worker pools can be expressed naturally.
