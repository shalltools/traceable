# `traceable`

A general-purpose Rust requirements traceability tool. Annotate tests
(and eventually production items) with formal requirement IDs; generate
an aerospace-grade traceability matrix from source plus markdown.

The first crate in the [ShallTools](https://github.com/shalltools)
suite. First real-world consumer: `TelemetryWorks/irig106-types`.

---

## Status

Pre-`v0.1`. Bootstrap phase. Read `ROADMAP.md` for the phased plan and
the design decisions that shape this code.

---

## Architecture

This is a **Cargo workspace** with three members:

- `crates/traceable/` — user-facing dev-dependency. Re-exports the
  `#[req(...)]` macro and a small runtime registry API. **Lib only.**
- `crates/traceable-macros/` — proc-macro implementation.
  Internal; pulled in by `traceable`. **`proc-macro = true`.**
- `crates/traceable-cli/` — the standalone CLI (`traceable check`,
  `traceable matrix`, `traceable list`). **Bin.**

Users add `traceable = "0.1"` to `dev-dependencies` and
`cargo install traceable-cli` for the CLI. They never import
`traceable-macros` directly.

---

## Build & Test Commands

```bash
# Workspace-wide
cargo fmt --all
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace

# Per crate
cargo test -p traceable
cargo test -p traceable-macros
cargo test -p traceable-cli

# Compile-fail tests for the macro
cargo test -p traceable-macros --test compile_fail

# Run the CLI locally against the workspace's own example project
cargo run -p traceable-cli -- check  --config examples/calculator/traceable.toml
cargo run -p traceable-cli -- matrix --config examples/calculator/traceable.toml
```

---

## Architectural Rules

- **Proc-macro does the minimum at compile time.** Validate ID syntax
  via regex, error on duplicates within one annotation, pass through
  the item, register in `inventory`. **Do not** read external files,
  parse markdown, or check IDs against the canonical list inside the
  proc-macro. The CLI does that work.
- **CLI does the semantic work.** Source scanning (via `syn`),
  markdown parsing, cross-referencing, matrix emission.
- **Single source of truth.** A `#[req(...)]` annotation is the only
  way to record a trace from source. The CLI never invents traces or
  modifies source.
- **The CLI is read-only on user code.** It reads source, reads
  requirements markdown, writes the matrix. It does not edit Rust files.
- **Public API is rustdoc'd.** `#![deny(missing_docs)]` on all lib
  crates. Every public item has at least one example.
- **No I/O in `traceable-macros`.** The proc-macro never opens files
  at expansion time. (See § 2.2 of `ROADMAP.md` for why.)

---

## Code Style

- **Test assertion order**: `assert_eq!(actual, expected)` — natural
  order, not Yoda. Do not suppress related lints.
- **Naming**: descriptive, straightforward. **No nautical themes.**
- **Errors**: `thiserror` for typed enums in libs; `anyhow` only in
  `main.rs`. No `Box<dyn Error>` in public APIs.
- **MSRV**: declared in workspace `Cargo.toml` and verified in CI.
- **Imports**: group as `std`, third-party, then `crate::`. One blank
  line between groups.
- **Doc comments**: `///` on public items, `//!` for module-level docs.

---

## The `#[req(...)]` Attribute (canonical example)

```rust
use traceable::req;

#[req("CALC-L3-001", "CALC-L3-004")]
#[test]
fn divide_by_zero_returns_error() {
    let actual = divide(10, 0);
    let expected = Err(CalcError::DivisionByZero);
    assert_eq!(actual, expected);
}
```

The macro:

1. Parses one or more string-literal args.
2. Validates each matches the configured ID syntax (default regex
   `[A-Z][A-Z0-9]*-L[123]-\d+`). Build error if not.
3. Errors on duplicates within one annotation.
4. Passes through the item unchanged, plus an `inventory::submit!`
   block for the registry.

v0.1 formally supports the attribute on `fn` only. Other item kinds
(struct, enum, impl, mod) land in v0.2.

---

## Configuration: `traceable.toml`

Project-root TOML file. Read by the CLI. (Not read by the macro in v0.1.)

Schema lives in `crates/traceable-cli/src/config.rs` and is documented
in `docs/configuration.md`. The full reference is in `ROADMAP.md` § 4.

Key keys:

- `[ids] pattern` — regex each ID must match.
- `[requirements] files` — markdown files defining canonical IDs.
- `[requirements] extractor` — how IDs are extracted (v0.1: `heading`).
- `[sources] include` / `exclude` — globs for source scanning.
- `[output] matrix` — where the matrix is written.

---

## CLI Commands (v0.1)

```text
traceable check        # validate; nonzero exit on any issue
traceable matrix       # emit traceability-matrix.md
traceable list         # list IDs found in source / requirements
traceable --help
```

`check` is the CI gate. `matrix` regenerates on demand. `list` is for
debugging configuration.

---

## When Adding a Feature

1. Update `ROADMAP.md` if scope is changing.
2. If the feature changes the public API of `traceable` or the
   `#[req]` attribute, write an ADR in `docs/adr/`.
3. Add tests:
   - Macro features → `compile_fail` tests + `cargo expand` golden.
   - CLI features → integration tests using fixture projects under
     `crates/traceable-cli/tests/fixtures/`.
4. Update `examples/calculator/` if the change affects user-facing
   workflow.
5. Update `CHANGELOG.md`.

---

## Out of Scope (v0.1)

- Editing or generating requirement markdown — `traceable` is
  read-only on requirements.
- Storing requirements (this is not a requirements management system).
- IDE integrations, LSP, jump-to-definition.
- HTML, JSON, or graph-format matrix output (markdown only in v0.1).
- Bidirectional links to DOORS, Polarion, Jama, etc.
- Custom extractors beyond `heading`.
- Production-code tracing (non-`fn` items).
- Compile-time semantic validation (only syntactic in v0.1).

If a request would pull `traceable` into being a requirements
*management* system, push back — this tool is a *bridge*, not a store.

---

## What NOT to Do

- Don't add file IO to `traceable-macros`. (See architectural rules.)
- Don't make the proc-macro depend on `traceable-cli` or
  `traceable-core`-style heavyweight crates.
- Don't expose internal types from `traceable-macros` through the
  `traceable` re-export — the macro crate is an implementation detail.
- Don't take a dependency on a markdown crate in `traceable-macros`.
- Don't bake project-specific assumptions into defaults; if a value
  could reasonably differ between projects, make it configurable.
- Don't widen `v0.1` scope. Stretch goals are in `ROADMAP.md` § 8 for
  a reason.

---

## Sibling Repos

- `shalltools/*` — future ShallTools crates (not yet created).
- `TelemetryWorks/irig106-types` — first `traceable` consumer.
