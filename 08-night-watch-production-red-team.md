# 08 · Night Watch: Production Red-Team

> While you sleep, an agent attacks today's diff like a hostile SRE and leaves you a ranked list of exactly what will break in prod — each item already triaged into ship / fix / decide.

- **Run mode:** `overnight`
- **How to use:** copy the prompt below → paste it into your agent (Claude Code / Cursor) → fill the `{{PLACEHOLDERS}}` → run. Or wire it to fire overnight with the setup further down.

---

## ⚡ The prompt — copy this

````text
You are a hostile Staff SRE doing a pre-production red-team of code that is about to ship. Your job is NOT to praise the code or summarize it. Your job is to find what breaks in PRODUCTION that the author did not think of, and hand the author a morning decision sheet.

SCOPE
- Repo: {{REPO_PATH}}
- Feature shipped today: {{FEATURE}}
- Today's changes: {{TODAYS_DIFF}}   (e.g. output of `git diff main...HEAD` or `git diff "@{midnight}"`)
- Stack / runtime: {{STACK}}          (e.g. Next.js + Postgres + Vercel + Redis)
- Prod scale assumptions: {{SCALE}}   (e.g. 50 RPS peak, 2M rows in `orders`, single primary DB)

RULES OF ENGAGEMENT
1. Attack ONLY code reachable from today's diff (the changed lines + everything they call/are called by). Trace the blast radius — do not review the whole repo.
2. Assume production reality, not the happy path: real concurrency, cold caches, partial failures, slow networks, malicious input, clock skew, retries, deploys mid-request, and the DB already full of messy legacy data.
3. For every risk you raise, you must answer: HOW does it trigger in prod, WHAT is the user-visible/data impact, and HOW likely is it at {{SCALE}}. No vague "could be improved" — if you can't name the trigger, drop it.
4. Be concrete and falsifiable. Cite file + line/function. Prefer a 2-line proof-of-failure (a request shape, a row state, a race interleaving) over prose.
5. No nitpicks, no style, no "consider adding a comment". Only things that cost an incident, money, data, or a 2am page.

HUNT THESE CLASSES (skip any that don't apply, add ones that do):
- Data integrity: missing transaction/rollback, non-idempotent writes, race conditions, lost updates, partial writes on failure, missing unique constraints.
- Failure modes: unhandled rejection/exception path, missing timeout, no retry vs infinite retry, what happens when {{STACK}} dependency (DB/cache/queue/3rd-party API) is down or slow.
- Scale & performance: N+1, unbounded query/result set, missing index for a new query pattern, full-table scan, payload/memory blowup, anything that's fine at 10 rows and dies at {{SCALE}}.
- Security: authz gap (not just authn), IDOR, injection, SSRF, secrets in logs, missing rate limit on a new public endpoint, PII in plaintext.
- Concurrency & ordering: double-submit, webhook/event replayed or out-of-order, two requests racing the same row, cron overlap.
- Migrations & rollout: backward-incompatible schema/API change, migration that locks a big table, code deployed before/after migration, no feature flag / no kill switch, breaks in-flight requests during deploy.
- Boundaries: null/empty/huge/unicode/negative input, timezone & DST, money/float rounding, pagination edges, enum drift.
- Observability gap: a failure here would be SILENT — no log, no metric, no alert (flag these; silent failures rank higher).

OUTPUT — write a single file to {{REPO_PATH}}/.nightwatch/prod-risk-$(date +%F).md and also print it. Use EXACTLY this format:

# Night Watch — Prod Risk Report ({{FEATURE}})
_Generated {{DATE}} · scope: today's diff · {{SCALE}}_

## ☕ TL;DR (read this in 30s)
- 🔴 Ship-blockers: <n>   🟠 Fix-this-week: <n>   🟡 Note/decide: <n>
- The one thing I'd fix before this hits prod: **<single sharpest risk in one line>**

## 🔴 SHIP-BLOCKERS — do not deploy without a decision
For each, this exact block:

### R1 · <punchy risk title>
- **Where:** `path/file.ext:LINE` (`functionName`)
- **Trigger:** <the exact prod condition that sets it off>
- **Impact:** <user-visible + data/$ consequence> · **Likelihood at scale:** High/Med/Low
- **Proof:** <2-line concrete failure: request shape / row state / race interleaving>
- **Decision for you:** <one crisp choice, e.g. "Wrap steps 2-4 in a tx — or accept partial writes? (I'd wrap.)">
- **Fix sketch:** <1-3 lines or a tiny diff. Do NOT apply it — propose only.>
- **Effort:** S/M/L

## 🟠 FIX THIS WEEK
<same block, terser>

## 🟡 NOTE / DECIDE LATER
<one line each — risk + the decision it implies>

## 🕳️ SILENT FAILURES (no log/metric/alert today)
<bullets: where a failure would go unnoticed in prod, and the one metric/log/alert to add>

## ❓ OPEN QUESTIONS FOR THE AUTHOR
<things only the author knows — answer these and the risk resolves>

CONSTRAINTS
- Rank strictly by blast radius × likelihood at {{SCALE}}. A High-likelihood data-loss bug outranks a clever-but-rare edge case. If you're unsure of likelihood, say so and rank conservatively.
- Hard cap: top 12 risks total. If you found more, keep only the ones that would actually page someone. Quality over a wall of findings.
- Do NOT modify any source file. The only file you write is the report. Read-only red-team.
- If today's diff is trivial/low-risk, say so plainly in the TL;DR and keep the report short. Do not manufacture risk to fill space.
- End with: `## ✅ Author action: <the single highest-leverage next move>`
````

---

## 🛠️ Claude workflow + cron/launchd setup (run it while you sleep)

SELF-HOSTED — runs on your own Mac via your Claude Max subscription (no API bill).

1) Save the prompt as a slash command. Create `~/.claude/commands/prod-risk.md`:

```
---
description: Overnight red-team — what breaks in prod that I didn't think of
---
<paste THE PROMPT here verbatim, then add this line at the bottom:>
Resolve placeholders from $ARGUMENTS; for any not given, infer {{TODAYS_DIFF}} from `git diff "@{midnight}"`, {{DATE}} from `date +%F`, and {{REPO_PATH}} from the current repo. Ask nothing — this runs unattended.
```

2) Test it interactively first (cheap, ~2 min):
```
cd /path/to/your/repo
claude "/prod-risk FEATURE=\"checkout v2\" STACK=\"Next.js + Postgres + Vercel\" SCALE=\"50 RPS, 2M orders\""
```

3) Make a runner script `~/bin/nightwatch.sh` (chmod +x it):
```
#!/bin/zsh
cd "$HOME/code/your-repo" || exit 1
git fetch -q origin
mkdir -p .nightwatch
claude -p \
  "/prod-risk FEATURE=\"$(git log -1 --pretty=%s)\" STACK=\"Next.js + Postgres + Vercel\" SCALE=\"50 RPS peak, 2M orders\"" \
  --permission-mode acceptEdits \
  >> .nightwatch/run.log 2>&1
# optional ping: terminal-notifier -title "Night Watch done" -message "$(ls -t .nightwatch/prod-risk-*.md | head -1)"
```
(`-p` = headless one-shot; `--permission-mode acceptEdits` lets it write the report unattended. Keep it read-only otherwise.)

4) Schedule it nightly at 03:00 with launchd. Create `~/Library/LaunchAgents/com.example.nightwatch.plist`:
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.example.nightwatch</string>
  <key>ProgramArguments</key>
  <array><string>/bin/zsh</string><string>-lc</string><string>$HOME/bin/nightwatch.sh</string></array>
  <key>StartCalendarInterval</key><dict><key>Hour</key><integer>3</integer><key>Minute</key><integer>0</integer></dict>
  <key>StandardErrorPath</key><string>/tmp/nightwatch.err</string>
  <key>StandardOutPath</key><string>/tmp/nightwatch.out</string>
</dict></plist>
```
Load it:
```
launchctl load ~/Library/LaunchAgents/com.example.nightwatch.plist
launchctl start com.example.nightwatch   # run once now to verify
```
(cron equivalent: `crontab -e` → `0 3 * * * /bin/zsh -lc "$HOME/bin/nightwatch.sh"`. launchd is better on macOS — it fires even if the machine was asleep at 03:00, on next wake.)

5) Morning: open `.nightwatch/prod-risk-YYYY-MM-DD.md`. Read the ☕ TL;DR, decide on the 🔴 blockers, done.

Native alternative: Claude Code's `/schedule` (Remote Tasks) can run this in the cloud nightly instead of launchd — use it when you don't want the Mac awake. launchd stays the move for private/offline repos.

---

## 🌅 What you wake up to

````text
.nightwatch/prod-risk-2026-06-11.md

# Night Watch — Prod Risk Report (checkout v2)
_Generated 2026-06-11 · scope: today's diff · 50 RPS peak, 2M orders_

## ☕ TL;DR (read this in 30s)
- 🔴 Ship-blockers: 2   🟠 Fix-this-week: 3   🟡 Note/decide: 4
- The one thing I'd fix before this hits prod: **double-clicking "Pay" charges the card twice — no idempotency key on the Stripe call.**

## 🔴 SHIP-BLOCKERS — do not deploy without a decision

### R1 · Double-charge on retry / double-click
- **Where:** `app/api/checkout/route.ts:88` (`createCharge`)
- **Trigger:** User double-clicks Pay, or the client retries on a slow 504. Two requests, no dedupe.
- **Impact:** Customer charged 2×, manual refund + chargeback risk. · **Likelihood at scale:** High
- **Proof:** Req A and Req B both pass the `status==='pending'` check before either writes `paid` → two `stripe.charges.create` calls fire.
- **Decision for you:** Pass an idempotency key keyed on `orderId` to Stripe — or add a unique partial index on `(order_id) WHERE status='paid'`? (I'd do both.)
- **Fix sketch:** `stripe.charges.create(body, { idempotencyKey: order.id })`
- **Effort:** S

### R2 · Order row updated outside the payment transaction
- **Where:** `lib/orders.ts:140` (`markPaid`)
- **Trigger:** Stripe succeeds, then the process dies before the DB write (deploy mid-request / OOM).
- **Impact:** Money taken, order stuck `pending` forever, silent. · **Likelihood at scale:** Med
- **Proof:** charge() and update() are separate awaits, no tx, no reconciliation job.
- **Decision for you:** Wrap charge+write in an outbox/tx, or add a reconcile cron? (Outbox.)
- **Fix sketch:** record intent row in same tx as order, settle async.
- **Effort:** M

## 🟠 FIX THIS WEEK
- **N+1 on cart render** — `getCart` (`cart.ts:52`) fires 1 query per line item; 40-item cart = 41 queries at 50 RPS. Add a join. (S)
- **No timeout on Stripe call** — default fetch hangs forever if Stripe is slow; exhausts the connection pool. Set 8s timeout. (S)
- **Coupon stacking race** — two tabs apply the same single-use coupon; both succeed (no row lock). (M)

## 🟡 NOTE / DECIDE LATER
- Float used for `discountAmount` (`pricing.ts:31`) → rounding drift. Decide: move to integer cents.
- New `/api/checkout` has no rate limit. Decide: per-user cap.

## 🕳️ SILENT FAILURES (no log/metric/alert today)
- R2's stuck-pending state emits nothing — add metric `orders.paid_no_row` + alert.
- Stripe error path swallows the body (`route.ts:101`) — log `err.code`.

## ❓ OPEN QUESTIONS FOR THE AUTHOR
- Is `/api/checkout` reachable unauthenticated? If yes, R1 + missing rate limit get worse.
- Does a reconciliation job already exist elsewhere? If yes, R2 drops to 🟡.

## ✅ Author action: add the Stripe idempotency key (R1) before deploy — 5 lines, kills the worst incident.
````

---

## Why the morning review pays off

You wake up to the single hardest, highest-stakes thinking already framed as decisions, not a blank repo. Instead of context-switching all morning ("what did I even change yesterday? is this safe to ship?"), you open one file, read a 30-second TL;DR, and make 2-3 real calls while your brain is freshest — that's instant flow state on the work that actually matters. It front-loads the senior-engineer paranoia (concurrency, migrations, silent failures) that you normally only discover in a postmortem at 2am. One prevented prod incident pays for a year of running this; the daily payoff is shipping yesterday's work with conviction before your first coffee instead of after your first outage.
