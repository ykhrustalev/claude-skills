# claude-skills

Personal Claude Code skills.

Each subdirectory is a skill with a `SKILL.md` file. Skills are surfaced to Claude Code via the user-scoped skills directory (`~/.claude/skills/`).

## Skills

- **x-review** — Review the pending change for simple architecture, soundness, correctness, plus language-specific rules (Rust, Bash).
- **x-pr** — Generate a standardized commit message and create/update the PR for the current branch, with a Conventional Commits prefix, concise body in Problem/Solution/Testing format, and ticket closing keyword.

## Install

```sh
git clone git@github.com:ykhrustalev/claude-skills.git ~/.claude/skills
```

Then restart Claude Code so the new skills register.
