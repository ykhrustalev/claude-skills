---
name: x-review
description: Review the pending change for simple architecture, soundness, and correctness, plus language-specific rules (Rust, Bash). Use when the user asks for an x-review, or invokes /x-review.
---

# x-review

## Scope

Unless the user specifies otherwise, review **the current branch's diff against `main`, plus any uncommitted (staged and unstaged) changes relevant to that work**. Concretely:

- Base: `main` (fall back to `master` if `main` doesn't exist).
- Range: `git diff main...HEAD` for committed work on the branch, plus `git diff HEAD` for uncommitted changes. Combine both.
- Untracked files that are clearly part of the in-progress change should be included; ignore unrelated local cruft.
- If the user names a different base (e.g. "review against `develop`"), a specific commit range, or a single file/PR, honor that instead.

## How the rules are organized

Rules are split into **general** (apply to every change) and **language-specific** (apply only to files in that language). Detect the languages present in the diff and run the matching language sections; skip the rest.

Review each rule as a **separate pass** — do not interleave findings across rules.

---

## General rules (all languages)

### G1. Simple architecture
- Is the design as simple as it can be for what the change actually needs to do?
- Flag unnecessary abstractions, premature generalization, speculative flexibility, or layers that exist "just in case".
- Flag duplicated concepts, parallel hierarchies, or anything that adds structure without paying for itself.
- Prefer collapsing over adding: would removing a layer, helper, or option make the code clearer without losing anything load-bearing?

### G2. Soundness
- Does the design hold up under the invariants it implicitly assumes? Call out hidden assumptions.
- Concurrency, ordering, lifecycle, error propagation, partial-failure, and idempotency: are they handled or quietly ignored?
- Are boundaries (user input, external APIs, untrusted data) validated? Are internal boundaries trusted appropriately?
- Are there race conditions, TOCTOU issues, unbounded growth, or resource leaks?

### G3. Correctness
- Does the code do what it's supposed to do, on the golden path and on the edges?
- Off-by-one, null/empty/missing, overflow, timezone/locale, encoding, floating-point, type coercion.
- Do tests actually exercise the behavior being changed, or just call the code? Are there obvious cases missing?
- Does the change match its stated intent (commit message, PR description, conversation)?

### G4. Comments — concise and value-adding
- Flag comments that restate what the code obviously does.
- Flag stale comments that no longer match the code, and TODO/FIXME left without context or owner.
- Flag multi-line block comments and docstrings that pad without adding insight.
- Keep comments that explain *why*, document non-obvious invariants, or warn about subtle bugs/workarounds.
- For each occurrence: cite the line and either propose a tighter rewrite or recommend deletion.

---

## Rust-specific rules

Apply when the diff touches `.rs` files.

> **Never propose `panic!()`, `unwrap()`, `expect()`, `todo!()`, or `unreachable!()` as a fix.** Not in production code, not in tests, not in match arms, not "to handle the unhandled variant". If a variant is genuinely unreachable, prove it via the type system (refactor the enum, use `#[non_exhaustive]` deliberately, or split into a narrower type). If it's reachable, return a `Result` with a typed error. The review must steer code *away from* panicking constructs, never toward them.

### R1. Iteration style — no raw `for` loops
- **Default: flag every `for` loop in the diff.** Walk the diff and list each one — do not pre-filter based on whether the rewrite "feels cleaner". Showing the rewrite is the point; the reader decides.
- For each loop, identify the pattern in the body and propose the matching combinator. Use this table — pick the most specific match, not just `for_each`:
  - **Accumulating into a variable** (`let mut acc = init; for x in xs { acc = f(acc, x); }`) → `xs.iter().fold(init, f)`. With `?` inside → `try_fold`.
  - **Summing / counting / producing a single number** → `.sum()` / `.count()` / `.product()` (don't reach for `fold` when a named reducer exists).
  - **Building a new collection** (`let mut v = Vec::new(); for x in xs { v.push(g(x)); }`) → `xs.iter().map(g).collect::<Vec<_>>()`. With a filter → `.filter(...).map(...).collect()` or `.filter_map(...)`.
  - **Building a `Result<Vec<_>, _>`** (loop body uses `?`) → `.map(...).collect::<Result<Vec<_>, _>>()` or `.try_fold`.
  - **Flattening nested iteration** (nested `for` over `Vec<Vec<_>>` or `for x in xs { for y in x.ys() { ... } }`) → `.flat_map(...)` / `.flatten()`.
  - **Conditionally pushing** (`if pred(x) { v.push(x) }`) → `.filter(pred).collect()` or `.filter(pred).cloned().collect()`.
  - **Both filter and transform** (`if let Some(y) = f(x) { v.push(y) }`) → `.filter_map(f).collect()`.
  - **Pure side effects** (`for x in xs { logger.log(x); }`) → `xs.iter().for_each(|x| logger.log(x))`. With `?` → `try_for_each`.
  - **Finding the first match** (early `break` with a captured value) → `.find(...)` / `.find_map(...)` / `.position(...)`.
  - **Boolean reductions** ("does anything match?" / "do all match?") → `.any(pred)` / `.all(pred)`. Not `fold`.
  - **Min/max** → `.min()` / `.max()` / `.min_by_key(...)` / `.max_by(...)`.
  - **Building a `HashMap`/`BTreeMap`** (`for (k, v) in pairs { map.insert(k, v); }`) → `pairs.into_iter().collect::<HashMap<_, _>>()`. Grouping → `.fold(HashMap::new(), |mut m, x| { m.entry(k(x)).or_default().push(x); m })`.
  - **Parallel iteration over two collections** (`for i in 0..xs.len() { f(&xs[i], &ys[i]); }`) → `xs.iter().zip(&ys).for_each(...)`.
  - **Windowed / pairwise access** (`for i in 1..xs.len() { f(&xs[i-1], &xs[i]); }`) → `xs.windows(2)` or `.tuple_windows()` (if `itertools` is a dep).
  - **Index needed alongside value** → `.enumerate()`.
  - **`itertools`-only constructs** (`group_by`, `chunks`, `tuple_windows`, `dedup_by`, `sorted`) — suggest **only if `itertools` is already in `Cargo.toml`** (same rule as R4: never recommend adding a dependency).
- Only these exceptions skip the flag (state which one applies):
  1. The loop body contains an early `break` with a value or a `return` that exits the *enclosing function* (not just the loop) — and the equivalent `find` / `find_map` / `position` rewrite is materially less readable.
  2. The loop uses labeled `break`/`continue` across nested loops.
  3. The iterand is `..` or `..=` over an integer range used purely for its index (e.g. `for i in 0..n { a[i] = b[i] + c[i]; }`) **and** the body indexes multiple independent collections in lockstep where `zip` would obscure intent.
- "It's a hot path" is **not** an exception at review time — note the perf consideration alongside the rewrite and let the author decide; do not pre-suppress the finding.
- Output for each occurrence: `file_path:line_number`, the original loop (1–3 lines), the iterator rewrite, and any allocation/perf note.
- If after walking the diff there are genuinely zero `for` loops, write "No `for` loops in diff." — do not write "No findings" without first confirming you scanned for the token `for ` in `.rs` hunks.

### R2. Tests — no `.unwrap()`, return `Result`
- Flag every `.unwrap()`, `.expect()`, or `panic!` inside `#[test]` / `#[tokio::test]` functions.
- Tests should be declared `fn name() -> Result<(), Box<dyn std::error::Error>>` (or a project-specific alias) and use `?` for error propagation.
- For each occurrence: cite the line and show the `?`-based rewrite, including the test signature change.

### R3. Tests — prefer the conveyor (table/builder) approach
- Flag groups of tests that repeat the same setup with only input/expected differences. They should be consolidated into a single table-driven test (a `cases` array iterated over) or a shared builder/fixture.
- Flag bespoke per-case helper functions that exist only because the same pattern is copy-pasted across tests.
- For each cluster: name the duplicated tests, sketch the consolidated `cases` table or builder, and call out which assertions become parameters.

### R4. Tests — suggest `rstest` *only* if already a dependency
- Check `Cargo.toml` / workspace `Cargo.toml` for `rstest` under `[dev-dependencies]`. If absent, **do not suggest it** — never recommend adding a new dependency.
- Apply this rule **after** R3: first consolidate copy-pasted tests into a conveyor (table/builder) shape, then — if `rstest` is already available — suggest converting the conveyor into `#[rstest]` with `#[case(...)]` attributes or fixtures.
- The order matters: don't jump to `#[rstest]` while tests are still duplicated. Conveyor first, `rstest` as the polish layer.
- For each suggestion: cite the file, show the `#[rstest]` + `#[case]` (or `#[fixture]`) rewrite, and keep the `Result`-returning signature from R2.

### R5. Custom errors — suggest `thiserror` *only* if already a dependency
- Check `Cargo.toml` / workspace `Cargo.toml` (including `[workspace.dependencies]`) for `thiserror`. If absent, **do not suggest it** — never recommend adding a new dependency.
- If `thiserror` is available, flag custom error types in the diff that hand-roll `impl std::error::Error`, `impl Display`, or `impl From<...>` boilerplate that `#[derive(thiserror::Error)]` plus `#[error("...")]` / `#[from]` would generate.
- Also flag ad-hoc error patterns that should become a typed error: `Box<dyn Error>` in library/internal APIs (tests are exempt per R2), stringly-typed errors (`String`, `&'static str`, `anyhow::anyhow!("...")` in non-application code), or enums missing `#[from]` conversions that force manual `map_err` at every call site.
- For each occurrence: cite the file/line, show the `#[derive(thiserror::Error, Debug)]` rewrite with appropriate `#[error("...")]` messages and `#[from]` attributes, and note which manual impls or `map_err` calls collapse away.

---

## Bash-specific rules

Apply when the diff touches `.sh`, `.bash` files, or files with a bash shebang.

### B1. Shebang — `#!/usr/bin/env bash`
- The first line of every executable bash script must be `#!/usr/bin/env bash`.
- Flag `#!/bin/bash`, `#!/bin/sh`, or missing shebangs in executable scripts.
- For each occurrence: cite the line and show the correction.

### B2. Strict mode — `set -euo pipefail`
- Every bash script must include `set -euo pipefail` near the top (after the shebang and any header comment).
- Flag scripts missing it, or that only set a subset (e.g. `set -e` alone).
- If `IFS` manipulation matters for the script's logic, also recommend `IFS=$'\n\t'`.
- For each occurrence: cite the line and show the correction.

### B3. Conditionals — prefer `[[ ... ]]`
- Flag `[ ... ]` (POSIX test) and bare `test` calls; replace with `[[ ... ]]`.
- Flag string comparisons that use `=` inside `[ ]` where `[[ ]]` would allow pattern matching, glob, or regex (`==`, `=~`).
- For each occurrence: cite the line and show the `[[ ]]` rewrite.

---

## How to run the review

1. Inspect the diff (`git diff main...HEAD`, `git diff HEAD`, `git status`, and the PR description if available) before reading individual files.
2. Read changed files in full where context matters — don't review hunks in isolation.
3. Determine which language sections apply based on file extensions / shebangs in the diff.
4. **Review each applicable rule as a separate pass.** Finish one rule fully before moving to the next.
5. For each rule, produce a dedicated section in the output with:
   - The rule id and name as a heading (e.g. `### R1. Iteration style`).
   - A list of findings, each citing `file_path:line_number`, stating the issue in one sentence, and showing the smallest fix (code snippet where useful).
   - A short **Fix plan** sub-list: ordered, concrete steps to address that rule's findings.
   - If the rule has no findings, write "No findings." — do not pad.
6. End with a consolidated **Verdict**: ship / ship with fixes / needs rework, plus a single ordered fix plan across all rules (highest-impact first).

## What to skip

- Style nits already handled by formatters/linters.
- Naming preferences that aren't actively misleading.
- Refactors unrelated to the change.
- Praise. Reviews should surface problems; silence on a file means it's fine.
