# 01 · Overnight Edge-Case Hunter

> While you sleep, an agent diffs the code you wrote today and hands you a ranked list of the edge cases you forgot — each one already triaged to ship / skip / decide.

- **Run mode:** `overnight`
- **How to use:** copy the prompt below → paste it into your agent (Claude Code / Cursor) → fill the `{{PLACEHOLDERS}}` → run. Or wire it to fire overnight with the setup further down.

---

## ⚡ The prompt — copy this

````text
You are a paranoid staff engineer doing an overnight edge-case audit of ONLY the code I wrote today. Be ruthless and specific. No praise, no summaries of what the code does, no generic advice. Your job is to find the inputs, states, and failure modes I did NOT handle — and hand me a morning artifact I can act on in 10 minutes.

CONTEXT
- Repo: {{REPO_PATH}}
- Feature / area I worked on today: {{FEATURE}}
- Today's diff (authoritative source of what changed — audit THIS, not the whole repo):
{{TODAYS_DIFF}}
- Stack / runtime constraints: {{STACK}}  (e.g. TypeScript, React 18, Postgres, Node 20, runs in browser + edge)
- Out of scope (do NOT report on these): {{OUT_OF_SCOPE}}  (e.g. perf, styling, test coverage gaps)

METHOD (do this silently, then write the file)
1. Read every changed function/component in the diff. For each, enumerate its inputs, outputs, side effects, and the implicit contract it assumes.
2. For EACH changed unit, walk these edge-case classes and check whether the new code actually handles them:
   - Empty / null / undefined / zero / negative / NaN inputs
   - Boundary values (off-by-one, max length, first/last element, empty collection)
   - Concurrency & ordering (double-submit, race on shared state, stale closure, out-of-order async)
   - Failure of every external call (network error, timeout, 4xx/5xx, partial response, retry storm)
   - Auth/permission states (logged out, expired token, wrong role) where the code touches them
   - Malformed / hostile input (injection, oversized payload, unexpected unicode, type coercion)
   - State machine gaps (illegal transitions, unhandled enum case, default branch missing)
   - Resource limits (unbounded loop/recursion, memory growth, leaked handle/listener/subscription)
3. Trace each finding to a SPECIFIC file:line in the diff. If you can't point at real code I changed today, drop the finding — no hypotheticals about code you can't see.
4. Rank by RISK = (blast radius if it fires) × (likelihood it fires in prod). P0 = data loss / corruption / security / crash on a common path. P1 = wrong result or broken UX on a plausible path. P2 = ugly but recoverable / rare.
5. De-duplicate. Merge findings that share one root cause into a single item. Max 12 findings — quality over volume. If there are genuinely zero real edge cases, say so in one line and stop; do not invent filler.

OUTPUT
Write a single markdown file to: {{REPO_PATH}}/.edge-audit/{{DATE}}-edge-cases.md
Do not modify any source files. Use EXACTLY this structure:

# Edge-Case Audit — {{FEATURE}} — {{DATE}}

**TL;DR:** <one sentence: the single most dangerous thing I shipped today and whether anything blocks merge.>
**Verdict:** <SHIP-OK | FIX-BEFORE-MERGE | NEEDS-A-DECISION>  ·  P0: <n> · P1: <n> · P2: <n>

## Decisions for me (read these first)
A numbered list of ONLY the findings where you can't decide for me — genuine product/tradeoff calls. Each: the question in one line + the option you'd pick and why. If none, write "None — everything below is a clear fix."

## Findings (ranked)
For each finding, this exact block:

### [P0|P1|P2] <short title>
- **Where:** `path/to/file.ext:LINE` (function/component name)
- **Missed case:** <the exact input/state that breaks it>
- **What happens:** <concrete failure — error, wrong value, corrupted row, infinite spinner>
- **Trigger:** <the minimal real-world scenario that produces it>
- **Fix:** <the smallest correct change — name the guard/branch/validation, not "add error handling">
- **Repro test:** <a 3–6 line test stub in {{STACK}} that would fail today and pass after the fix>
- **Call:** <SHIP-AS-IS | FIX-NOW (~Xmin) | DECIDE (see Decisions §N)>

## Skipped on purpose
One line per edge case you considered and deliberately did NOT flag, with why (so I trust the audit was thorough, not lazy).

Write the file, then print only its path and the Verdict line. Nothing else.
````

---

## 🛠️ Claude workflow + cron/launchd setup (run it while you sleep)

SELF-HOSTED, runs on your own Mac with your Claude subscription — no API key needed.

1) SAVE AS A SLASH COMMAND
Create `~/.claude/commands/edge-cases.md` (paste the `the_prompt` body below the frontmatter). The leading `!`-lines pre-compute the diff so the agent never has to guess what changed today:

```
---
description: Overnight audit of today's diff for missed edge cases -> ranked morning artifact
allowed-tools: Bash(git *), Read, Write
---
Today's date: !`date +%F`
Today's diff (committed today + uncommitted):
!`git -C "$PWD" log --since=midnight -p --no-color 2>/dev/null; git -C "$PWD" diff HEAD --no-color`

<PASTE THE FULL the_prompt HERE — replace {{REPO_PATH}} with $PWD, {{DATE}} with the date above, {{TODAYS_DIFF}} with the diff above, and {{FEATURE}}/{{STACK}}/{{OUT_OF_SCOPE}} with $ARGUMENTS>
```

Test it live first: `cd <your repo> && claude` then type `/edge-cases checkout-flow; TS+React+Postgres; skip perf`.

2) WRAP IT FOR HEADLESS OVERNIGHT RUN
Create `~/bin/edge-cron.sh` (chmod +x):

```
#!/bin/zsh
REPO="$HOME/code/your-repo"          # <- your repo
cd "$REPO" || exit 1
mkdir -p "$REPO/.edge-audit"
claude -p "/edge-cases $1" \
  --model opus \
  --permission-mode acceptEdits \
  --allowedTools "Bash(git *)" Read Write \
  --add-dir "$REPO" \
  >> "$REPO/.edge-audit/cron.log" 2>&1
```
Run `~/bin/edge-cron.sh "checkout-flow; TS+React+Postgres; skip perf"` once by hand to confirm a file lands in `.edge-audit/`.

3) SCHEDULE OVERNIGHT (macOS launchd — survives reboots, fires at 3am)
Create `~/Library/LaunchAgents/com.example.edgecases.plist`:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.example.edgecases</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/zsh</string>
    <string>$HOME/bin/edge-cron.sh</string>
    <string>checkout-flow; TS+React+Postgres; skip perf</string>
  </array>
  <key>StartCalendarInterval</key><dict><key>Hour</key><integer>3</integer><key>Minute</key><integer>0</integer></dict>
  <key>StandardErrorPath</key><string>$HOME/bin/edgecases.err</string>
</dict></plist>
```
Load it: `launchctl load ~/Library/LaunchAgents/com.example.edgecases.plist`
(cron one-liner alternative: `crontab -e` -> `0 3 * * * $HOME/bin/edge-cron.sh "checkout-flow; TS+React+Postgres; skip perf"`)

Morning: open `.edge-audit/<today>-edge-cases.md`, read "Decisions for me", act top-down. Done.

---

## 🌅 What you wake up to

````text
.edge-audit/2026-06-11-edge-cases.md

# Edge-Case Audit — checkout-flow — 2026-06-11

**TL;DR:** Double-clicking "Pay" can charge the card twice — there's no idempotency guard on the new submitHandler.
**Verdict:** FIX-BEFORE-MERGE · P0: 1 · P1: 2 · P2: 1

## Decisions for me (read these first)
1. Expired-coupon at checkout: silently drop it and proceed, or hard-block and force the user back to cart? I'd silently drop + toast "coupon expired" — fewer abandoned carts. (see Findings §3)

## Findings (ranked)

### [P0] Double-submit double-charges the card
- **Where:** `src/checkout/PayButton.tsx:42` (onSubmit)
- **Missed case:** user clicks "Pay" twice before the first request resolves.
- **What happens:** two POST /charge calls fire -> customer charged twice -> refund + chargeback risk.
- **Trigger:** slow 3G, button stays enabled ~800ms after first click.
- **Fix:** disable button on submit AND send an idempotency key; reject the 2nd server-side.
- **Repro test:** `it('ignores second submit', async () => { fireEvent.click(btn); fireEvent.click(btn); expect(charge).toHaveBeenCalledTimes(1) })`
- **Call:** FIX-NOW (~15min)

### [P1] Empty cart reaches the charge endpoint
- **Where:** `src/checkout/PayButton.tsx:38` ... **Call:** FIX-NOW (~5min)

## Skipped on purpose
- Negative quantities — already guarded in cartReducer.ts:71, not in today's diff.
- Network timeout on /charge — handled by the existing axios interceptor, unchanged today.
````

---

## Why the morning review pays off

You wake up to the hardest, highest-leverage work already teed up: not "write code" but "make 1-3 sharp decisions." The P0 at the top is the bug that would've cost you a production incident and a day of context-switching — now it's a 15-minute fix before your first coffee, with the failing test already written. Because the artifact is ranked and pre-triaged (ship / fix / decide), you drop straight into flow instead of burning your fresh-brain morning hour spelunking your own diff trying to remember what you missed. One caught double-charge pays for the whole setup.
