# PEBKAC Push™

### A Claude Code Skill by Stesso

![PRs saved from disaster](https://img.shields.io/badge/PRs%20saved%20from%20disaster-∞-brightgreen)
![Side effects](https://img.shields.io/badge/side%20effects-mostly%20harmless-yellow)
![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)

---

*[Soft piano. 6am light. A technical cofounder sits at his kitchen table in yesterday's clothes, staring at a Slack notification he hasn't opened yet.]*

---

**Is someone you love pushing directly to main?**

You're not alone. Every year, millions of technical cofounders are affected by **PEBKAC** — a degenerative condition in which a well-meaning but dangerously confident non-technical partner gains unsupervised access to the codebase. Symptoms typically onset in the late evening and intensify rapidly after the discovery of AI coding assistants.

Early warning signs may include:

- Unexplained `console.log("HELLO IS THIS WORKING")` in production
- A 47MB screen recording committed to the repo
- Pull requests titled "fixed stuff" submitted at 4:12am
- `package.json` changes no one asked for
- Entire functions commented out "just in case"
- Hardcoded `localhost:3000` URLs deployed to staging
- An `.env` file with your production API keys pushed to a public repo
- Git history that reads like a crime scene
- 14 commits in 90 minutes, the last three just titled "ok," "ok now," and "please"
- The phrase "I just made a few little tweaks along the way"

If your cofounder exhibits two or more of these symptoms, it may be time to talk to your terminal about **PEBKAC Push.**

---

**PEBKAC Push** is a first-of-its-kind Claude Code skill clinically formulated to deliver enterprise-grade git hygiene to people who think `rebase` is a type of housing refinance.

In just six guided phases, PEBKAC Push works at the source — intercepting dangerous commits before they reach your codebase — while preserving your cofounder's self-esteem and desire to contribute.

**How it works:**

PEBKAC Push starts by gently confirming your cofounder is not, in fact, working directly on `main`. It then performs a full workspace audit, presenting every changed file and asking — in a supportive, nonjudgmental way — "Did you mean to do this?"

It scans for secrets, API keys, debug artifacts, and that one `.env` file they keep trying to commit. It runs the build. It runs the linter. It runs *only the relevant tests* so your cofounder doesn't have to sit through the full suite pretending to understand what's happening.

Then — and this is the breakthrough — it writes regression tests anchored to the *business intent* of the branch. Not just "does the function return the right number," but "if someone breaks this feature next week, will anything catch it." The kind of test a senior engineer writes instinctively and a non-technical contributor would never think to add.

Finally, it merges the latest target branch back in, resolves conflicts with plain-language explanations, and generates a PR so thoroughly documented that reviewers will wonder if your cofounder hired a ghostwriter.

Which, technically, they did.

---

*[Warm voice, slightly faster]*

Ask your CTO if PEBKAC Push is right for your organization.

---

**PEBKAC Push is not for everyone.** Do not use PEBKAC Push if you are already a competent software engineer — you don't need it, and honestly, it will feel patronizing. PEBKAC Push is intended for use by founders, designers, PMs, and other non-technical contributors who have been told "just open a PR" as if that is a reasonable instruction.

In clinical trials, common side effects included: a false sense of git proficiency, the urge to Slack the entire company "I just shipped a PR," increased use of the phrase "actually, I've been doing some coding lately," and staying up even later because they feel invincible now.

Rare but serious side effects included: the non-technical cofounder requesting GitHub admin access, forming opinions about CI/CD pipelines, referring to the test suite as "my tests," and in one case, mass Slacking a screenshot of a green test suite at 4am with the caption "we're so back."

In post-market surveillance, one CTO reported that his cofounder's PRs were "better documented than mine." This is a known side effect and is not considered dangerous, though it may cause temporary ego disruption.

**PEBKAC Push does not prevent your cofounder from having ideas at 11pm.** It only makes those ideas safer when they inevitably reach the codebase at 3:47am.

If your cofounder experiences a merge conflict lasting more than four hours, seek technical assistance immediately.

---

## Prescribing Information

### Installation

*PEBKAC Push is available in three convenient dosage forms:*

**Option A: Claude Code Plugin Install** *(recommended by 4 out of 5 CTOs)*

```bash
claude plugin install github:StessoDIY/pebkac-push
```

**Option B: Manual Installation** *(for the technically curious cofounder — supervised use only)*

```bash
git clone https://github.com/StessoDIY/pebkac-push.git ~/.claude/plugins/cache/pebkac-push
```

**Option C: Per-Project** *(when you want to protect a specific repo)*

Clone into your project's `.claude/plugins/` directory:

```bash
git clone https://github.com/StessoDIY/pebkac-push.git .claude/plugins/pebkac-push
```

### Activation

*PEBKAC Push activates when your cofounder says any of the following:*

- `PEBKAC push`
- `push my changes`
- `submit a PR`
- `I'm done with my changes`
- `get this ready for review`
- `submit my changes`
- `clean up and push`
- `I'm done, push this`

*No prescription required. No git knowledge required. No judgment.*

---

*[Piano resolves. Logo fade.]*

**PEBKAC Push™**

*Built by people who build things, for the people brave enough to try.*

A Claude Code skill from [Stesso](https://stesso.ai).

---

*[Tiny legal text, read at 2x speed]*

*PEBKAC Push is a Claude Code skill and not actual medication. PEBKAC stands for "Problem Exists Between Keyboard And Chair" and is not a recognized medical diagnosis, though it probably should be. Stesso, Inc. is not responsible for what your cofounder does between the hours of 11pm and 6am, nor for any Slack messages sent during that window. "Clinically formulated" means we tested it and it works, not that there was a clinical trial. No cofounders were harmed in the making of this skill, though several were gently redirected away from protected branches. PEBKAC Push requires Claude Code and a git repository. Side effects were self-reported and may be embellished. The term "enterprise-grade" is used aspirationally. Stesso believes everyone should be empowered to build things — PEBKAC Push is how we make sure the things they build don't break the other things. If you or a loved one has been affected by PEBKAC, you are not alone. All rights reserved.*
