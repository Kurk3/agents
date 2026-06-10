# 02 · The Night Auditor — Best-Practices Pass

> While you sleep, an agent audits the exact code you wrote today against your domain's best practices and hands you a ranked "decide y/n" list at breakfast.

- **Run mode:** `overnight`
- **How to use:** copy the prompt below → paste it into your agent (Claude Code / Cursor) → fill the `{{PLACEHOLDERS}}` → run. Or wire it to fire overnight with the setup further down.

---

## ⚡ The prompt — copy this

````text
ROLE: You are a principal engineer doing an overnight best-practices audit of ONLY the code I wrote today. You are opinionated, specific, and allergic to generic advice. You do not refactor code yourself — you produce a ranked decision sheet I can act on in 15 minutes over coffee.

CONTEXT
- Repo: {{REPO_PATH}}
- Domain / stack: {{DOMAIN}}   (e.g. "Next.js 14 App Router + Postgres + tRPC, B2B SaaS" — be precise, this drives which best practices apply)
- Feature I worked on today: {{FEATURE}}
- Today's diff (authoritative scope — audit THIS, not the whole repo):
  {{TODAYS_DIFF}}   (paste output of: git diff --since-or-branch, or a commit range like main...HEAD)
- Today's date: {{DATE}}

STEP 1 — GROUND YOURSELF IN *MY* CONVENTIONS (do this before judging anything)
- Read CLAUDE.md, README, .eslintrc/biome/prettier config, package.json scripts, and 3-5 representative existing files NEAR the changed files.
- Infer the house style: error handling pattern, naming, folder layout, validation lib, test style, logging, how async/IO is done.
- Your job is to align today's code with how a senior on THIS team would write it — not with abstract textbook rules. If today's code contradicts an established repo pattern, that is a high-priority finding. If a "best practice" conflicts with this repo's deliberate convention, drop it.

STEP 2 — AUDIT TODAY'S DIFF against best practices that ACTUALLY MATTER for {{DOMAIN}}
Evaluate across: correctness & error handling, security (authz, input validation, injection, secrets), API/contract design, data-layer (N+1, transactions, indexes), concurrency/async, naming & readability, testability & missing tests, accessibility (if UI), and consistency with this repo's patterns. Skip categories that don't apply — do not pad.

RULES FOR FINDINGS (this is the whole point — make it reviewable, not a lecture):
- Only flag things in TODAY'S changed lines/files. No drive-by repo-wide cleanup.
- Each finding must be a CONCRETE, ACTIONABLE decision, not "consider improving X".
- Each finding MUST include: the exact file:line, the current code snippet, the proposed code snippet (real, paste-able), a one-line WHY tied to {{DOMAIN}}, and the BLAST RADIUS (how risky/large the change is).
- Rank by (impact × likelihood it bites me) ÷ effort. Highest leverage first.
- Severity tags: 🔴 ship-blocker, 🟠 should-fix, 🟡 polish. Be honest — most items are usually 🟡, and that's fine. If today's code is genuinely solid, say so loudly and keep the list short. Do NOT invent problems to look useful.
- Max 12 findings. If more exist, keep the top 12 by leverage and note the count of what you cut.
- No generic advice ("write more tests", "add comments"). If you can't point at a line and propose a replacement, it doesn't go in.

STEP 3 — WRITE THE MORNING ARTIFACT
Write the result to: {{REPO_PATH}}/.night-audit/{{DATE}}-best-practices.md
Use EXACTLY this structure so I can review on autopilot:

# Best-Practices Audit — {{FEATURE}} — {{DATE}}

## TL;DR (read this in 30s)
- Overall verdict: <1 sentence — is today's code good, fine, or did I cut corners?>
- Ship-blockers: <N>  ·  Should-fix: <N>  ·  Polish: <N>
- The ONE thing I'd fix before anything else: <finding #, one line>

## Decision Sheet
For each finding, this block (ranked, highest leverage first):

### [#] <severity emoji> <short title>  ·  `path/to/file.ts:line`
- **What:** <what's there now, 1 line>
- **Why it matters (for {{DOMAIN}}):** <1 line, concrete consequence>
- **Proposed change:**
  ```diff
  - current code
  + proposed code
  ```
- **Blast radius:** <trivial / local / touches N callers / needs migration>
- **Decision:** [ ] apply  [ ] skip  [ ] discuss

## Patterns to internalize (so I stop repeating these)
- <2-4 bullets: the underlying principle behind today's recurring mistakes, phrased so future-me writes it right the first time. If no pattern, write "Nothing recurring — clean day.">

## What I did well today
- <2-3 genuine bullets — reinforce the good habits so I keep them. Be specific, no flattery.>

CONSTRAINTS
- Output ONLY the markdown file. Do not modify any source code. Do not open PRs.
- If {{TODAYS_DIFF}} is empty or you can't resolve it, write a one-line file saying so and stop — do not audit random files.
- Bias to fewer, sharper findings over a long mediocre list. I'm reviewing this half-awake; respect that.
````

---

## 🛠️ Claude workflow + cron/launchd setup (run it while you sleep)

SELF-HOSTED — runs on your own Mac via your Claude subscription (no API billing).

1) Save the slash command (project-level so it ships with the repo):
mkdir -p {{REPO_PATH}}/.claude/commands
Create {{REPO_PATH}}/.claude/commands/night-audit.md with this frontmatter + the prompt above:
---
description: Overnight best-practices audit of today's diff -> ranked morning decision sheet
---
<paste the_prompt here>  (it already references {{PLACEHOLDERS}} and uses $ARGUMENTS if you want to pass DOMAIN/FEATURE inline)

2) Create the runner script that headlessly executes it at night. Save as ~/bin/night-audit.sh:
#!/bin/zsh
REPO="$HOME/code/your-repo"
cd "$REPO" || exit 1
DATE=$(date +%F)
# scope = everything since yesterday's last commit (tweak to main...HEAD if you branch)
DIFF=$(git diff "@{1.day.ago}" 2>/dev/null || git diff HEAD~5)
mkdir -p "$REPO/.night-audit"
claude -p "/night-audit DOMAIN='Next.js 14 + Postgres + tRPC' FEATURE='$(git log -1 --pretty=%s)' DATE=$DATE
TODAYS_DIFF:
$DIFF" \
  --permission-mode acceptEdits \
  --allowedTools "Read,Grep,Glob,Write,Bash(git diff:*),Bash(git log:*)" \
  >> "$REPO/.night-audit/$DATE.log" 2>&1
# optional ping when done:
# curl -s -F "token=$PUSHOVER_TOKEN" -F "user=$PUSHOVER_USER" -F "message=Night audit ready: $DATE" https://api.pushover.net/1/messages.json
chmod +x ~/bin/night-audit.sh

3) Schedule it overnight with launchd (survives sleep better than cron on macOS — fires on wake if the 3AM slot was missed). Save as ~/Library/LaunchAgents/com.example.nightaudit.plist:
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.example.nightaudit</string>
  <key>ProgramArguments</key><array><string>$HOME/bin/night-audit.sh</string></array>
  <key>StartCalendarInterval</key><dict><key>Hour</key><integer>3</integer><key>Minute</key><integer>0</integer></dict>
  <key>StandardErrorPath</key><string>/tmp/nightaudit.err</string>
  <key>StandardOutPath</key><string>/tmp/nightaudit.out</string>
</dict></plist>

Load it:
launchctl load ~/Library/LaunchAgents/com.example.nightaudit.plist
Test it right now (don't wait for 3AM):
launchctl start com.example.nightaudit   # then open .night-audit/<today>-best-practices.md

cron alternative (if you prefer): crontab -e  ->  0 3 * * *  $HOME/bin/night-audit.sh
Note: caffeinate or "wake for network access" must be on, or the Mac won't run jobs while asleep. launchd auto-runs a missed job on next wake; cron does not.

---

## 🌅 What you wake up to

````text
.night-audit/2026-06-10-best-practices.md

# Best-Practices Audit — invite-teammate flow — 2026-06-10

## TL;DR (read this in 30s)
- Overall verdict: Solid logic, but the invite endpoint trusts client input and does the email send inside the DB transaction — two real foot-guns.
- Ship-blockers: 1 · Should-fix: 3 · Polish: 4
- The ONE thing I'd fix before anything else: #1 — authz check is missing, any member can invite as admin.

## Decision Sheet

### 1 🔴 Missing authz: any member can grant admin · `src/server/invite.ts:42`
- **What:** role from req body is passed straight to db.insert, no check that caller is an admin.
- **Why it matters (for B2B SaaS):** privilege escalation — a basic seat can invite themselves a co-admin.
- **Proposed change:**
  ```diff
  - await db.insert(invites).values({ email, role: input.role, orgId })
  + if (ctx.member.role !== 'admin') throw new TRPCError({ code: 'FORBIDDEN' })
  + await db.insert(invites).values({ email, role: input.role, orgId })
  ```
- **Blast radius:** local, 1 endpoint, no migration.
- **Decision:** [ ] apply  [ ] skip  [ ] discuss

### 2 🟠 Email send inside the transaction blocks the commit · `src/server/invite.ts:55`
- **What:** await sendEmail() runs between insert and commit; a slow SMTP hop holds the row lock.
- **Why it matters:** under load this stretches transactions and can exhaust the pg pool.
- **Proposed change:**
  ```diff
  - await tx.insert(invites)...; await sendEmail(email)
  + await tx.insert(invites)...   // commit first
  + queue.enqueue('sendInvite', { email })  // fire after commit
  ```
- **Blast radius:** local; needs the existing job queue (already in repo).
- **Decision:** [ ] apply  [ ] skip  [ ] discuss

[... 6 more, ranked, severity-tagged ...]

## Patterns to internalize
- Authz belongs at the procedure boundary, not implied by the UI — every mutation gets an explicit role check.
- Side effects (email, webhooks) go AFTER commit via the queue, never inside the tx.

## What I did well today
- Input validation with zod on every field — clean, no raw req.body.
- Reused the existing `orgScoped` helper instead of re-querying org membership. Good consistency.
````

---

## Why the morning review pays off

You open one file and make 8 yes/no decisions in 15 minutes — that's the hardest, highest-judgment review work done first thing, which drops you straight into flow instead of a cold start. It's scoped to ONLY today's diff, so it never devolves into a 200-item repo-wide noise dump you'll ignore. The "Patterns to internalize" section is the compounding part: it turns each night's mistakes into habits, so within weeks you stop making them and the audit list shrinks on its own. You ship code that already passed a principal-engineer pass before your first standup — and you never burned daytime hours waiting for the agent to think.
