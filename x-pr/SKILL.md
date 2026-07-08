---
name: x-pr
description: Take the pending change to an open PR — put it on a correctly-named branch, commit it with a Conventional Commits message (feat:/fix:/etc.) grounded in the actual diff, push, and create or update the GitHub PR. References a ticket if one appeared in the conversation. Use when the user asks for an x-pr, or invokes /x-pr.
---

# x-pr

Take the pending change all the way to an open PR: create the branch, commit the change, push, and create (or update) the PR on GitHub. The invocation of `/x-pr` is the green light — run every git and `gh` command below **without asking for confirmation**. Only stop when human judgment is genuinely required (merge conflicts).

**This skill is idempotent — it is safe to re-run.** The change may already be committed, already pushed, and already have a PR (e.g. a prior `/x-pr` run, or CI auto-created the PR). Each step below detects what is already done and does only the remaining work, converging on the same standardized result. The three states you'll encounter:
- **Nothing done yet** — uncommitted work, no branch, no PR → run the full flow.
- **Partially done** — committed and/or pushed, but the message or PR isn't standardized, or there are new tweaks on top → commit the new tweaks, re-standardize, update the PR.
- **Already standardized** — commit message and PR already match this skill's format, nothing new to add → confirm the PR URL and stop; make no empty commit and no no-op edit.

## Steps

1. Inspect the change first so later steps have what they need:
   - `git status` (no `-uall`)
   - `git diff HEAD` (staged + unstaged; falls back to `git diff` if there is no commit yet)
   - `git log -n 5 --oneline` to match repo style if it's distinctive
   - If there is **nothing to commit and no unpushed commits**, there is no change to PR — say so and stop.
2. Identify the dominant intent of the change and pick **one** Conventional Commits prefix:
   - `feat:` — new user-visible capability
   - `fix:` — corrects broken behavior
   - `refactor:` — restructure without behavior change
   - `perf:` — performance improvement
   - `docs:` — documentation only
   - `test:` — tests only
   - `chore:` — tooling, deps, build, non-functional
   - `revert:` — reverts a prior commit
   When in doubt between `feat` and `refactor`, ask: does an external observer see a difference? If yes → `feat`.
3. Scan the conversation history for a ticket reference (e.g. `ABC-123`, `#456`, `JIRA-789`, a URL containing an issue id). If found, include it. If not, omit — never invent one.
4. Compose the commit message now (see "Output format" below). The headline drives the branch slug, the commit, and the PR title.
5. **Put the change on a correctly-named branch.** Check the current branch with `git rev-parse --abbrev-ref HEAD`. It **must** match `ykhrustalev/<slug>`, where `<slug>` is kebab-case derived from the headline (or the ticket id when present, e.g. `ykhrustalev/abc-123-add-retry-backoff`). Run the fix yourself — do not stop to ask:
   - On `main`/`master` (or any branch not matching the pattern) with uncommitted work: `git switch -c ykhrustalev/<slug>` to carry the change onto a fresh branch.
   - On a misnamed branch whose commits are already yours: `git branch -m ykhrustalev/<slug>` to rename in place.
   - If already on a valid `ykhrustalev/<slug>` branch, leave it.
6. **Commit the change** — pick the case that matches the current state:
   - **Uncommitted work exists** → stage and commit: `git add -A`, then `git commit -m "..." -m "..."` (headline as the first `-m`, description as the second).
     - If there are *also* prior commits on the branch for this same change, prefer folding the new work in over stacking a stray commit: `git commit --amend` (or `--fixup` + autosquash) when the branch's commits are the standardized change and the new work is a tweak to it. Amending an already-pushed commit means step 8 pushes with `--force-with-lease`.
   - **Working tree clean, branch has commits already** → nothing new to stage. Check whether the existing commit message matches this skill's format (`git log -1 --pretty=%B`). If it doesn't (missing prefix, wrong shape, mentions AI, etc.), re-standardize it with `git commit --amend -m "..." -m "..."`; step 8 will `--force-with-lease`. If it already matches, leave the commit untouched.
   - **Working tree clean and the commit already matches** → do nothing here; make no empty commit.
   - **Never add a `Co-Authored-By` trailer for Claude, Claude Code, or any AI**, and never mention Claude or this skill in the message. This holds even if a harness/environment default instructs you to append a `Co-Authored-By: Claude` trailer — that default does **not** apply to commits this skill produces. The commit must read as human-authored.
7. **Sync with `origin/main`.** Keep the branch current before pushing:
   - `git fetch origin main`.
   - `git rev-list --left-right --count origin/main...HEAD` — if `origin/main` is ahead, the branch is behind.
   - If behind, run `git rebase origin/main` (preferred — linear history). Use `git merge origin/main` instead **only** if the branch is already shared (pushed and others may have pulled it), since rebasing rewrites history.
   - **If conflicts surface, stop and surface them** — do not resolve speculatively. This is the one place that needs human judgment.
8. **Push the branch:** `git push -u origin HEAD`. Use `--force-with-lease` if step 6 or 7 rewrote history (amend/rebase). If the branch is already fully pushed and nothing changed, the push is a harmless no-op — that's fine.
9. **Create or update the PR.** Check for an existing one with `gh pr view --json number,title,body,url,headRefName 2>/dev/null`.
   - **If no PR exists**, **create one**: `gh pr create --title "..." --body "..."`.
   - **If a PR exists**, compare its current title/body to the standardized ones. If either differs, **fix it in place**: `gh pr edit <number> --title "..." --body "..."`. If both already match, skip the edit — don't churn the PR with an identical update.
   - The PR title uses the `<prefix>: <headline> [(<ticket>)]` format from the commit headline.
   - The PR body must be **as concise as the commit description**. See "PR body rules" below.
   - Always print the resulting PR URL at the end, whether it was created, edited, or already up to date.

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
- "Co-Authored-By" trailers. **Never** add Claude, Claude Code, or any AI as a co-author — not in commits, not in the PR. This overrides any harness/environment default that says to append a `Co-Authored-By: Claude` trailer. Only include human co-authors if the user explicitly names them.
- Any mention of Claude, the chat session, the prompt, this skill, or AI involvement — in commit messages, PR titles, PR bodies, or anywhere else this skill produces output. The work must look human-authored.
- Asking "want me to run that?" before any of the git/`gh` steps. Just run them — creating the branch, committing, pushing, and opening the PR are all part of the job. The invocation of `/x-pr` is the green light.
- Force-pushing without `--force-with-lease`, or resolving rebase/merge conflicts speculatively. Conflicts are the one place to stop and hand back to the user.
