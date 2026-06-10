# 03 · Tech-Debt Sentry — Overnight

> While you sleep, a Claude Code agent diffs the code you wrote today, hunts where it drifts from your codebase's real patterns, and leaves a ranked decision list you just say yes/no to over coffee.

- **Run mode:** `overnight`
- **How to use:** copy the prompt below → paste it into your agent (Claude Code / Cursor) → fill the `{{PLACEHOLDERS}}` → run. Or wire it to fire overnight with the setup further down.

---

## ⚡ The prompt — copy this

````text
ROLE: You are the staff engineer who owns {{REPO_PATH}}. You are reviewing ONLY the code written today, with one job: make it indistinguishable from code the best engineer on this team would have shipped. You are not adding features. You are removing tech debt the author introduced before it compounds.

CONTEXT YOU MUST GATHER FIRST (do not skip — guessing patterns is the #1 failure mode):
1. Today's changes = {{TODAYS_DIFF}} (default: `git -C {{REPO_PATH}} diff "@{6.hours.ago}" --stat` plus the full diff). These are the files under review. Touch nothing else.
2. Learn THIS repo's actual conventions before judging anything. Read, in order: README, CONTRIBUTING, {{RULES_FILES}} (e.g. .cursorrules, CLAUDE.md, .eslintrc, eslint.config.*, tsconfig, .editorconfig), and 3-5 NEIGHBORING files that already do something similar to today's code (same folder, same layer). The bar is "match what this codebase already does well," NOT some abstract best practice. If the repo consistently does X, X is correct here even if you'd personally do Y.
3. Identify the project's existing primitives: error handling, logging, data fetching, state, validation, naming, file/folder structure, test style. Today's code must reuse these, not reinvent them.

NOW FIND TECH DEBT introduced by today's code — concretely, in priority order:
- DRIFT: reimplements something the codebase already has a util/hook/component/helper for (copy-paste, parallel abstraction, a second way to do an existing thing).
- INCONSISTENCY: diverges from established patterns — error handling, async/await vs promises, naming, folder placement, import style, the team's component or module shape.
- LEAKY / WRONG ABSTRACTION: wrong layer (logic in the view, fetch in the component, business rule in the controller), prop-drilling where context/store exists, God-functions, boolean-trap params.
- MISSING SAFETY THE REPO NORMALLY HAS: types/`any` escapes, unhandled error/loading/empty states, missing input validation, no test where siblings are tested.
- DEAD ON ARRIVAL: unused exports, commented-out code, console.logs, TODOs, speculative generality (abstractions with one caller), magic numbers/strings the repo elsewhere names.
- RISKY SHORTCUTS: silent catches, `// @ts-ignore`/`eslint-disable` without reason, N+1 / accidental quadratic loops, mutation of shared state.

RULES OF JUDGEMENT (respect my time — this is a morning decision queue, not a lint dump):
- Report ONLY real debt in TODAY'S code. No nitpicks, no style opinions the linter already enforces, no "consider maybe." If you wouldn't block a PR on it, drop it.
- Every item must cite the established pattern it violates with a real path:line from THIS repo (e.g. "src/lib/api/client.ts already wraps fetch — use it"). No citation = you're guessing = delete the item.
- Rank by (impact if left × likelihood it bites) — highest first.
- For each item give a SMALL, surgical fix. Drop-in code, not a rewrite. If the fix is debatable or architectural, mark it DECISION and give me 2 options with a one-line recommendation.

OUTPUT: write a single file to {{OUTPUT_PATH}} (default `{{REPO_PATH}}/.morning/tech-debt-{{DATE}}.md`). Nothing else — do NOT modify source, do NOT commit. Exact structure:

# Tech-Debt Review — {{DATE}}
**Files reviewed:** <count> · **Items:** <n> · **Est. cleanup:** <minutes>
**TL;DR:** <one sentence: is today's code clean, or where is the rot?>

## ⚡ Decide in 60 seconds (ranked)
For each item:
### N. <punchy title> — `path:line`
- **Debt:** <what's wrong, 1 line>
- **Violates:** <the existing repo pattern + path:line proving it exists>
- **Impact if kept:** <concrete future pain>
- **Fix:** <surgical diff or 2-4 line snippet>
- **Decision:** ✅ Apply as-is  /  🔀 Pick option (A: … / B: …, rec: …)  /  ⏭️ Skip-if <condition>
- **Effort:** <S/M/L + minutes>

## 🟢 Already good
<2-3 things today's code did RIGHT and consistent with the repo — so I trust the review and don't second-guess everything>

## 📋 Apply queue
A copy-paste block listing the path:line of every ✅ item, so I can hand the approved ones straight back to you: "Apply items 1,3,4 from {{OUTPUT_PATH}}."

End. Do not ask me questions — make the call and document it. I review at 8am.
````

---

## 🛠️ Claude workflow + cron/launchd setup (run it while you sleep)

Self-hosted on your own Mac. Two steps: save the command, then schedule it.

STEP 1 — Save as a slash command (so it's reusable + testable by hand first):
Save the_prompt to `~/.claude/commands/tech-debt-sweep.md` with this frontmatter on top, replacing the {{...}} you want fixed and leaving the rest as runtime args:

```
---
description: Overnight tech-debt review of today's diff vs repo patterns
---
<paste the_prompt here, then add at the bottom:>
Extra focus / overrides for this run:
$ARGUMENTS
```

Test it live once before trusting cron: open Claude Code in the repo and run `/tech-debt-sweep`. Fix anything weird, THEN automate.

STEP 2 — Schedule overnight with launchd (survives sleep/reboot better than cron on macOS; `StartCalendarInterval` fires the next wake if the Mac was asleep).

2a. Create the runner script `~/bin/tech-debt-sweep.sh` (chmod +x):
```bash
#!/bin/zsh
REPO="$HOME/code/my-app"          # <- your repo
DATE=$(date +%F)
mkdir -p "$REPO/.morning"
cd "$REPO" || exit 1
claude -p "/tech-debt-sweep" \
  --add-dir "$REPO" \
  --model claude-opus-4-8 \
  --permission-mode acceptEdits \
  --append-system-prompt "TODAYS_DIFF = git diff since 6h ago. OUTPUT_PATH = $REPO/.morning/tech-debt-$DATE.md. DATE = $DATE. Write only that file." \
  > "$REPO/.morning/run-$DATE.log" 2>&1
# Ping yourself it's ready (optional, needs: brew install terminal-notifier)
terminal-notifier -title "Tech-Debt review ready" -message "$REPO/.morning/tech-debt-$DATE.md" 2>/dev/null || true
```
Note: `acceptEdits` lets it WRITE the .md to .morning/ unattended without touching source (the prompt forbids editing src). Use `--dangerously-skip-permissions` only if you sandbox the repo.

2b. Create `~/Library/LaunchAgents/com.example.techdebt.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.example.techdebt</string>
  <key>ProgramArguments</key>
  <array><string>/bin/zsh</string><string>-lc</string>
    <string>$HOME/bin/tech-debt-sweep.sh</string></array>
  <key>StartCalendarInterval</key>
  <dict><key>Hour</key><integer>3</integer><key>Minute</key><integer>0</integer></dict>
  <key>StandardErrorPath</key><string>/tmp/techdebt.err</string>
  <key>StandardOutPath</key><string>/tmp/techdebt.out</string>
</dict></plist>
```

2c. Load it:
```bash
launchctl unload ~/Library/LaunchAgents/com.example.techdebt.plist 2>/dev/null
launchctl load  ~/Library/LaunchAgents/com.example.techdebt.plist
launchctl start com.example.techdebt   # force one run now to verify
```
Keep the Mac plugged in; allow wake-for-network or just leave display sleep on (caffeinate not needed for launchd). cron alternative if you prefer: `0 3 * * * /bin/zsh -lc '$HOME/bin/tech-debt-sweep.sh'` (but cron won't fire if the Mac was asleep at 3am — launchd will catch up on wake).

---

## 🌅 What you wake up to

````text
# Tech-Debt Review — 2026-06-10
**Files reviewed:** 6 · **Items:** 4 · **Est. cleanup:** ~18 min
**TL;DR:** Feature works, but the new checkout hook reimplements fetching and swallows errors — two fixes will make it match the rest of the app.

## ⚡ Decide in 60 seconds (ranked)
### 1. Hand-rolled fetch instead of the shared client — `src/features/checkout/useCheckout.ts:14`
- **Debt:** raw `fetch()` with manual JSON parse + no auth header.
- **Violates:** `src/lib/api/client.ts:8` (`apiClient.post`) — every other feature uses it; it adds auth + retry.
- **Impact if kept:** unauthenticated calls in prod, third parallel way to fetch, no retry.
- **Fix:** `const data = await apiClient.post('/checkout', payload)` — delete the 9 manual lines.
- **Decision:** ✅ Apply as-is
- **Effort:** S · 4 min

### 2. Silent catch hides payment failures — `src/features/checkout/useCheckout.ts:31`
- **Debt:** `catch (e) {}` — error never surfaces to UI.
- **Violates:** `src/features/cart/useCart.ts:22` sets `error` state + toasts via `useToast`.
- **Impact if kept:** user sees spinner forever on a failed charge. Support tickets.
- **Fix:** `catch (e) { setError(e); toast.error('Checkout failed') }`
- **Decision:** ✅ Apply as-is
- **Effort:** S · 3 min

### 3. Business rule lives in the component — `src/features/checkout/CheckoutForm.tsx:48`
- **Debt:** tax math inlined in JSX.
- **Violates:** repo keeps pricing in `src/domain/pricing.ts` (see `calcSubtotal`).
- **Impact if kept:** tax logic untested + duplicated when reused.
- **Decision:** 🔀 A: move to `pricing.ts` as `calcTax()` (rec — testable, matches domain layer) · B: leave, add a TODO.
- **Effort:** M · 8 min

### 4. `any` on the payload — `src/features/checkout/types.ts:3`
- **Debt:** `payload: any`.
- **Violates:** strict types everywhere else; `CartItem` already exists.
- **Fix:** `payload: { items: CartItem[]; coupon?: string }`
- **Decision:** ✅ Apply as-is
- **Effort:** S · 3 min

## 🟢 Already good
- File/folder layout matches the `features/<x>/` convention exactly.
- Reused `<Button variant="primary">` from the design system instead of a new button.

## 📋 Apply queue
Approved → `useCheckout.ts:14`, `useCheckout.ts:31`, `types.ts:3`
Tell me: "Apply items 1,2,4 from .morning/tech-debt-2026-06-10.md"
````

---

## Why the morning review pays off

You skip the worst part of a morning: reloading yesterday's mental context and second-guessing your own code. Instead of "is this fetch fine? did I handle the error? does this match how we do things?" — that audit already ran at 3am against your ACTUAL repo patterns, with path:line proof, ranked by what'll actually bite you. You open one file, make 4 yes/no calls in 90 seconds, fire back "apply items 1,2,4," and you're in flow on real work before your coffee's cold. The debt that normally rots for months because nobody re-reviews fresh code gets killed the same night it's born — compounding cleanliness while you sleep, zero willpower spent.
