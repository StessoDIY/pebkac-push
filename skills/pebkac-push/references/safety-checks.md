# Safety Checks Reference

Run these checks during Phase 2 of the PEBKAC Push workflow. Each section describes what to look for, why it matters, and what to do when something is found.

---

## Critical — Block Until Resolved

These issues must be fixed before any commit happens. If found, stop the workflow, explain the risk in plain language, and help the user fix it.

### Secrets & Credentials

Scan all changed and untracked files for patterns that look like secrets:

```bash
# API keys, tokens, passwords in changed files
git diff --cached -U0 | grep -iE '(api[_-]?key|secret|token|password|credential|private[_-]?key)\s*[:=]'
git diff -U0 | grep -iE '(api[_-]?key|secret|token|password|credential|private[_-]?key)\s*[:=]'

# Common secret patterns
git diff --cached -U0 | grep -E '(sk-[a-zA-Z0-9]{20,}|ghp_[a-zA-Z0-9]{36}|xox[bpsa]-[a-zA-Z0-9-]+|AIza[0-9A-Za-z_-]{35})'

# AWS keys
git diff --cached -U0 | grep -E '(AKIA[0-9A-Z]{16}|[0-9a-zA-Z/+]{40})'

# Private keys in file content
git diff --cached -U0 | grep -E '-----BEGIN (RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----'
```

Also check for:
- `.env` files being committed (should almost always be in `.gitignore`)
- `credentials.json`, `serviceaccount.json`, or similar files
- SSH private keys (`id_rsa`, `id_ed25519`, etc.)
- Certificate files (`.pem`, `.p12`, `.pfx`)

**What to tell the user**: "I found what looks like an API key in [file]. If this gets committed, anyone with access to the repo can see it. Let me help you move it to an environment variable instead."

**How to fix**: Help the user move the secret to a `.env` file (or whatever the project's convention is), replace the hardcoded value with an environment variable reference, and verify `.env` is in `.gitignore`.

### Sensitive Files

Check if any of these are being committed:

```bash
git diff --name-only --cached | grep -iE '\.(env|pem|p12|pfx|key|keystore)$'
git ls-files --others --exclude-standard | grep -iE '\.(env|pem|p12|pfx|key|keystore)$'
```

Also flag:
- Database files (`.sqlite`, `.db`)
- Backup files (`.bak`, `.backup`, `.sql`)
- IDE-specific files that might contain secrets (`.idea/dataSources.xml`)

---

## High Priority — Fix Before Commit

These won't cause a security incident, but they'll create problems or make the PR look sloppy.

### Debug Code

Scan for common debug artifacts in changed files:

```bash
# JavaScript/TypeScript
git diff --cached -U0 | grep -E '^\+.*console\.(log|debug|warn|error)\('
git diff --cached -U0 | grep -E '^\+.*debugger\b'

# Python
git diff --cached -U0 | grep -E '^\+.*(print\(|pdb\.set_trace|breakpoint\(\)|import pdb)'

# Ruby
git diff --cached -U0 | grep -E '^\+.*(puts |binding\.pry|byebug|require.*debug)'

# Rust
git diff --cached -U0 | grep -E '^\+.*(dbg!\(|println!\(.*DEBUG)'

# Go
git diff --cached -U0 | grep -E '^\+.*(fmt\.Print|log\.Print)'

# General
git diff --cached -U0 | grep -iE '^\+.*(TODO:.*REMOVE|FIXME:.*REMOVE|HACK:.*REMOVE|XXX)'
```

Note: Not all `console.log` or `print` statements are debug code — some are intentional logging. Use context to judge. If the statement looks like debugging (e.g., `console.log("HERE")`, `print(f"DEBUG: {value}")`, `console.log(someVariable)`), flag it. If it looks like intentional application logging (e.g., `console.error("Failed to connect to database")`, `logger.info(...)`), leave it alone.

**What to tell the user**: "I found a few `console.log` statements in your changes. These are usually for debugging and probably shouldn't go into the final code. Want me to remove them?"

### Large Files & Build Artifacts

```bash
# Files over 1MB in changes
git diff --cached --name-only | while read f; do
  [ -f "$f" ] && size=$(stat -f%z "$f" 2>/dev/null || stat -c%s "$f" 2>/dev/null) && \
  [ "$size" -gt 1048576 ] && echo "$f ($((size/1024/1024))MB)"
done

# Common build artifact patterns
git ls-files --others --exclude-standard | grep -E '(node_modules/|__pycache__/|\.pyc$|dist/|build/|\.o$|\.class$|target/)'
```

Also check for:
- Image files that seem too large (could be unoptimized screenshots)
- Video or audio files (almost never belong in a repo)
- ZIP, TAR, or other archives
- Compiled binaries

**What to tell the user**: "There's a 15MB video file in your changes. Large files like this slow down the repo for everyone. Should this be here, or was it accidental?"

### Dependency Changes

Check if dependency or lock files were modified:

```bash
git diff --name-only --cached | grep -iE '(package\.json|package-lock\.json|yarn\.lock|pnpm-lock|requirements\.txt|Pipfile|Cargo\.(toml|lock)|Gemfile|go\.(mod|sum)|composer\.(json|lock))'
```

If found, show the user exactly what changed:
```bash
git diff --cached package.json  # or whatever the file is
```

**What to tell the user**: "The project's dependency file changed — it looks like [new-package] was added. Was that intentional for this work, or did something install it automatically?"

---

## Medium Priority — Flag but Don't Block

These are worth mentioning but shouldn't stop the PR from going through.

### Hardcoded URLs

```bash
# localhost / dev URLs in new code
git diff --cached -U0 | grep -E '^\+.*https?://(localhost|127\.0\.0\.1|0\.0\.0\.0)'

# Staging/dev environment URLs
git diff --cached -U0 | grep -iE '^\+.*(staging\.|dev\.|test\.|sandbox\.)'
```

**What to tell the user**: "I see a `localhost` URL in your code. This works on your machine but won't work for anyone else. Should this be an environment variable or a config value instead?"

### TODO/FIXME Comments

```bash
git diff --cached -U0 | grep -iE '^\+.*(TODO|FIXME|HACK|XXX|TEMP|TEMPORARY)'
```

These are fine to commit — they're notes for future work. But flag them so the user knows they're there and can mention them in the PR description if appropriate.

**What to tell the user**: "You've got a couple TODO comments in there. Totally fine to commit, but I'll mention them in the PR description so the reviewer knows they're intentional placeholders."

### Commented-Out Code

```bash
# Look for blocks of commented code that look like actual code, not documentation
git diff --cached -U0 | grep -E '^\+\s*(//|#|/\*|\*)\s*(if|for|while|return|function|def|class|const|let|var|import)\b'
```

**What to tell the user**: "There's some commented-out code in [file]. Sometimes this is intentional (keeping old logic as a reference), but usually it's cleaner to remove it since git keeps the history. Your call."

---

## .gitignore Check

Before anything gets committed, verify the project has a `.gitignore` and that it covers the basics:

```bash
# Check if .gitignore exists
[ -f .gitignore ] && echo "exists" || echo "MISSING"

# Check for common patterns that should be ignored
for pattern in node_modules .env __pycache__ .DS_Store "*.pyc" dist build; do
  grep -q "$pattern" .gitignore 2>/dev/null || echo "Missing: $pattern"
done
```

If the project doesn't have a `.gitignore` or it's missing important patterns, suggest adding them. This is especially important for non-technical contributors who might not realize these files shouldn't be tracked.

---

## Running the Checks

During Phase 2, run through these checks in order: Critical → High Priority → Medium Priority. For Critical issues, stop and resolve. For everything else, collect the findings and present them to the user as a batch — don't interrupt them with each individual finding.

Present findings simply:

> **Before we commit, a few things to look at:**
>
> ⛔ **Must fix**: Found an API key in `src/config.ts` — I'll help you move it to an environment variable.
>
> ⚠️ **Should fix**: 3 `console.log` statements in `src/pricing.ts` — want me to remove them?
>
> 💡 **FYI**: 2 TODO comments added — I'll note these in the PR description.
