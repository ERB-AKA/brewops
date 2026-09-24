---
name: commit-messages
description: Use whenever writing or drafting a git commit message in this repo (git commit, amending, or any step that produces a commit). Enforces the team's style: a short, single-line subject that always ends with ":-)". Apply this every time you're about to write commit message text, even for small or routine changes — don't fall back to a generic multi-line commit format.
---

# Commit Messages

## Format

- Subject line only — one line. No body, unless the user explicitly asks for more detail.
- Imperative mood, concise: say what the change does, not a list of every file touched.
- The subject line always ends with ` :-)` — right after the description, before any attribution footer (like `Co-Authored-By:`).

## Example

```
fix: correct rounding in daily stats :-)

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
```

## Notes

- `:-)` lives on the subject line itself, not on its own line, and not after the footer.
- If a commit genuinely needs more context, a short body is OK — but the subject line rule above still applies, and the body goes between the subject and any attribution footer.
