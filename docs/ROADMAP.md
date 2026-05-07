# `traceable` — Roadmap

> A general-purpose Rust requirements traceability tool.
> The first ShallTools crate.

---

## 1. Vision

`traceable` is a small, focused tool that lets any Rust project annotate
tests (and eventually production items) with formal requirement IDs and
generate an aerospace-grade traceability matrix. It exists because there
is no good Rust-native option for requirements traceability today, and
because anyone working under DO-178C, ASPICE, ISO 26262, IEC 62304, or
just self-imposed engineering rigor needs one.

**`traceable` is not a requirements management system.** It does not
store, edit, or version requirements themselves — those live in markdown
(or whatever format the project chooses, with a configured extractor).
`traceable` is the bridge: source code annotations on one side,
requirements markdown on the other, a traceability matrix in the middle.

The first real-world consumer is `irig106-types` (TelemetryWorks).
Bootstrapping `traceable` against a real project ensures the API is
shaped by use, not speculation.

---

## 2. Architecture

`traceable` is a Cargo workspace with three crates:

```
traceable/                                  # workspace root
├── Cargo.toml                              # workspace
├── crates/
│   ├── traceable/                          # user-facing dev-dep (lib)
│   │   ├── src/lib.rs                      # re-exports macro, registry API
│   │   └── tests/
│   ├── traceable-macros/                   # proc-macro crate
│   │   ├── Cargo.toml                      # proc-macro = true
│   │   └── src/lib.rs                      # #[req(...)] attribute
│   └── traceable-cli/                      # standalone binary
│       ├── src/
│       │   ├── main.rs
│       │   ├── config.rs                   # traceable.toml schema
│       │   ├── parse_source.rs             # syn-based source scanner
│       │   ├── parse_requirements.rs       # markdown extractor
│       │   ├── matrix.rs                   # build & emit matrix
│       │   └── report.rs                   # orphans, missing, summary
│       └── tests/                          # integration tests w/ fixtures
└── examples/
    └── irig106-style/                      # walkthrough fixture
```

### 2.1 What each crate does

| Crate              | What it is                          | How users get it                |
| ------------------ | ----------------------------------- | ------------------------------- |
| `traceable`        | Lib. Re-exports `#[req]`. Registry. | `dev-dependencies` in Cargo.toml|
| `traceable-macros` | Proc-macro impl. Internal.          | Pulled in by `traceable`        |
| `traceable-cli`    | Binary `traceable`.                 | `cargo install traceable-cli`   |

### 2.2 Compile-time vs CLI-time work

This is the central design decision and worth fixing in writing.

**The proc-macro does the minimum.** At expansion it (a) parses the
attribute args, (b) validates each ID matches the configured *syntax*
regex (e.g. `[A-Z][A-Z0-9]*-L[123]-\d+`), (c) emits the annotated item
unchanged, optionally with an `inventory::submit!` registration block.
It does **not** read any external files, parse markdown, or check that
the ID is actually defined somewhere. That keeps compile time cost
negligible and avoids the well-known fragility of doing file IO inside
proc-macro expansion.

**The CLI does the semantic work.** `traceable check` and
`traceable matrix` walk the source, parse the requirements markdown,
cross-reference, report orphans/missing/duplicates, and emit the matrix.
The CLI is what runs in CI as a gate.

If the syntax regex catches typos (it usually does — most typos break
format) and the CLI catches semantic mismatches (it always does), you
get nearly the same defect detection as compile-time semantic
validation, with none of the proc-macro fragility.

---

## 3. The `#[req(...)]` Attribute

```rust
use traceable::req;

#[req("TYPES-L3-001", "TYPES-L3-007")]
#[test]
fn rtc_from_le_bytes_masks_to_48_bits() {
    let actual = Rtc::from_le_bytes([0xFF; 6]).as_raw();
    let expected = 0x0000_FFFF_FFFF_FFFF;
    assert_eq!(actual, expected);
}
```

Behavior:

- Accepts one or more string-literal args.
- Each must match the configured ID regex; build error if not.
- Duplicate IDs within a single annotation are an error.
- The attribute may be applied to `fn`, `struct`, `enum`, `impl`,
  or `mod` items. v0.1 only formally supports `fn`. Production-code
  tracing on other items lands in v0.2.
- The expanded item is identical to the input plus an `inventory`
  registration so the CLI can correlate registered items with parsed
  source if it ever needs to.

---

## 4. Configuration: `traceable.toml`

A TOML file at the project root. The CLI looks for it; the proc-macro
does not (it only validates syntax against a hard-coded default regex
in v0.1; configurable via env var in a later version if needed).

```toml
# traceable.toml
[ids]
# Regex each requirement ID must match.
# Default: "[A-Z][A-Z0-9]*-L[123]-\\d+"
pattern = "[A-Z][A-Z0-9]*-L[123]-\\d+"

[requirements]
# Markdown files defining the canonical requirement IDs.
files = [
  "docs/requirements/L1-requirements.md",
  "docs/requirements/L2-requirements.md",
  "docs/requirements/L3-requirements.md",
]

# How requirements are extracted from markdown.
# v0.1 supports: "heading" — any heading whose text starts with an
# ID matching the pattern is treated as a requirement definition.
extractor = "heading"

[sources]
# Globs for Rust source files to scan for #[req(...)] annotations.
include = ["crates/*/src/**/*.rs", "crates/*/tests/**/*.rs"]
exclude = ["**/target/**"]

[output]
# Where the generated matrix is written.
matrix = "docs/requirements/traceability-matrix.md"
```

---

## 5. CLI Surface (v0.1)

```text
traceable check        # validate; nonzero exit on orphans/missing/typos
traceable matrix       # emit traceability-matrix.md
traceable list         # list all req IDs found in source or in reqs files
traceable --help
```

`check` is the CI gate. `matrix` regenerates the matrix on demand.
`list` is a convenience for debugging configuration.

---

## 6. Markdown Format Convention (v0.1)

The `heading` extractor recognizes any heading whose text starts with an
ID matching the pattern:

```markdown
### TYPES-L3-001 — `Rtc::from_le_bytes` masks to 48 bits

`Rtc::from_le_bytes([u8; 6])` SHALL produce a value whose `as_raw()`
result is masked to the lower 48 bits of `u64`.

**Source**: IRIG 106-22 § 6.1.2.
**Verification**: unit test.
```

The heading text after the ID (separated by space, em-dash, colon,
or `—`) is captured as the short title for the matrix.

---

## 7. Phased Plan

### Phase 0 — Bootstrap

| Deliverable                                                | Status   |
| ---------------------------------------------------------- | -------- |
| `ROADMAP.md`                                               | This file|
| `CLAUDE.md`                                                | Done     |
| `.claude/settings.json`                                    | Done     |
| Workspace skeleton (3 crates with empty `lib.rs`/`main.rs`)| Pending  |
| Apache-2.0 + MIT dual license (see § 10)                   | Pending  |
| GitHub repo `shalltools/traceable`                         | Pending  |
| Reserve all three crate names on crates.io with 0.0.1      | Pending  |

### Phase 1 — `traceable-macros` v0.1

| Deliverable                                                | Status   |
| ---------------------------------------------------------- | -------- |
| `#[req(...)]` parses and validates one+ string-literal args| Pending  |
| Default ID syntax regex                                    | Pending  |
| Duplicate-ID-in-one-annotation error                       | Pending  |
| Pass-through of the annotated item                         | Pending  |
| `inventory` registration of each annotation                | Pending  |
| `trybuild` tests for compile-fail cases                    | Pending  |
| `cargo expand` golden tests for happy path                 | Pending  |

### Phase 2 — `traceable` lib v0.1

| Deliverable                                                | Status   |
| ---------------------------------------------------------- | -------- |
| Re-export `req` macro from `traceable-macros`              | Pending  |
| Public types: `RequirementId`, `Trace`, `Annotation`       | Pending  |
| Runtime registry API: iterate registered annotations       | Pending  |
| Crate-level rustdoc with worked example                    | Pending  |

### Phase 3 — `traceable-cli` v0.1

| Deliverable                                                | Status   |
| ---------------------------------------------------------- | -------- |
| `traceable.toml` parsing (serde + toml crate)              | Pending  |
| Source scanner using `syn` + `walkdir`                     | Pending  |
| Markdown requirements extractor (`heading` mode)           | Pending  |
| Cross-reference engine                                     | Pending  |
| `traceable check` with structured exit codes               | Pending  |
| `traceable matrix` markdown emitter                        | Pending  |
| `traceable list` (req-side, source-side, both)             | Pending  |
| Integration tests with fixture projects                    | Pending  |

### Phase 4 — Documentation & Examples

| Deliverable                                                | Status   |
| ---------------------------------------------------------- | -------- |
| `README.md` — pitch, install, 5-minute walkthrough         | Pending  |
| `docs/format.md` — markdown format spec                    | Pending  |
| `docs/configuration.md` — full `traceable.toml` reference  | Pending  |
| `docs/usage.md` — CLI commands and CI patterns             | Pending  |
| `examples/irig106-style/` — fixture project showing usage  | Pending  |
| ADRs documenting key design calls (§ 2.2 in particular)    | Pending  |

### Phase 5 — Coordinated Release

| Deliverable                                                | Status   |
| ---------------------------------------------------------- | -------- |
| `traceable-macros` v0.1.0 → crates.io                      | Pending  |
| `traceable` v0.1.0 → crates.io                             | Pending  |
| `traceable-cli` v0.1.0 → crates.io                         | Pending  |
| GitHub release with binaries via `cargo dist` (or similar) | Pending  |
| Announcement: rust-lang.org users forum, /r/rust           | Pending  |

### Phase 6 — `irig106-types` Adoption (validation)

| Deliverable                                                | Status   |
| ---------------------------------------------------------- | -------- |
| Use `traceable` as dev-dep in `irig106-types`              | Pending  |
| File any usability issues against `traceable` v0.1.0       | Pending  |
| Cut `traceable` v0.1.1 (or v0.2.0) with fixes              | Pending  |

---

## 8. Stretch / Deferred (not v0.1)

- Production-code tracing (`#[req]` on `struct`/`enum`/`impl`/`mod`).
- Compile-time semantic validation against a configured req file
  (would need careful handling of `CARGO_MANIFEST_DIR` and caching).
- Output formats beyond markdown (JSON, HTML, SVG dependency graph).
- Bidirectional links to external requirements management systems
  (DOORS, Polarion, Jama, etc.) via configurable URL templates.
- IDE integration (LSP-like server for jump-to-requirement).
- Coverage gates: enforce minimum trace coverage in CI.
- Custom extractors beyond `heading` (frontmatter, YAML sidecar, …).
- `traceable init` to scaffold `traceable.toml` and starter req files.
- `traceable lint` for stylistic checks on requirement markdown.

---

## 9. Conventions

- **Test assertions**: `assert_eq!(actual, expected)` — natural order,
  not Yoda. Do not suppress related lints.
- **Naming**: descriptive and straightforward. **No nautical themes.**
- **Public API**: every public item has rustdoc with at least one
  example. Use `#![deny(missing_docs)]` in lib crates.
- **Errors**: `thiserror` for typed enums; `anyhow` only in `main.rs`
  of the CLI. No `Box<dyn Error>` in public APIs.
- **MSRV**: declare a Minimum Supported Rust Version in `Cargo.toml`
  and verify in CI.

---

## 10. Licensing

Recommendation: **Apache-2.0 OR MIT** dual license. This is the Rust
ecosystem standard, maximizes adoption, and matches what people expect
when adding a dev-dependency. (TelemetryWorks crates use Apache-2.0
only — different audience, different choice.)

---

## 11. Open Questions

1. **Crate naming on crates.io.** `traceable` is confirmed available.
   `traceable-macros` and `traceable-cli` need confirmation before
   publish.
2. **`inventory` vs `linkme` vs roll-our-own** for the registration
   side-effect of `#[req]`. `inventory` is more popular; `linkme`
   has stronger guarantees but more platform caveats. ADR candidate.
3. **MSRV target.** Probably 1.74 or later for stable inherent
   associated types and other recent niceties. Pin in Phase 0.
4. **Distribution.** `cargo dist` for prebuilt CLI binaries on
   GitHub releases? Stretch for v0.1, real for v1.0.
5. **What happens when a project uses `traceable` without a
   `traceable.toml`?** Default config, or hard error? Lean toward
   default config so adoption is one line in `Cargo.toml`.

---

## 12. References

- Parent org: ShallTools (this repo is the first ShallTools crate).
- First consumer: `TelemetryWorks/irig106-types`.
- Inspiration: aerospace-style L1/L2/L3 requirement traceability;
  pytest markers for the annotation pattern; `cargo-deny`'s CLI shape.

---

**Last updated**: 2026-05-06.
**Owner**: Joey Huckabee.
