# 05 · The Morning Code Review

> While you sleep, a senior-level agent reviews the code you shipped today and leaves a ranked markdown file of fixes, risks, and "study this" gaps — so you wake up and just decide, not investigate.

- **Run mode:** `overnight`
- **How to use:** copy the prompt below → paste it into your agent (Claude Code / Cursor) → fill the `{{PLACEHOLDERS}}` → run. Or wire it to fire overnight with the setup further down.

---

## ⚡ The prompt — copy this

````text
You are a ruthless staff-level engineer doing a 1:1 code review of everything I shipped today in {{REPO_PATH}}. Reviewing me, not the whole repo. Be specific, opinionated, and senior — no "consider maybe possibly." If something is wrong, say it's wrong and show the fix.

INPUTS
- Repo: {{REPO_PATH}}
- Today's work = the diff: {{TODAYS_DIFF}}   (default: `git -C {{REPO_PATH}} diff --since=midnight` plus any uncommitted changes; if empty, fall back to the last {{N_COMMITS}} commits authored by {{GIT_AUTHOR}})
- Stack / domain: {{STACK}}   (e.g. "React + TS + Node, Postgres, payments")
- House style: read {{REPO_PATH}}/CLAUDE.md, .cursorrules, .editorconfig, lint config, and the surrounding code. My code must match THIS codebase's conventions, not generic ones.

STEP 1 — Get the diff. Run the git commands yourself. Read each changed file in full (not just the hunk) so you judge in context. Skip generated/lockfiles/vendored code.

STEP 2 — Review along these axes, hardest-hitting first:
  1. Correctness & bugs — logic errors, off-by-one, null/undefined, race conditions, wrong async/await, error paths swallowed, broken edge cases.
  2. Security & data — injection, authz gaps, secrets, unvalidated input, PII handling.
  3. Design & architecture — wrong abstraction, leaky boundaries, premature coupling, a simpler structure I missed.
  4. Convention drift — where I diverged from THIS repo's existing patterns (naming, error handling, folder layout, test style).
  5. Performance — N+1s, needless re-renders, O(n^2) in hot paths, missing indexes/memoization. Only call it out if it's plausibly real here, not theoretical.
  6. Tests — what's untested that should scare me; the one test that would've caught a real bug.

STEP 3 — Grade me honestly. A skill signal, not flattery. For each finding, tag SEVERITY (CRITICAL / HIGH / MEDIUM / LOW / NIT) and CONFIDENCE (high/med/low). Drop NITs that don't change behavior — I don't want noise at 8am.

STEP 4 — Diagnose ME, not just the code. Across today's findings, what's the *recurring* mistake — the pattern I keep making? Name the 1-3 concepts I'd level up fastest by studying, each with: why it bit me today (cite the exact file:line), and one concrete resource or search query. This is the highest-value section. Be a mentor with a spine.

OUTPUT — write a single file to {{OUTPUT_PATH}}/review-{{DATE}}.md and nothing else. Format exactly:

# Morning Code Review — {{DATE}}
**Verdict:** <one brutal honest sentence: ship it / fix-then-ship / rethink>  ·  **Grade:** <A–F>  ·  <N files, +X/-Y lines>

## Decide now (top 5, ranked by impact × confidence)
For each, a checkbox so I act, not read:
- [ ] **[SEV·conf] <one-line title>** — `path/to/file.ts:42`
  - **What:** the problem in one sentence.
  - **Why it matters:** the concrete consequence (prod break / data loss / future pain).
  - **Fix:** a ready-to-paste diff or exact change. If trivial, just give the corrected code.
  - **Decision:** `[ ] apply  [ ] skip  [ ] discuss`

## Worth a look (medium, collapsed detail)
Same shape, terser.

## Level me up (the real payoff)
The 1-3 concepts to study, each: what I did → the better mental model → one link/search query. Tie every point to a real file:line from today.

## Skipped / assumptions
What I didn't review and why (so you trust the rest).

RULES: Sort strictly by impact then confidence — the thing I should fix first is line 1. Every claim cites a real file:line; if you can't, you're guessing — cut it. Prefer a pasteable fix over prose. Do not modify any source files. Do not open a PR. Just leave me the file. Target a 5-minute read that turns into 30 minutes of high-leverage fixes.
````

---

## 🛠️ Claude workflow + cron/launchd setup (run it while you sleep)

SELF-HOSTED, two files. Assumes the `claude` CLI is on your PATH (run `which claude` to check).

1) SAVE THE SLASH COMMAND (project-level so it ships with the repo, or ~/.claude/commands for global):
mkdir -p ~/.claude/commands and save the `the_prompt` text as ~/.claude/commands/morning-review.md
The {{PLACEHOLDERS}} stay literal in the file — they get filled by the wrapper below. (Use $ARGUMENTS / $1 if you prefer Claude's native arg substitution.)

2) WRAPPER SCRIPT that runs it headless — save as ~/bin/morning-review.sh, then `chmod +x ~/bin/morning-review.sh`:
----
#!/bin/zsh
REPO="$HOME/code/your-project"          # <- your repo
OUT="$HOME/agent-output/reviews"
mkdir -p "$OUT"
DATE=$(date +%F)
DIFF=$(git -C "$REPO" diff --since=midnight 2>/dev/null; git -C "$REPO" diff 2>/dev/null)
claude -p "/morning-review" \
  --add-dir "$REPO" --add-dir "$OUT" \
  --model claude-opus-4-8 \
  --permission-mode acceptEdits \
  --append-system-prompt "REPO_PATH=$REPO  OUTPUT_PATH=$OUT  DATE=$DATE  TODAYS_DIFF below:
$DIFF" \
  >> "$OUT/run-$DATE.log" 2>&1
osascript -e "display notification \"Morning review ready\" with title \"Code Review · $DATE\"" 2>/dev/null
----
(Use --permission-mode acceptEdits because the agent only writes ONE md file in OUT; never give it --dangerously-skip-permissions for a repo you care about.)

3) SCHEDULE OVERNIGHT with launchd (survives sleep better than cron on macOS — runs at next wake if missed). Save as ~/Library/LaunchAgents/com.example.morningreview.plist:
----
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.example.morningreview</string>
  <key>ProgramArguments</key>
  <array><string>/bin/zsh</string><string>-lc</string><string>$HOME/bin/morning-review.sh</string></array>
  <key>StartCalendarInterval</key><dict><key>Hour</key><integer>5</integer><key>Minute</key><integer>30</integer></dict>
  <key>StandardErrorPath</key><string>/tmp/morningreview.err</string>
  <key>StandardOutPath</key><string>/tmp/morningreview.out</string>
</dict></plist>
----
Load it: launchctl load ~/Library/LaunchAgents/com.example.morningreview.plist
Test it now without waiting: launchctl start com.example.morningreview

CRON ALTERNATIVE (if you skip launchd): `crontab -e` then add:
30 5 * * *  /bin/zsh -lc '$HOME/bin/morning-review.sh'
Note: plain cron won't fire if the Mac is asleep at 5:30 — that's why launchd (or `caffeinate`/Mac mini always-on) is the move. Runs at 5:30, review's waiting at the top of your agent-output folder when you open the laptop.

---

## 🌅 What you wake up to

````text
~/agent-output/reviews/review-2026-06-10.md

# Morning Code Review — 2026-06-10
**Verdict:** Fix-then-ship — two real bugs hiding behind passing tests.  ·  **Grade:** B-  ·  9 files, +412/-87 lines

## Decide now (top 5)
- [ ] **[CRITICAL·high] Race condition double-charges on retry** — `src/payments/charge.ts:54`
  - **What:** `await chargeCard()` runs before the idempotency key is persisted, so a retry within ~200ms charges twice.
  - **Why it matters:** Real money, real refunds, real support tickets.
  - **Fix:** persist `idempotencyKey` to Redis *before* the gateway call; bail if it already exists. (diff included below)
  - **Decision:** `[ ] apply  [ ] skip  [ ] discuss`
- [ ] **[HIGH·high] `useEffect` refetches on every keystroke** — `src/components/SearchBar.tsx:31`
  - **What:** missing debounce + `query` in deps fires a request per character.
  - **Why it matters:** ~12 requests per search; you'll feel it at scale.
  - **Fix:** wrap in `useDebouncedCallback(…, 300)`.
  - **Decision:** `[ ] apply  [ ] skip  [ ] discuss`
- [ ] **[HIGH·med] Convention drift: throwing strings, not AppError** — `src/api/orders.ts:88` …
- [ ] **[MEDIUM·high] No test for the empty-cart path** — `__tests__/cart.test.ts` …
- [ ] **[MEDIUM·med] N+1 on order.items in the list endpoint** — `src/api/orders.ts:120` …

## Level me up (the real payoff)
1. **Idempotency in distributed systems** — you keep treating "await finished" as "state is safe." It isn't. The charge bug and the order bug share this root. → search: "idempotency key pattern stripe payments"
2. **React effect dependencies & debouncing** — third time this month an effect fired more than you meant. Mental model: effects describe *synchronization*, not *events*. → "you might not need an effect react docs"

## Skipped / assumptions
Didn't review the generated Prisma client or the e2e snapshots. Assumed Redis is already wired (saw it in `lib/redis.ts`).
````

---

## Why the morning review pays off

You open the laptop to a 5-minute read that's pre-ranked by impact × confidence — no triage, no "where do I even start." The hardest thinking (judging your own code like a stranger would) is already done; you just check boxes and paste fixes, which drops you straight into flow on the highest-leverage work before the day's noise hits. The "Level me up" section compounds: it turns each night's mistakes into a targeted study list, so the same bug stops recurring instead of shipping for the third time. Net: the 30-45 min you'd lose self-reviewing (badly, with your own blind spots) becomes a 30-min execution sprint on problems already vetted by a senior — every single morning, for free, while you slept.
