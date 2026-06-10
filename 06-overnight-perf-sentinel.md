# 06 · Overnight Perf Sentinel

> While you sleep, Claude Code profiles everything you wrote today and leaves a ranked "here's what will blow up in prod, here's the fix" file on your desk by morning.

- **Run mode:** `overnight`
- **How to use:** copy the prompt below → paste it into your agent (Claude Code / Cursor) → fill the `{{PLACEHOLDERS}}` → run. Or wire it to fire overnight with the setup further down.

---

## ⚡ The prompt — copy this

````text
You are a staff performance engineer doing a forensic review of ONLY the code I wrote today. Be skeptical, specific, and ruthless about real-world impact. No fluff, no praise, no generic advice.

REPO: {{REPO_PATH}}
FEATURE / AREA WORKED ON TODAY: {{FEATURE}}
TODAY'S CHANGES: {{TODAYS_DIFF}}   # e.g. `git diff main...HEAD` or `git log --since=midnight -p`
EXPECTED SCALE: {{SCALE}}          # e.g. "5k req/min, 2M rows in `orders`, p95 budget 200ms"
STACK: {{STACK}}                   # e.g. "Next.js 14, Postgres 15 + Prisma, Node 20, Vercel"

## Scope
Analyze ONLY lines that changed in TODAY'S CHANGES, plus the immediate call sites and data they touch. Do not audit the whole repo. If a changed function calls into existing hot code, follow it one level deep — that's where the bottleneck usually hides.

## Hunt for these, in priority order
1. N+1 queries / queries inside loops / missing eager-loads
2. Missing or wrong DB indexes for the WHERE/ORDER BY/JOIN you introduced (name the exact index DDL)
3. O(n^2)+ algorithms, nested loops over request-scoped data, accidental full scans
4. Unbounded data: no pagination/LIMIT, loading full tables/collections into memory, unbounded arrays
5. Blocking the event loop / sync I/O / sequential awaits that should be Promise.all
6. Re-renders & client cost (if frontend): unmemoized expensive work, heavy work in render, oversized payloads to the client
7. Cache misses: recomputing deterministic work, no memo/cache where it's free, cache key bugs
8. Network waterfalls: sequential dependent fetches, chatty APIs, no batching
9. Memory: leaks, retained closures, large allocations per request, missing streaming

## Verify before you claim
For each suspected bottleneck, confirm it's real by reading the surrounding code and data flow. State the trigger condition ("fires when a user has >50 line items"). If you can cheaply measure it, run it: `cd {{REPO_PATH}}` and use EXPLAIN ANALYZE on the actual query, `node --prof`, a quick benchmark loop, or count the queries a code path emits. Prefer evidence over suspicion. Mark anything you could not verify as ⚠️ UNVERIFIED and say what data you'd need.

## Rank by impact
Score each finding: IMPACT (latency/cost/blast-radius at {{SCALE}}) x CONFIDENCE x (1/FIX_EFFORT). Sort the report by that. A 5ms win nobody hits ranks below a 2s p99 spike on the hot path. Be honest when something is a non-issue — say "skip, not worth it."

## OUTPUT — write a single file: {{REPO_PATH}}/reviews/perf-{{DATE}}.md
Make it a decision document I can action in 10 minutes half-awake. Exact format:

```
# Perf Review — {{DATE}} — {{FEATURE}}
TL;DR: <one sentence — the single thing to fix first and why>
Verdict: 🔴 ship-blocker | 🟠 fix this week | 🟢 clean

## Decisions to make (ranked)
### 1. <short title>  [IMPACT: High | CONFIDENCE: High | FIX: 10 min]
- **What:** <the bottleneck, plain language>
- **Where:** `path/to/file.ts:42-58`
- **Why it bites:** <trigger condition + est. cost at {{SCALE}}, e.g. "N+1 → ~120 queries/request, +600ms p95">
- **Evidence:** <EXPLAIN output / query count / benchmark, or ⚠️ UNVERIFIED + what's missing>
- **Fix:** <concrete change; include the diff or the exact index DDL>
- **DECISION:** [ ] apply now  [ ] ticket it  [ ] won't fix — <one-line tradeoff>
### 2. ...
(repeat, ranked)

## Skipped / non-issues
- <thing that looked scary but isn't, one line each — so I don't re-investigate>

## Study (max 3)
- <specific concept + 1 link> tied to a real finding above. No generic reading lists.
```

Rules: ground every finding in a real file:line from today's diff. No more than 7 findings — if there are more, keep the top 7 by score. If the code is genuinely clean, say so loudly and stop; do not invent problems. Total read time of the file should be under ~3 minutes.
````

---

## 🛠️ Claude workflow + cron/launchd setup (run it while you sleep)

## 1. Save the slash command
Create `~/.claude/commands/perf-review.md` (user-level → works in every repo):

```markdown
---
description: Overnight perf review of today's code — ranked fixes, decisions ready by morning
---
<paste the_prompt here, leaving the {{PLACEHOLDERS}} as literal text>

Resolve placeholders from $ARGUMENTS (repo path, feature, scale, stack). For TODAYS_DIFF run `git -C <repo> log --since="6am" -p` (fall back to `git -C <repo> diff` if empty). DATE = `date +%F`.
$ARGUMENTS
```

Test it live first: `cd ~/myrepo && claude` then `/perf-review feature="checkout v2" scale="5k req/min, 2M orders"`. Tune the prompt until the morning file is exactly what you want — THEN schedule it.

## 2. Wrap it in a headless runner script
`~/.claude/scripts/perf-review.sh`:

```bash
#!/bin/zsh
REPO="$HOME/code/myapp"          # <-- your repo
cd "$REPO" || exit 1
mkdir -p "$REPO/reviews"
DATE=$(date +%F)
DIFF=$(git log --since="6am" -p 2>/dev/null || git diff)
[ -z "$DIFF" ] && { echo "no changes today, skipping"; exit 0; }

claude -p \
  "/perf-review repo=$REPO feature=\"$(git log -1 --pretty=%s)\" scale=\"5k req/min\" stack=\"Next.js+Postgres+Prisma\"" \
  --add-dir "$REPO" \
  --model claude-opus-4-8 \
  --permission-mode bypassPermissions \
  --output-format text \
  >> "$REPO/reviews/.perf-run.log" 2>&1
# optional: ping yourself when it's done
osascript -e 'display notification "Perf review ready in /reviews" with title "Overnight agent"'
```
`chmod +x ~/.claude/scripts/perf-review.sh`

## 3. Schedule it overnight with launchd (more reliable than cron on macOS — survives sleep/wake, no Full-Disk-Access cron gotchas)
`~/Library/LaunchAgents/com.example.perfreview.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.example.perfreview</string>
  <key>ProgramArguments</key>
  <array><string>/bin/zsh</string><string>-lc</string>
    <string>$HOME/.claude/scripts/perf-review.sh</string></array>
  <key>StartCalendarInterval</key>
  <dict><key>Hour</key><integer>3</integer><key>Minute</key><integer>0</integer></dict>
  <key>StandardErrorPath</key><string>/tmp/perfreview.err</string>
  <key>StandardOutPath</key><string>/tmp/perfreview.out</string>
</dict></plist>
```
Load it: `launchctl load ~/Library/LaunchAgents/com.example.perfreview.plist`
Fire it once now to verify: `launchctl start com.example.perfreview` → check `~/code/myapp/reviews/`.
Keep the Mac awake at 3am: `sudo pmset repeat wakeorpoweron MTWRFSU 02:55:00`.

(Plain cron alternative if you prefer: `crontab -e` → `0 3 * * * $HOME/.claude/scripts/perf-review.sh` — but launchd is better on a laptop that sleeps.)

---

## 🌅 What you wake up to

````text
File waiting at `~/code/myapp/reviews/perf-2026-06-11.md`:

```
# Perf Review — 2026-06-11 — checkout v2
TL;DR: getCart() fires an N+1 — every cart line hits the DB separately; kills p95 at scale.
Verdict: 🔴 ship-blocker

## Decisions to make (ranked)
### 1. N+1 in cart total  [IMPACT: High | CONFIDENCE: High | FIX: 10 min]
- What: We loop cart.items and `await prisma.product.findUnique` per item.
- Where: `src/server/cart.ts:73-88`
- Why it bites: cart of 40 items → 41 queries. At 5k req/min that's +650ms p95, ~200k extra qps on Postgres.
- Evidence: instrumented the path → 41 queries logged. EXPLAIN on findUnique = index OK, it's the count.
- Fix: one `findMany({ where: { id: { in: ids } } })` + map. Diff included below.
- DECISION: [ ] apply now  [ ] ticket  [ ] won't fix

### 2. Missing index on orders.user_id  [IMPACT: Med | CONFIDENCE: High | FIX: 1 migration]
- Where: `src/server/orders.ts:21`  — new `WHERE user_id = $1 ORDER BY created_at DESC`
- Evidence: EXPLAIN ANALYZE → Seq Scan, 2.1M rows, 480ms.
- Fix: `CREATE INDEX CONCURRENTLY idx_orders_user_created ON orders(user_id, created_at DESC);` → 480ms→3ms.
- DECISION: [ ] apply now  [ ] ticket  [ ] won't fix

## Skipped / non-issues
- `formatPrice()` called in render — memoized already, <0.1ms. Skip.

## Study (max 3)
- Prisma `in` batching vs DataLoader — relevant to finding #1.
```
````

---

## Why the morning review pays off

You wake up to a single 3-minute file that has already done the expensive part — finding WHERE the bottleneck is and PROVING it with EXPLAIN/query-counts. No staring at a blank profiler. You just walk a ranked list of decisions with checkboxes: apply, ticket, or won't-fix. That's the hardest, highest-leverage review work of the day done before your first coffee — which drops you straight into flow instead of "waiting for the agent to think" at 2pm. One real N+1 or missing index caught before prod saves hours of incident response and a p99 fire. Run it nightly and your hot paths simply never rot.
