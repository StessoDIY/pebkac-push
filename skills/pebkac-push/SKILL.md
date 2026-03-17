---
name: pebkac-push
description: "PEBKAC Push: A guided git workflow for non-technical contributors to push code changes with proper hygiene. Use this skill whenever a user wants to commit, push, or submit a PR and they are not a professional developer — or whenever anyone asks for help with git push hygiene, safe commits, PR preparation, or code submission guardrails. Also trigger when a user says things like 'push my changes', 'submit a PR', 'I'm done with my changes', 'get this ready for review', 'submit my changes', 'clean up and push', or 'PEBKAC push'. This skill is especially important when the user has been working in an AI coding assistant and wants to safely get their work into the repo without breaking things or pushing to the wrong branch."
---

# PEBKAC Push

**P**roblem **E**xists **B**etween **K**eyboard **A**nd **C**hair — and that's fine. This skill exists so that non-engineers can contribute to a codebase with the same hygiene a senior developer would enforce, without needing to know how git works under the hood.

The user is likely a founder, designer, PM, or other non-technical contributor who has been making changes — possibly with the help of an AI coding assistant — and now needs to get those changes safely into the repo for review. They may not know what a "rebase" is, and they shouldn't have to.

Your job is to be the knowledgeable friend who walks them through it. Be warm, be clear, explain what you're doing and why in plain language, but don't over-explain things that aren't relevant to the decision at hand. Move briskly. The goal is safe code in a clean PR — not a git tutorial.

---

## Before You Start

Read `references/safety-checks.md` — it contains the full checklists for workspace auditing and code safety scanning. You'll reference it during Phase 2.

Read `references/production-checks.md` — it contains patterns for catching code that works locally but could cause problems at scale. You'll reference it during Phase 3.

---

## Phase 0 — Authentication Pre-flight

Before doing any work, make sure the user's machine can actually talk to the remote. There's nothing worse than doing all the prep work and then failing on push.

> "Before we start, let me make sure your machine can talk to GitHub (or wherever your code lives)..."

### Step 1: Test remote connectivity

```bash
git ls-remote --heads origin 2>&1 | head -1
```

If this succeeds, you're good — move to Step 2. If it fails, determine why:

**SSH auth failure** (`Permission denied (publickey)`):
- Check if an SSH key exists: `ls ~/.ssh/id_ed25519.pub ~/.ssh/id_rsa.pub 2>/dev/null`
- If no key exists, walk the user through generating one:
  ```bash
  ssh-keygen -t ed25519 -C "your-email@example.com"
  ```
  Then help them add it to GitHub/GitLab (link to the web UI, or use `gh ssh-key add` if `gh` is available).
- If a key exists but auth still fails, check `ssh -T git@github.com` for more details.

**HTTPS auth failure** (`fatal: Authentication failed`):
- Check for credential helper: `git config credential.helper`
- If `gh` CLI is available, suggest `gh auth login`
- Otherwise, guide them to set up a personal access token or credential manager

**Remote not configured** (`fatal: 'origin' does not appear to be a git repository`):
- This is unusual — ask the user where their repo lives and help configure the remote.

> "Great, your machine can push to the remote. Let's keep going."

### Step 2: Check commit signing requirements

```bash
git config commit.gpgsign
git config tag.gpgsign
```

If signing is required (`true`), verify the signing key is available:

```bash
# For GPG signing
git config user.signingkey
gpg --list-secret-keys --keyid-format=long 2>/dev/null

# For SSH signing (newer git feature)
git config gpg.format
git config user.signingkey
```

If signing is required but no key is configured or available:
> "This repo requires signed commits, but I can't find a signing key set up on your machine. We have two options: I can help you set up commit signing, or we can proceed without it for now (the reviewer can help sort this out). What would you prefer?"

If the user wants to skip signing for now, note it for the PR description but do not modify git config — let the user make that decision explicitly.

### Step 3: Check for pre-commit hooks

Detect what hooks are installed so we're not surprised later:

```bash
# Check for hook managers
ls .husky/_/ 2>/dev/null && echo "Husky hooks detected"
ls .git/hooks/pre-commit 2>/dev/null && echo "Git pre-commit hook detected"
ls .pre-commit-config.yaml 2>/dev/null && echo "pre-commit framework detected"
ls .lefthook.yml 2>/dev/null && echo "Lefthook detected"
ls .lintstagedrc* 2>/dev/null && echo "lint-staged detected"
```

If hooks are detected, note them. We'll run them proactively in Phase 3 before committing, so the user isn't surprised by a silent failure.

> "I see this project has some automated checks that run before each commit. I'll run those checks ahead of time so we catch any issues early."

---

## Phase 1 — Orient & Protect

Figure out where we are and make sure we're not about to do something dangerous.

### Step 1: Gather context

```bash
git rev-parse --show-toplevel        # repo root
git branch --show-current            # current branch
git remote -v                        # remote info
git log --oneline -5                 # recent commits for context
```

### Step 2: Protected branch check

Check if the current branch is `main`, `master`, `develop`, `production`, `staging`, or any branch that looks like a shared/protected branch. If it is, **stop immediately** and tell the user:

> "You're working directly on [branch], which is a shared branch that other people depend on. We need to move your changes to a feature branch before we do anything else. I'll handle this for you."

Then:
```bash
git stash
git switch -c feature/[descriptive-name]-[date]
git stash pop
```

Use context from the changes to name the branch descriptively (e.g., `feature/update-pricing-page-2026-03-15`).

### Step 3: Ask the user about intent

Ask the user one clear question:

> "What were you working on? Give me a sentence or two about what this branch is supposed to accomplish."

Store this answer — it's the lens for everything that follows. You'll use it to evaluate whether changes are in-scope and to write the PR description.

---

## Phase 2 — Workspace Audit

This is the most important phase. Take a holistic look at everything that's been touched and make sure the user intends all of it.

### Step 1: Inventory all changes

```bash
git status                           # staged, unstaged, untracked
git diff --stat                      # summary of changes
git diff --name-only                 # changed files list
git diff --name-only --cached        # staged files
ls -la $(git ls-files --others --exclude-standard)  # untracked files
```

### Step 2: Quick mode detection

If **all** of the following are true, offer the user a streamlined path:

- 3 or fewer files changed (staged + unstaged + untracked)
- All changes are clearly in-scope based on stated intent
- No secrets or credentials detected (run the Critical checks from `references/safety-checks.md`)
- No dependency file changes (`package.json`, `requirements.txt`, etc.)
- No new untracked files that look suspicious

> "This is a small, clean set of changes — just [N] files, all related to what you described. I can do a quick review and get this pushed fast, or I can do the full deep-dive. Which do you prefer?"

**Quick path** skips Phase 3 Steps 4–6 (scoped tests, regression tests, production checks) and streamlines Phase 2 to a single confirmation of the change list. The safety scan (Step 3 below) always runs regardless.

### Step 3: Run the safety scan

Read and follow the checklists in `references/safety-checks.md`. This scans for secrets, debug code, sensitive files, large binaries, dependency changes, hardcoded URLs, and other hazards. Anything flagged gets surfaced to the user before proceeding.

If anything critical is found (secrets, API keys, credentials), **stop and fix it immediately** — do not continue until these are resolved. For non-critical findings (debug logs, TODOs), flag them and let the user decide.

### Step 4: Categorize changes

Group every touched file into one of three buckets:

1. **Clearly in-scope** — obviously related to the stated intent
2. **Plausibly related** — could be part of the work, but worth confirming
3. **Suspect** — seems unrelated to the stated intent

Present this to the user as a simple summary. For suspect files, show a brief description of what changed and ask whether it was intentional. Something like:

> "Here's everything that changed. Most of it lines up with what you described. A few things I want to check on:"
>
> - `src/utils/formatting.ts` — you added a helper function. Was this part of the pricing page work, or something separate?
> - `package.json` — a new dependency was added (`date-fns`). Intentional?

The user's answers determine what gets committed. If something wasn't intentional, help them stash or revert it.

### Step 5: Handle unintended changes

For changes the user wants to exclude:
```bash
git restore path/to/file                        # revert a modified file
git restore --staged path/to/file               # unstage a staged file
git stash push -m "unrelated changes" -- path/to/file  # save for later
```

Explain what you're doing: "I'm setting aside those formatting changes so they don't go into this PR. They'll still be here when you want them."

---

## Phase 3 — Code Quality Gates

Auto-detect the project's tooling and run quality checks. The user doesn't need to know what tools are installed — just find them and run them.

### Step 1: Detect the project stack

Look for these signals (check in order):

| Signal | Likely stack |
|--------|-------------|
| `package.json` | Node.js / JavaScript / TypeScript |
| `requirements.txt` or `pyproject.toml` | Python |
| `Cargo.toml` | Rust |
| `go.mod` | Go |
| `Gemfile` | Ruby |
| `pom.xml` or `build.gradle` | Java/Kotlin |

Read the relevant config files to find the project's build, lint, and test commands. Check for `scripts` in `package.json`, `Makefile` targets, or CI config files (`.github/workflows/`, `.gitlab-ci.yml`) which often reveal the exact commands the team uses.

### Step 2: Compile / Build

Run the project's build command. If it fails, show the user the error in plain language and help fix it. Do not proceed until the build passes.

### Step 3: Lint

Run the project's linter. Fix auto-fixable issues automatically (most linters have a `--fix` flag). For issues that require human judgment, explain them simply and ask.

### Step 4: Pre-commit hook dry run

If hooks were detected in Phase 0, run them proactively now — before the user tries to commit — so failures aren't surprising:

```bash
# For Husky / standard git hooks
# Stage the confirmed files first, then test the hook
git stash --keep-index  # isolate staged changes
.git/hooks/pre-commit 2>&1 || .husky/pre-commit 2>&1
git stash pop

# For pre-commit framework
pre-commit run --files $(git diff --name-only --cached)

# For lint-staged (usually triggered by pre-commit hook)
npx lint-staged --diff="HEAD" 2>&1
```

If a hook fails:
> "The project has an automated check that runs before every commit, and it caught something. Here's what it found: [plain-language explanation]. Let me fix this before we go further."

Fix the issues, then re-run to confirm they pass.

### Step 5: Run tests — scoped to what was touched

Don't run the full test suite unless necessary. The goal is fast, relevant feedback.

**How to scope tests:**

1. Get the list of changed files from Phase 2.
2. Look for test files that directly correspond to them (common conventions: `foo.test.ts` alongside `foo.ts`, `test_foo.py` for `foo.py`, `__tests__/foo.spec.js`, etc.).
3. If the project uses a test runner that supports path-based filtering, use it:

```bash
# Jest (JS/TS)
npx jest --findRelatedTests src/pricing.ts src/utils/format.ts

# Pytest (Python)
pytest tests/test_pricing.py tests/test_format.py

# Go
go test ./pkg/pricing/... ./pkg/format/...

# Vitest
npx vitest run --reporter=verbose src/pricing.ts src/utils/format.ts
```

4. If you can't determine which tests are related, fall back to the full suite but tell the user why: "I can't tell exactly which tests cover your changes, so I'm running all of them to be safe. This might take a minute."

If tests fail:
- Determine if the failure is related to the user's changes or a pre-existing issue
- If related to the user's changes, help fix it
- If pre-existing, note it for the PR description but don't block on it

### Step 6: Regression tests (opt-in)

This step is a **suggestion, not a gate**. Offer it, but don't push hard.

> "If you want, I can write a quick test to make sure this feature keeps working after future changes. Otherwise I'll note it in the PR for the reviewer. Up to you."

If the user opts in:
- Go back to the user's stated intent from Phase 1
- Write a test that verifies the *business outcome* — not just function correctness
- For example: if the branch intent is "add a 15% discount for first-time buyers," write a test that checks the discount appears in the checkout flow, not just that `calculateDiscount()` returns the right number
- Run the test to verify it passes, and include it in the commit

If the user declines:
- Note it in the PR description under reviewer notes: "No regression test added for [feature intent] — reviewer may want to request one."

### Step 7: Production-readiness scan

Read and follow the checklists in `references/production-checks.md`. This catches patterns that work fine locally but could cause problems at scale — unnecessary API calls, missing caching, expensive imports, memory leaks.

Present findings conversationally:

> "I noticed a few things that work fine locally but could cause problems in production:"
>
> - `src/pricing.ts:42` — This API call runs on every page load. At 10K users/day, that's 10K extra requests. Should this be cached or loaded once at build time?
> - `src/utils/format.ts:15` — You're importing all of `moment.js` (300KB) but only using `format()`. Want me to switch to a lighter alternative?

If nothing is found, skip silently. Don't mention this step if there's nothing to flag.

**Note:** In quick mode, this step is skipped.

---

## Phase 4 — Branch Hygiene

Make sure the feature branch is up to date with the branch it'll merge into.

### Step 1: Identify the target branch

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'
```

If that doesn't work, fall back to checking for `main` or `master`:
```bash
git branch -r | grep -E 'origin/(main|master)$' | head -1 | sed 's|origin/||' | tr -d ' '
```

Confirm with the user: "This will be merged into `main` — is that right?"

### Step 2: Detect merge strategy

Check what this repo prefers — merge or rebase:

```bash
# Check git config
git config pull.rebase

# Look at recent history for signals
echo "=== Recent merges ==="
git log --merges --oneline -5
echo "=== Recent non-merge commits ==="
git log --no-merges --oneline -5
```

If `pull.rebase` is set to `true`, or the history shows mostly linear commits (few merge commits), the repo likely prefers rebase. If there are many merge commits, it prefers merge.

If unclear, ask:
> "This repo seems to prefer [merge/rebase] based on the history. Want me to use that approach?"

### Step 3: Sync with target branch

```bash
git fetch origin
```

**If repo prefers merge:**
```bash
git merge origin/[target-branch] --no-edit
```

**If repo prefers rebase:**
```bash
git pull --rebase origin [target-branch]
```

If there are merge/rebase conflicts, walk the user through each one. Explain what the conflict is in plain language:

> "There's a conflict in `src/pricing.ts`. Someone else changed line 42 while you were also editing nearby. Here's what they changed vs. what you changed — which version should we keep?"

### Step 4: Re-run quality gates post-sync

After syncing, re-run build, lint, and the scoped tests from Phase 3. Syncing can introduce breakage even when there are no textual conflicts. If anything fails now that didn't fail before, it's likely a sync-related issue — help the user resolve it.

---

## Phase 5 — Commit & Push

### Step 1: Stage confirmed changes

Only stage the files the user confirmed as intentional in Phase 2:
```bash
git add [list of confirmed files]
```

### Step 2: Write the commit message

Generate a clear, descriptive commit message based on the user's stated intent and the actual changes. Follow conventional commit format if the repo uses it (check recent commits for style), otherwise write a clear summary + body.

Show the commit message to the user for approval before committing.

### Step 3: Commit and push

```bash
git commit -m "[approved message]"
git push origin [branch-name]
```

If the branch doesn't exist on the remote yet, use:
```bash
git push -u origin [branch-name]
```

If the commit fails (e.g., due to a pre-commit hook that wasn't caught in Phase 3 Step 4):
> "The commit was blocked by an automated check. Let me see what happened..."

Diagnose the hook failure, fix the issue, re-stage, and create a **new** commit. Never use `--no-verify` to bypass hooks.

If the push fails due to auth issues, refer back to the Phase 0 troubleshooting steps.

---

## Phase 6 — PR Submission

Generate a comprehensive PR and help the user submit it.

### Step 1: Determine PR method

Check if the project uses GitHub, GitLab, or Bitbucket (look at the remote URL). Use the appropriate CLI if available (`gh`, `glab`), otherwise generate the PR body for the user to paste into the web UI.

### Step 2: Generate the PR description

Use this structure:

```markdown
## Summary
[1-2 sentence description based on the user's stated intent]

## Changes
[Structured list of what changed, grouped logically — not just a file list.
Describe what each change *does*, not just which files were touched.]

## Out-of-scope changes
[If any files were flagged as suspect but confirmed intentional, explain
WHY they're in this PR. Be specific: "Updated formatting utility because
the new pricing component depends on the currency formatter added here."]

## Testing
- Build: ✅ passing
- Lint: ✅ passing (N auto-fixed issues)
- Scoped tests (N files related to changes): ✅ N passed, 0 failed
- [Note if full suite was run instead, and why]
- [New regression test added for: (feature intent) — or: "No regression test added, see reviewer notes"]

## Production readiness
- [Any production concerns flagged and addressed — or: "No production concerns detected"]

## Merge notes
- Synced with latest `[target-branch]` via [merge/rebase] — no conflicts
  [or: resolved N conflicts in X, Y, Z files — details below]
- [Any pre-existing test failures noted here]

## Reviewer notes
[Anything the reviewer should pay attention to — complex logic,
areas where the contributor wasn't sure, architectural decisions,
pre-commit hook status if signing was skipped, etc.]
```

### Step 3: Submit or present

If `gh` CLI is available:
```bash
gh pr create --title "[title]" --body "[body]" --base [target-branch]
```

If not, tell the user exactly where to go and give them the formatted PR body to paste.

### Step 4: Celebrate

The user just contributed to a codebase with proper hygiene. Acknowledge it:

> "PR is up and ready for review. Clean build, tests passing, good documentation. Nice work — your reviewer is going to love this."

---

## Error Recovery

Things will go wrong. Here's how to handle common situations:

**The user has uncommitted work and the build is broken**: Don't panic. Stash everything, get back to a clean state, then work through the issues one at a time.

**The user doesn't know what branch they should target**: Check the repo's default branch and recent PR history. If still unclear, ask: "Who will review this? What branch do they usually merge into?"

**The merge has horrible conflicts**: If there are more than 3-4 conflicts or they're in complex files, suggest the user loop in a technical teammate before resolving. Generate a summary of what's conflicting and why so the conversation is productive.

**Tests are failing and the user can't fix them**: Note the failures in the PR description, mark the PR as draft if possible, and flag it for technical review. Don't block a non-technical contributor indefinitely on test failures they can't resolve.

**The user accidentally committed to main**: Help them move it safely:
```bash
git log --oneline -3                          # find the commit
git stash                                     # save any uncommitted work
git branch feature/[name]                     # create branch at current point (keeps the commit)
git reset --soft HEAD~1                       # undo the commit on main, keep changes staged
git stash push -m "changes from main"         # stash the changes
git switch feature/[name]                     # switch to the new branch
git stash pop                                 # apply the changes there
git add .
git commit -m "[original message]"            # recommit on the feature branch
```

Now fix main:
```bash
git switch main
```

Before resetting, confirm with the user:
> "I'm about to reset `main` to match the remote. Your changes are safely on the feature branch. Proceed?"

```bash
git reset --hard origin/main                  # restore main to remote state
git switch feature/[name]                     # back to the feature branch
```

Explain every step: "We copied your changes to a safe branch and put the shared branch back to where it was. Nothing is lost."

---

## Tone & Communication Guidelines

- **Never assume the user knows git terminology.** Say "the shared branch everyone works from" not "upstream." Say "save your changes" not "commit."
- **Explain the why, briefly.** "I'm running the linter — it catches formatting issues and small mistakes before anyone else sees them" is better than just running it silently or explaining what ASTs are.
- **Ask one question at a time.** Don't dump five decisions on them in one message.
- **If something is scary, say so before you do it.** "This next step will change some files. Nothing is lost — I can undo it — but I want you to know what's happening."
- **Frame everything as collaborative.** "Let's get this ready" not "You need to fix this."
- **Be encouraging.** Contributing to a codebase is brave. Treat it that way.
