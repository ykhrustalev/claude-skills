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
- Flag any `for` loop that could be expressed with `.iter()` / `.into_iter()` / `.iter_mut()` plus combinators (`map`, `filter`, `fold`, `collect`, `try_fold`, `flat_map`, etc.).
- Exceptions: loops with early `break`/`return` that don't map cleanly to combinators, side-effectful loops where a functional rewrite is meaningfully less clear, or hot paths where the iterator form is provably worse.
- For each occurrence: cite the line, show the iterator-based rewrite, and note any allocation/perf implications.

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
