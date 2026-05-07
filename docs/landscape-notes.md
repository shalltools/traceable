# `traceable` — Landscape Notes

Reference notes from a survey of existing requirements traceability and
ADR tooling. Used to inform `traceable`'s design and positioning. Not a
specification — a working document we revise as we learn more.

---

## TL;DR — The Hard Truth First

**`mantra` (`mhatzl/mantra`) is uncomfortably close to what we designed.**
Same language (Rust), same target audience (safety-critical Rust devs),
same `#[req(req_id)]` attribute macro, same workspace shape (CLI +
macros + supporting crates), MIT licensed, active, presented at RustWeek
2025. Before committing more time to `traceable`, we need a clear answer
to: **what does `traceable` do that `mantra` doesn't?**

Three honest options:
1. Continue with `traceable` and pick a real differentiator (see § 7).
2. Contribute to `mantra` instead.
3. Reframe `traceable` around a niche `mantra` doesn't cover.

These notes exist to make that decision an informed one.

---

## 1. The Closest Peer — `mantra`

- **Repo**: <https://github.com/mhatzl/mantra>
- **Author**: Manuel Hatzl
- **License**: MIT
- **Status**: Active. Talk at RustWeek 2025.
- **Crates published**: `mantra`, `mantra-rust-macros`,
  `mantra-rust-trace`, `mantra-schema`, `mantra-lang-tracing`.

What it does:
- Rust CLI for managing requirement traces, coverage, reports.
- Uses `tree-sitter` to scan source — **language-agnostic**, not just Rust.
- Stores data in **SQLite** (`sqlite://mantra.db?mode=rwc` by default).
- Markdown wiki for requirements, parent/child via dot notation
  (`store.manufacturer`).
- HTML reports with custom templates.
- Manual reviews for non-functional reqs that can't be traced to source.
- Embedded support via `defmt` feature for coverage logs on devices.

The macro:
```rust
use mantra_rust_macros::{req, reqcov};

#[req(req_id)]
fn some_fn() {
    reqcov!(function_like_trace);
}
```

What's notable:
- The `req`/`reqcov` split is interesting: `req` is the static trace
  (where in source the requirement is implemented); `reqcov!()` is the
  runtime coverage breadcrumb. Our v0.1 plan didn't separate these.
- `[req(<id>)]` syntax (without quotes) treats the ID as a Rust
  identifier. Our plan used string literals. Theirs is more
  ergonomic but constrains ID format more rigidly.
- The macro can be applied to many item kinds — `fn`, `const`, `type`,
  `struct`, `mod`, `trait`, `trait fn`. Our v0.1 said `fn` only.

What's missing or weaker (where we could differentiate):
- No mention of **L1/L2/L3 hierarchy** as a first-class concept.
  Their model is flat with optional dot-namespacing.
- SQLite as the data store is *more* than most Rust projects want.
  Our plan keeps everything in markdown and Cargo files.
- Configuration (`mantra.toml`) has many options but no obvious
  workflow for aerospace/automotive certification standards.
- Tree-sitter dependency requires a C compiler — extra installation
  friction in air-gapped or restricted environments.

---

## 2. Other Rust-Native — `SARA`

- **Repo**: <https://github.com/cledouarec/sara>
- **Author**: Christophe Le Douarec
- **License**: Apache 2.0
- **Status**: Active (HN launch Jan 2026, dev.to coverage Jan 2026).
- **Crates**: `sara-cli` and `sara-core`.

Different angle from both `mantra` and `traceable`:
- **No source code annotations.** Pure markdown + YAML frontmatter.
- **Knowledge graph model** with 9 document types: `solution`,
  `use_case`, `scenario`, `system_requirement`, `system_architecture`,
  `software_requirement`, `software_design_document`, etc.
- Bidirectional links auto-inferred (define one direction, get the
  reverse for free).
- Validates: orphans, broken links, circular dependencies.
- Has multiple repository support — one config file references
  multiple repos.

Example frontmatter:
```yaml
---
id: "SCEN-001"
type: scenario
name: "User Login"
refines:
  - "UC-001"
derives:
  - "SYSREQ-001"
---
```

CLI traversal:
```text
sara query SCEN-001 --downstream
SCEN-001: User Login Scenario
├── SYSREQ-001: Response Time Requirement
│   └── SYSARCH-001: Authentication Architecture
│       └── SWREQ-001: JWT Token Generation
│           └── SWDD-001: Auth Service Implementation
```

What's notable:
- The **9 document types as a baked-in vocabulary** is closer to what
  aerospace expects than `mantra`'s flat model.
- **No code annotations** is a deliberate choice: focuses on the
  requirements-side traceability, leaves implementation links to other
  tools or to plain ID references in code.
- The **knowledge graph** is the data model — much richer than a
  matrix. Cycle detection comes for free.

What we could borrow:
- The 9-type vocabulary as a default schema.
- Bidirectional inference.
- Cycle and orphan detection patterns.

---

## 3. The Established Player — `OpenFastTrace` (OFT)

- **Repo**: <https://github.com/itsallcode/openfasttrace>
- **License**: Apache-2.0 (the gist says Java-based)
- **Language**: Java
- **Maturity**: Most mature open-source option. `4.2.0` as of late 2025.
- **Plugins**: Maven, Gradle.

Concepts worth stealing:
- **Specification item chain** with artifact types:
  `feat → req → dsn → impl → (utest | itest)`. Some items terminate
  the chain (no further coverage required). This maps cleanly onto
  aerospace: **stakeholder need → system req → design → impl → test**.
- **ID format with revision**: `req~tracing.deep-coverage~1`. The `~1`
  is a **revision number**. When a parent's revision increments,
  children that referenced the old revision are flagged as stale.
  This is genuinely useful for change management — neither `mantra`
  nor our plan included it.
- **Cross-project requirements via Maven repositories.** A "Software
  Architecture Design" project publishes its requirements as a Maven
  zip; downstream projects import them and prove coverage. Same
  pattern would work in Rust with crates.io or a private registry.
- **Multiple report formats**: plain text (great for grep/CLI piping),
  HTML, XML (`aspec` augmented specobject for tool interop).
- **`ReqM2 SpecObject` interop** for migrating from older toolchains.

What's worth NOT copying:
- Java + Maven/Gradle workflow doesn't match our user.
- The `~name~rev` ID format is dense and hard to type.

---

## 4. The Most Full-Featured — `StrictDoc`

- **Repo**: <https://github.com/strictdoc-project/strictdoc>
- **Language**: Python
- **License**: Apache-2.0
- **Status**: Very active (`0.16.x` series, 2026 releases).

Concepts worth knowing about:
- **DSL approach** — `.sdoc` files with a custom grammar via TextX.
  Recently added Markdown support.
- **UID / MID / HASH model**:
  - `UID` — human-readable, may change over time.
  - `MID` — stable machine ID, doesn't change.
  - `HASH` — hash of requirement content; auto-calc.
  - Solves the rename problem cleanly.
- **Web server + static HTML export.** Has its own server with full
  traceability graph visualization. PDF via HTML2PDF.
- **ReqIF import/export** for interop with DOORS, Polarion, etc.
- **Test report integration** — reads JUnit XML and links results
  back to requirements.
- **Tree-sitter for source parsing** (Python, C, eventually Rust).
  Requirements found in source code via SPDX-style tags.
- Used in **Linux kernel ELISA** safety effort.

Why not just use this:
- Python, not Rust.
- DSL friction — the `.sdoc` format is its own thing to learn.
- Heavier than what most Rust projects want.

What we could borrow:
- The **UID/MID/HASH triplet** for stable references.
- **JUnit XML test report integration** as a one-way data flow.
- **Source-side tags + sidecar files** as a complementary model
  (some teams want neither pure-source nor pure-markdown).

---

## 5. The DSL Approach — `TRLC` (BMW)

- **Repo**: <https://github.com/bmw-software-engineering/trlc>
- **License**: GPL-3.0 (note: copyleft, not adoption-friendly).
- **Language**: Python (reference implementation).
- **Companion**: `LOBSTER` (AGPL-3.0) for actual traceability.

What it is:
- A **domain-specific language** for requirements.
- `.rsl` files = schema; `.trlc` files = requirement data.
- Hand-written lexer + recursive descent parser (very precise error
  locations).
- **User-defined check rules** with custom error messages, e.g.:
  ```
  checks Requirement {
      top_level == true or derived_from != null,
      error "linkage incorrect",
      """You must either link this requirement to other requirements
      using the derived_from attribute, or you need to set top_level
      to true."""
  }
  ```
- VS Code extension. Bazel rules. Static analysis with type checking.
  Uses CVC5 SMT solver for `--verify` (constraint checking).

Concepts worth thinking about:
- **Schema-first requirements**. Define what fields a requirement has,
  what types they take, what constraints they must satisfy. The data
  is then validated against that schema.
- **User-definable validation rules** with friendly error messages.
- **Type system for requirements metadata.** Requirements can have
  decimal, integer, string, enumeration, tuple, array, reference
  fields — all type-checked.

Why not just use this:
- Python only (reference impl).
- GPL-3.0 makes downstream adoption painful.
- Heavyweight schema-first model imposes a learning curve.
- Designed for a BMW-scale organization.

What we could borrow:
- The idea that **requirements have schemas**, not just IDs and titles.
- User-definable validation rules.

---

## 6. Other Notable Mentions

- **Reqflow** (C++, MIT). Regex extraction from `.docx`/`.txt`.
  Generates HTML/CSV/text matrices. Old but the regex-from-anything
  approach is clever for legacy doc situations.
- **Sphinx-Needs** (Python). Sphinx extension. Used in ISO 26262
  contexts. If a team is already on Sphinx, this is the path.
- **Open-Needs** (Python). Multi-tool docs-as-code (Sphinx + AsciiDoctor
  + MkDocs).
- **OSRMT / aNimble** (Java). Traditional GUI-driven requirements
  management tools. Not relevant for our audience but mentioned for
  completeness.
- **FRET** (Electron + formal methods). Specialized natural language
  for safety-critical requirements. Different problem.

---

## 7. Where Could `traceable` Differentiate?

If we proceed, the design needs a clear answer to "why not `mantra`."
Here are real options. Each is a coherent positioning we could pick.

### Option A — Aerospace-grade rigor as a first-class concept
- **L1/L2/L3 hierarchy** baked in as required structure.
- **Artifact-type chain** like OFT (`feat → req → dsn → impl → test`)
  but with idiomatic Rust ergonomics.
- **Revision numbers** like OFT (`REQ-001@2`) for change detection.
- **DO-178C / ASPICE / ISO 26262 / IEC 62304 reference templates**
  in the docs, not bolted on.
- Compile-time validation that L3 IDs are well-formed under the chosen
  hierarchy template.

This is "the requirements traceability tool that *speaks* aerospace,"
where `mantra` is "the requirements traceability tool that *can do*
aerospace if you set it up right."

### Option B — Markdown-only, no database, fewer dependencies
- No SQLite. No tree-sitter (Rust-only via `syn`).
- One config file (`traceable.toml`), markdown reqs, Rust source —
  that's all. The output is markdown.
- Easier to install in air-gapped / restricted environments.
- "The lightest plausible requirements traceability tool that still
  meets aerospace expectations."

`mantra` aims for "feasible middle ground." `traceable` (Option B)
aims for "literally the simplest thing that could work."

### Option C — Schema-first traceability for Rust
- Requirements have **typed fields** (à la TRLC) defined in a project
  schema, not just (id, title, body).
- The macro and CLI both validate against the schema.
- **Lifts TRLC's good ideas into idiomatic Rust** without the
  copyleft licensing or DSL cost.

This is the most ambitious option and the most distinctive — there's
no Rust-native schema-first requirements tool.

### Option D — A standards-interop tool
- Native ReqIF read/write so teams can move requirements between
  DOORS, Polarion, Jama, and `traceable` without lock-in.
- Integration with JUnit XML test reports (à la StrictDoc).
- "The Rust-native bridge between heavyweight RM systems and your
  source code."

### Option E — Don't. Contribute to `mantra` instead.
- Submit PRs for L1/L2/L3 hierarchy support, aerospace templates,
  whatever else we'd want.
- Keep `irig106-types` as a real-world test case.
- Move on to other ShallTools things.

---

## 8. Recommendation

**Top pick: Option A + B hybrid.** Aerospace-grade rigor as the
positioning, with markdown-only-no-database as the implementation
philosophy. This is internally consistent and gives us a clear
two-line pitch:

> `traceable` — Rust-native requirements traceability for safety-critical
> projects. L1/L2/L3 hierarchy, no database, no DSL, just markdown and
> code.

**Second pick: Option E.** Honest acknowledgment that `mantra` is
already 80% of what we'd build, and our time is better spent on
different ShallTools (not yet identified) plus contributions back.

The decision should not be made under pressure. **Worth sitting on for a
day**, especially given the real cost of building a parallel tool. If
the answer is still A+B after a day, build it. If it's E, that's a fine
outcome and we move on.

---

## 9. Open Questions to Resolve Before Building

1. Is the user willing to live with `mantra` being the "default
   recommendation" for most projects, with `traceable` as the choice
   for projects with formal hierarchy/template needs?
2. Does Option C (schema-first) interest us enough to justify the
   extra design work? It would be more distinctive but more work.
3. Is `mantra`'s `[req(id)]` (Rust ident, no quotes) actually better
   than our `#[req("ID-001")]` (string literal)? Worth deciding —
   their pattern is more ergonomic but constrains valid IDs.
4. Do we want runtime coverage breadcrumbs (mantra's `reqcov!`) at
   all? Static traces alone might be enough for our use cases.
5. What's the smallest credible v0.1 under Option A+B? L1/L2/L3 in,
   schema-first out, ReqIF deferred, runtime coverage deferred?

---

## 10. References

- `mantra` — <https://github.com/mhatzl/mantra>
- `SARA` — <https://github.com/cledouarec/sara>
- `OpenFastTrace` — <https://github.com/itsallcode/openfasttrace>
- `StrictDoc` — <https://github.com/strictdoc-project/strictdoc>
- `TRLC` — <https://github.com/bmw-software-engineering/trlc>
- `LOBSTER` — <https://github.com/bmw-software-engineering/lobster>
- `Reqflow` — <https://goeb.github.io/reqflow/>
- `Sphinx-Needs` — see linked gist below.
- Comprehensive gist of open-source RM tools by stanislaw (StrictDoc
  author): <https://gist.github.com/stanislaw/aa40eb7de9f522ad482e5d239c435ff8>

---

**Last updated**: 2026-05-06.
**Status**: Working document. Revise as more is learned.
