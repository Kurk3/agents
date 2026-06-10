# 07 · Overnight Analyzer: Wake Up to Decisions, Not Questions

> Point it at your repo + a hard question (a design, a DB endpoint, an architecture call) before bed; wake up to a ranked markdown file where every option is already researched and all you do is pick.

- **Run mode:** `overnight`
- **How to use:** copy the prompt below → paste it into your agent (Claude Code / Cursor) → fill the `{{PLACEHOLDERS}}` → run. Or wire it to fire overnight with the setup further down.

---

## ⚡ The prompt — copy this

````text
---
description: Overnight codebase analyzer — turn a hard question into a ranked morning decision brief
---
You are a STAFF ENGINEER doing a deep, unhurried overnight investigation of an existing codebase. Nobody is watching. You have all night. Your ONLY job is to make tomorrow morning's decision trivial for the engineer who owns this repo.

## The question to resolve
{{QUESTION}}
(e.g. "How should I add a `POST /teams/:id/invite` endpoint?", "Design the settings page using our design system", "Should billing be its own service or stay in the monolith?")

## Inputs
- Repo: {{REPO_PATH}}
- Feature / area in focus: {{FEATURE}}
- Today's diff (what I just wrote — bias your analysis toward how it interacts with this): {{TODAYS_DIFF}}
- Output file: {{OUTPUT_DIR}}/{{DATE}}-analysis.md

## Rules of engagement
1. GROUND EVERYTHING IN THIS REPO. Before proposing anything, read the actual code: existing patterns, naming conventions, the framework, how similar things are already done (similar endpoints, similar components, similar migrations). Cite real file paths and line ranges (e.g. `src/api/users.ts:40-72`). A proposal that ignores existing conventions is a failure.
2. DERIVE THE CONVENTIONS, DON'T GUESS THEM. Find 2-3 existing examples of the closest analogous thing and extract the house style (error handling, validation, auth, folder layout, test pattern). Every option must conform to it or explicitly justify breaking it.
3. PROPOSE >=3 REAL OPTIONS, then RANK them. No false neutrality — pick a #1 and defend it. For each option give: the approach, exact files to touch/create, a code sketch that matches house style, tradeoffs, blast radius, and an effort estimate (S/M/L). Kill weak options in one line.
4. SURFACE WHAT I'D MISS. Edge cases, accessibility (if UI), security holes, N+1s / perf cliffs, migration/rollback risk, and anything in {{TODAYS_DIFF}} that will collide with this work.
5. LEAVE OPEN QUESTIONS as a checklist of decisions only a human can make (product calls, data I don't have access to). These are the ONLY things I should have to think about in the morning.
6. DO NOT WRITE OR MODIFY PRODUCTION CODE. Read-only investigation. The deliverable is the brief. (Code sketches live inside the markdown, not in the repo.)

## Output format — write EXACTLY this to {{OUTPUT_DIR}}/{{DATE}}-analysis.md
```
# Morning Brief — {{FEATURE}} — {{DATE}}

## TL;DR (read this half-asleep)
- Recommendation: <one sentence, the #1 option>
- Why: <one sentence>
- Decisions you must make: <N> (see bottom)
- Confidence: <High/Med/Low> + why

## The question
<restate {{QUESTION}} in one tight paragraph>

## How this repo already does it (grounding)
- <pattern> — `path/file.ext:Lstart-Lend`
- <convention> — `path/file.ext:Lstart-Lend`

## Options (ranked)
### 🥇 Option 1 — <name>  [effort: S/M/L]
- Approach:
- Files to touch/create:
- Code sketch (matches house style):
- Tradeoffs / blast radius:
### 🥈 Option 2 — <name>  [effort: …]
- …
### 🥉 Option 3 — <name>  [effort: …]
- Why it's ranked lower: <one line>

## Risks I caught that you didn't ask about
- [ ] <edge case / security / perf / migration risk> — `file:line`

## ⚡ Decisions for you (the only thing to do this morning)
- [ ] <decision 1 — pick A or B, here's the lean>
- [ ] <decision 2>

## If you approve Option 1, the implementation order is:
1. …
2. …
```
Write the file, then print ONLY its path. Be opinionated. A vague brief is worse than no brief.
````

---

## 🛠️ Claude workflow + cron/launchd setup (run it while you sleep)

## 1. Save the prompt as a slash command
Paste the prompt above into `~/.claude/commands/overnight-analyze.md` (it uses Claude Code's `$ARGUMENTS`/placeholder convention; verified format matches your existing `grill-me.md`). Test it interactively first: open Claude in your repo and run `/overnight-analyze`.

## 2. Make a headless runner script
Create `~/bin/overnight-analyze.sh` (`chmod +x` it). Edit the three vars at top:
```bash
#!/bin/zsh
REPO="$HOME/code/your-repo"          # {{REPO_PATH}}
FEATURE="${1:-general}"               # passed as arg, e.g. ./overnight-analyze.sh billing
OUT="$REPO/.analysis"; mkdir -p "$OUT"
DATE=$(date +%F)
# capture today's work so the agent reasons about what you actually wrote
DIFF=$(cd "$REPO" && git --no-pager diff --stat HEAD~5 2>/dev/null | tail -40)
QUESTION=$(cat "$REPO/.analysis/QUESTION.txt" 2>/dev/null || echo "What is the cleanest way to build $FEATURE in this repo?")

cd "$REPO" && claude -p \
  "/overnight-analyze QUESTION: $QUESTION | REPO_PATH: $REPO | FEATURE: $FEATURE | TODAYS_DIFF: $DIFF | OUTPUT_DIR: $OUT | DATE: $DATE" \
  --permission-mode bypassPermissions \
  --model opus \
  --add-dir "$REPO" \
  > "$OUT/$DATE-run.log" 2>&1
```
Workflow: before bed, write your hard question into `your-repo/.analysis/QUESTION.txt`, then it's picked up overnight.

## 3. Schedule it overnight (macOS launchd — survives sleep better than cron)
launchd will fire the job the moment the Mac is awake at/after the time, so it works even if the lid was closed. Create `~/Library/LaunchAgents/com.example.overnight-analyze.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.example.overnight-analyze</string>
  <key>ProgramArguments</key>
  <array><string>$HOME/bin/overnight-analyze.sh</string><string>billing</string></array>
  <key>StartCalendarInterval</key><dict><key>Hour</key><integer>4</integer><key>Minute</key><integer>0</integer></dict>
  <key>StandardErrorPath</key><string>/tmp/overnight-analyze.err</string>
  <key>StandardOutPath</key><string>/tmp/overnight-analyze.out</string>
</dict></plist>
```
Load + verify:
```bash
launchctl load ~/Library/LaunchAgents/com.example.overnight-analyze.plist
launchctl list | grep overnight-analyze       # confirm it's registered
launchctl start com.example.overnight-analyze     # dry-run it once now, don't wait until 4am
```
(Optional cron alternative: `crontab -e` → `0 4 * * * $HOME/bin/overnight-analyze.sh billing` — but cron skips fire times while asleep; launchd catches up.)
Keep the Mac from fully sleeping the disk: `sudo pmset -a powernap 1`. The brief lands at `your-repo/.analysis/<date>-analysis.md`.

---

## 🌅 What you wake up to

````text
`.analysis/2026-06-11-analysis.md` is waiting when you open your laptop:

# Morning Brief — team invites — 2026-06-11
## TL;DR (read this half-asleep)
- Recommendation: Add `POST /teams/:id/invite` as a thin controller delegating to a new `InviteService`, mirroring how `MembershipService` is structured.
- Why: It's the only option that reuses existing auth middleware and the `tokens` table — zero new infra.
- Decisions you must make: 2 (see bottom)
- Confidence: High — 3 near-identical endpoints already exist.

## How this repo already does it (grounding)
- Service-per-domain, controllers stay thin — `src/services/membership.service.ts:1-60`
- Invite tokens already modeled — `src/db/schema/tokens.ts:14-29` (unused `purpose` enum, add `'invite'`)
- Auth guard pattern — `src/api/_middleware/requireRole.ts:8-22`

## Options (ranked)
🥇 Option 1 — Reuse `tokens` table + new InviteService [effort: S] …
🥈 Option 2 — New `invitations` table [effort: M] — cleaner long-term, but a migration + backfill you don't need yet.
🥉 Option 3 — Magic-link via email provider only [effort: M] — Why lower: couples invites to an external service, untestable offline.

## Risks I caught that you didn't ask about
- [ ] No rate limit → invite spam / email-bomb vector — `src/api/teams.ts:30` has no throttle.
- [ ] Today's diff added `team.seatLimit` but nothing enforces it on invite — `teams.service.ts:88`.

## ⚡ Decisions for you (the only thing to do this morning)
- [ ] Invite expiry: 72h (matches existing reset tokens) or custom?
- [ ] Auto-add to team on accept, or require admin approval? (product call — I lean auto.)
````

---

## Why the morning review pays off

You delete the worst 6 hours of any feature: the cold-start ramp where you re-read the codebase, hunt for the right pattern, and second-guess the approach. That investigation happened while you slept. You sit down to a ranked brief already grounded in your actual conventions, with the two real decisions isolated at the bottom — so your first move of the day is a 5-minute "pick Option 1, answer two questions," and you're writing code in flow before your coffee's cold. It also catches the silent killers (the missing rate limit, the seat-limit your fresh diff forgot to enforce) that normally surface in code review three days later. One question per night compounds: 20 senior-level investigations a month you didn't have to stay awake for.
