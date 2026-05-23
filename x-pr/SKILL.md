---
name: x-pr
description: Generate a standardized commit message for the pending change with a Conventional Commits prefix (feat:/fix:/etc.), a concise headline, and a description grounded in the actual diff. References a ticket if one appeared in the conversation. Use when the user asks for an x-pr, or invokes /x-pr.
---

# x-pr

Produce a single, well-formed commit message for the current change and either create or update the PR on GitHub. Do **not** run `git commit` (the user controls local commits) — but **do** run `gh pr create` / `gh pr edit` without prompting once the branch is ready.

## Steps

1. **Sync with `origin/main` first.** The branch must be up-to-date with fresh `origin/main` before producing the commit message or editing a PR:
   - `git fetch origin main` to get the latest.
   - Check `git rev-list --left-right --count origin/main...HEAD` — if `origin/main` has commits the branch doesn't have, the branch is behind.
   - If behind, propose one of:
     - `git rebase origin/main` (preferred — keeps history linear), or
     - `git merge origin/main` (when the branch is shared and rewriting history would break collaborators).
   - Show the command, explain which to pick based on whether the branch is shared (has been pushed and others may have pulled it), and **stop** until the user has integrated. Do not auto-run rebase or merge — these can produce conflicts that need human judgment.
   - If conflicts surface, surface them and stop; do not resolve speculatively.
2. Inspect the change:
   - `git status` (no `-uall`)
   - `git diff` (staged + unstaged)
   - `git log -n 5 --oneline` to match repo style if it's distinctive
3. Identify the dominant intent of the change and pick **one** Conventional Commits prefix:
   - `feat:` — new user-visible capability
   - `fix:` — corrects broken behavior
   - `refactor:` — restructure without behavior change
   - `perf:` — performance improvement
   - `docs:` — documentation only
   - `test:` — tests only
   - `chore:` — tooling, deps, build, non-functional
   - `revert:` — reverts a prior commit
   When in doubt between `feat` and `refactor`, ask: does an external observer see a difference? If yes → `feat`.
4. Scan the conversation history for a ticket reference (e.g. `ABC-123`, `#456`, `JIRA-789`, a URL containing an issue id). If found, include it. If not, omit — never invent one.
5. Check the branch name with `git rev-parse --abbrev-ref HEAD`. It **must** match `ykhrustalev/<slug>`:
   - `<slug>` is kebab-case, derived from the headline (or the ticket id when present, e.g. `ykhrustalev/abc-123-add-retry-backoff`).
   - If the current branch is `main`, `master`, or doesn't match the pattern, propose a rename command and **stop** before producing the commit message:
     `git switch -c ykhrustalev/<slug>` (if on main/master) or `git branch -m ykhrustalev/<slug>` (to rename current).
   - Do not run the rename automatically — show the command and let the user run it.
   - Once the branch is correctly named, continue to the output step.
6. Ensure the branch is pushed: `git push -u origin HEAD` (use `--force-with-lease` if the prior step rebased). Required so a PR can be opened against it.
7. Check whether a PR already exists for this branch with `gh pr view --json number,title,body,headRefName 2>/dev/null`.
   - **If a PR exists**, treat it as auto-created and **fix it in place**: update title and body to the standardized format using `gh pr edit <number> --title "..." --body "..."`. Run this without asking.
   - **If no PR exists**, **create one without asking** using `gh pr create --title "..." --body "..."`. Do not stop to confirm — the user invoked `/x-pr`, that is the confirmation.
   - The PR title uses the `<prefix>: <headline> [(<ticket>)]` format from the commit headline.
   - The PR body must be **as concise as the commit description**. See "PR body rules" below.
   - After running, print the resulting PR URL.

## PR body rules

The PR body should be *concise but scannable*. A reviewer reads it in 15 seconds and walks away knowing (a) **why** the change exists and (b) **what** concrete things it touches. Dense one-paragraph bodies fail (a); multi-section essays fail both.

### Required shape

```
**Problem**
<1–3 sentences explaining what was wrong / missing / needed.>

**Solution**
- <concrete change 1 — one line>
- <concrete change 2 — one line>
- <concrete change 3 — one line>

**Testing** (optional)
- <how the change was verified — manual steps, new tests, or commands run>

Closes #123
```

- **Problem** — 1–3 sentences. The motivation. Not the diff.
- **Solution** — 2–6 bullets, each one short line naming a *conceptual* change (a new behavior, a renamed concept, a new flag, a removed code path). Use bullets whenever the PR has more than one distinct change. Skip the section only when the PR really is one-thing.
- **Testing** — *optional*. Include only when verification is non-obvious: a manual repro, a new integration test worth pointing at, a command the reviewer should run. Skip entirely when "unit tests cover it" is the whole story. Never write "Tested manually" with no detail.
- **`Closes #123`** — on its own line, at the very end, blank line above it. Only when the PR actually closes the ticket. Use `Refs #123` for related-but-not-closing. Never invent a number.

### Length

- Soft target: 6–15 lines. Hard cap: ~20.
- One dense paragraph is **worse** than the same content split into Problem + Solution bullets — even when it's shorter. If a sentence contains "; furthermore," "additionally," or three semicolons, it's a bullet list in disguise.
- Skip sections that don't earn their place. A trivial PR can be just a 1-line Problem + `Closes #x`. An almost-self-evident PR can be just title + `Closes #x`. Testing is optional — only include when it adds something.

### Forbidden

- **No "Notes" / "Background" / "History" sections** unless the repo's PR template requires them. (A `**Testing**` block is allowed — see Required shape — but only when it adds substance; "Test plan" as a checklist of TODOs is not.)
- **No mention of prior PRs, branch renames, session context, or prior attempts.**
- **No Claude / AI mention anywhere.** Not in title, body, footer, trailer, bullet, or parenthetical. No "🤖 Generated with Claude Code", no "Drafted by…", no emoji robot, no link to claude.com / claude.ai, no reference to the chat session, prompts, or this skill. The PR must read as if a human wrote it.
- **No documentation quotes / vendor citations** ("Per Apple's docs…", "According to RFC…") unless the problem genuinely depends on them — and even then, one short clause inside the Problem section.
- **No diff narration.** If the reviewer sees it in the file list, don't restate it. Bullets describe *what changed conceptually*, not *which file was edited*.
- **No defensive framing.** Don't pre-justify edge cases the reviewer hasn't asked about.

### Style

- Bullets start with a present-tense verb: "Routes cells through a three-tier queue", "Adds `--reassign-stranded` flag", "Surfaces three stranded categories on stderr".
- Code identifiers in backticks.
- Be consistent within a bullet list: either all bullets are full sentences ending in a period, or none are.

### Example (good)

```
**Problem**
`Variant.clients` was decorative — the runner ignored it and
dispatched to any available worker, starving single-allowed cells.

**Solution**
- Routes cells through a three-tier queue: pinned → restricted-by-allowed-clients → unassigned
- Workers drain narrowest-first to avoid starving single-allowed cells
- Adds `PinnedAbsent` / `PinnedExcluded` / `NoOverlap` stranded categories on stderr
- Adds `--reassign-stranded` to relax the `PinnedExcluded` path

**Testing**
- New integration test in `tests/runner_routing.rs` covers pinned + excluded overlap
- Manual repro: `cargo run -- --config fixtures/three-tier.yaml` shows all stranded cells routed

Closes #202
```

Same information density as a one-paragraph blob, but the reviewer can scan it. `## Testing` could be dropped if there were nothing non-obvious to point at.

## Output format

```
<prefix>: <headline> [(<ticket>)]

<description>
```

- **Headline**: ≤ 72 chars, imperative mood ("add", "fix", "remove" — not "added"/"adds"), no trailing period, lowercase after the prefix unless a proper noun.
- **Ticket**: append in parentheses at the end of the headline if one was provided, e.g. `feat: add retry backoff to upload client (ABC-123)`. Skip if none.
- **Description**: 1–4 short lines. Explain the *why* and any non-obvious *what*. Skip restating what the diff already shows. Skip if the headline is fully self-explanatory.
- No marketing language, no "this commit", no trailing summary.

## Examples

```
feat: add exponential backoff to S3 upload client (DATA-412)

Uploads were failing intermittently under load. Retries now use
jittered backoff up to 30s, matching the retry budget in the ingest
service.
```

```
fix: prevent double-charge when webhook retries within 5s

The idempotency key was scoped per-request instead of per-payment,
so duplicate webhooks bypassed the check.
```

```
chore: bump ruff to 0.6.9
```

## What to avoid

- Multiple prefixes (`feat/fix:`) — pick one.
- Fabricated ticket numbers.
- Restating the diff line-by-line.
- "Co-Authored-By" trailers. **Never** add Claude (or any AI) as a co-author. Only include human co-authors if the user explicitly names them.
- Any mention of Claude, the chat session, the prompt, this skill, or AI involvement — in commit messages, PR titles, PR bodies, or anywhere else this skill produces output. The work must look human-authored.
- Running `git commit` — local commits are the user's job.
- Asking "want me to run that?" before `gh pr create` / `gh pr edit`. Just run it. The invocation of `/x-pr` is the green light.
