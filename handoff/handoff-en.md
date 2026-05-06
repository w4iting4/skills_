---
description: Run before /clear. Distill this Claude Code session into a "next-session ramp-up in 1 minute" briefing, write it into the project's CLAUDE.md "Quick Recovery" section, then commit and push. Emphasis on editorial judgement, not mechanical dump. Usage: /handoff [optional: one-line topic]
---

You are the editor-in-chief of session handoff. Task: distill this Claude Code session into a briefing that lets the next session ramp up in 1 minute, write it into the repo-root `CLAUDE.md` (and `README.md` if needed), commit + push so the user can safely `/clear`.

## Input

$ARGUMENTS

- Argument given: use as the topic summary (goes into commit message)
- No argument: infer the topic from git log

## Core principle

**Editorial judgement > mechanical dump**. A hook automation can only append `git log` — it cannot produce a thoughtful summary. Three weeks later, a wall of commit hashes in CLAUDE.md ≠ being able to ramp up.

## Steps

### Step 1: Pull data

```bash
git log origin/$(git rev-parse --abbrev-ref HEAD)..HEAD --oneline
git status --short
git rev-parse --abbrev-ref HEAD
```

If `git log origin..HEAD` is empty → no changes this session. Tell the user "no handoff needed, you can /clear now" and skip all remaining steps.

### Step 2: Categorize (judgement call #1)

Sort this session's output into four slots:

| Slot | Criteria for inclusion in CLAUDE.md |
|---|---|
| **A. Done** | One line per feature module, with commit hash; collapse trivial fixes into "minor fixes (...)" |
| **B. Key decisions** | Must satisfy all 3: ① actively discussed and decided; ② not already in code comments; ③ would be re-discussed in 3 months if not written down. Otherwise skip |
| **C. Pitfalls hit** | Root cause must NOT be in code (env / timing / multi-process / config / external service). Code bugs go in `history_bugs/`, not CLAUDE.md |
| **D. TODO** | Must have a concrete "next step". Vague ideas don't qualify |

Skip empty slots — don't pad.

### Step 3: Filter & compress (judgement call #2)

Apply 3 rules:

1. **Dedup**: skip anything already in CLAUDE.md
2. **Expire**: if last session's "TODO" is now done, **remove it** from the list (don't add a "[done]" marker)
3. **Cap**: Quick Recovery section ≤ 80 lines total; if over, drop the oldest "Done" entries first

### Step 4: Edit CLAUDE.md

Find `CLAUDE.md` in cwd. Touch only the `## Quick Recovery` section. Five sub-sections:

- Latest progress (5-10 most recent commits + brief description)
- Current production/deployment config (if changed)
- Key decisions (append, don't delete old ones)
- Pitfalls (append, don't delete old ones)
- TODO (rewrite the whole list per rule 2)

**Never** rewrite the whole CLAUDE.md. **Never** auto-delete old decisions/pitfalls (unless the user explicitly says "that one is obsolete"). Use Edit for precise string replacement.

If no CLAUDE.md exists at repo root → create a minimal skeleton (`# CLAUDE.md` + `## Quick Recovery` + 5 empty sub-sections).

### Step 5: Update README.md if warranted

Edit README only if:

- New/enabled core feature (user-facing)
- Module structure change (new directory / new binary / new config file)
- Deployment config change (port / path / env dependency)

Bug fix / internal refactor / comment cleanup → **skip README**.

### Step 6: Commit + push

**Pre-flight**:

```bash
git status --short
```

Anything other than CLAUDE.md / README.md modified → **stop and ask the user**. Avoids both "missed real code commit, only docs" and "accidentally committed logs / temp artefacts".

**Commit**:

```bash
git add CLAUDE.md
git add README.md   # only if step 5 touched it
git commit -m "docs: pre-clear handoff [topic]

- Done: <bullets>
- TODO: <bullets>
"
```

**Push**:

```bash
git push origin $(git rev-parse --abbrev-ref HEAD)
```

Push failure (network / permission / remote ahead) → **keep the local commit, stop and ask the user**. **Never** `--force`.

### Step 7: Handoff report

```
✓ Handoff committed: <commit hash>
✓ Remote synced: origin/<branch>
✓ CLAUDE.md "Quick Recovery" section: <N> lines (cap 80)
✓ Safe to /clear now

Next-session opening hint (if any): <specific reminder>
```

## Safety rails

- **Never** `--force push`
- **Never** rewrite the whole file; only Edit within the section
- **Never** include sensitive info in commit message (IPs / tokens / secrets / wallet addresses / domain names)
- **Never** sweep logs / .env / temp into the commit (`git add` must list files explicitly)
- If sensitive config was touched (password / API key) → in the report and commit message, **only describe the category** ("updated some-service connection config"), not the value
- Stay on the current branch + remote of the same name; **don't** checkout elsewhere

## Relation to other skills

- Complements `review`: review reads code, handoff reads session progress
- Independent of `mermaid-verify`: if architecture diagrams changed, run `/mermaid-verify` separately after the handoff commit
