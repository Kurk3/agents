# 04 · Sleep-to-Ship: 10 Design Options Waiting at Breakfast

> Overnight, an agent designs feature N inside YOUR existing design system, generates 10+ accessible options, and hands you one ranked markdown file where every decision is a checkbox.

- **Run mode:** `overnight`
- **How to use:** copy the prompt below → paste it into your agent (Claude Code / Cursor) → fill the `{{PLACEHOLDERS}}` → run. Or wire it to fire overnight with the setup further down.

---

## ⚡ The prompt — copy this

````text
You are a Staff Product Designer + senior frontend engineer running OVERNIGHT and UNATTENDED. Nobody answers questions until morning. When unsure, make a defensible choice, log the assumption, and keep going. Your job: produce a morning-review artifact so good I can wake up and just DECIDE.

# Mission
Design the UI for: {{FEATURE}}
Repo: {{REPO_PATH}}
Today's diff (context for what just shipped / what this connects to): {{TODAYS_DIFF}}
Output file: {{REPO_PATH}}/design-review/{{DATE}}-{{FEATURE_SLUG}}.md

# STAGE 0 — GROUND (do this FIRST, do not skip — skipping = frankenstein UI)
Before proposing anything, map the system so every option is an EXTENSION of it, not a foreign object:
- Find and read the design tokens / theme (search for: tailwind.config, theme.*, tokens.*, *.css custom props, styled-system, CSS vars, design-system or ui package). List the actual color, spacing, radius, typography, shadow, z-index tokens by name.
- Find 2-3 EXISTING components closest to {{FEATURE}} (buttons, forms, modals, cards, tables). Read them. Note the real patterns: how they handle state, variants, sizing, focus rings, loading, empty, error.
- Note the framework + styling approach (React/Vue/Svelte? Tailwind/CSS-modules/styled-components?) and the a11y baseline already in use (aria patterns, focus management, any existing axe/eslint-jsx-a11y config).
- Write a 6-line "System Snapshot" at the top of the output: stack, token source, 3 reused components, a11y baseline, one anti-pattern to avoid (something the codebase clearly never does).
If you cannot find a design system, say so explicitly and derive a minimal token set from the most-repeated values in the CSS — do NOT invent a new aesthetic.

# STAGE 1 — GENERATE >= 10 DISTINCT OPTIONS
Each option must be genuinely different in APPROACH, not a color swap. Span this space:
information density (compact ↔ spacious), layout (modal / drawer / inline / full-page / wizard / split), progressive disclosure, primary interaction model, empty/loading/error treatment, and mobile-first vs desktop-first. At least 2 options must be deliberately "boring + safe" (ship-today) and at least 1 deliberately ambitious (needs buy-in).

For EACH option produce:
- **Name + one-line pitch** (what makes it different)
- **When it wins** (the user/context this is best for)
- **Reuses** (exact existing components + tokens it builds on — name them)
- **New surface** (what genuinely new component/pattern it introduces, if any — fewer is better)
- **Accessibility** (keyboard path, focus order, ARIA roles, contrast check against the real tokens, what a screen reader announces — be concrete, cite WCAG 2.2 criteria like 2.4.7, 1.4.3, 4.1.2 where relevant)
- **Effort** (S/M/L) + **Risk** (what could go wrong)
- **A real artifact**: a working component code sketch in the repo's ACTUAL stack/styling using REAL token names (not pseudocode), OR an ASCII/markdown wireframe if layout is the point. Code must compile-plausibly and import from the real design system paths you found in Stage 0.

# STAGE 2 — SELF-CRITIQUE (the design oracle, since there are no tests for taste)
You have no visual test, so simulate one. For every option run these gates and KILL or downgrade failures:
1. CONTINUITY: does it sit inside the System Snapshot or stick out as a foreign element? Anything that needs a new token/font/shadow not already in the system gets flagged.
2. A11Y: keyboard-only operable? Focus visible? Contrast >= 4.5:1 (3:1 large)? No ARIA-only-no-semantics? Fail = downgrade.
3. STATES: does it specify empty, loading, error, and success? Missing states = downgrade.
4. EFFORT-VS-PAYOFF: is the ambition worth the build cost given today's diff context?
Drop or merge the weakest until you have a clean set. Keep >= 10 surviving options.

# STAGE 3 — RANK + DECISION ARTIFACT (this is what I read in the morning)
Write {{REPO_PATH}}/design-review/{{DATE}}-{{FEATURE_SLUG}}.md with EXACTLY this structure:

1. **TL;DR (read in 30s)** — your single top recommendation, the runner-up, and the one bold bet. 3 sentences.
2. **System Snapshot** (from Stage 0).
3. **Decision Queue** — a checklist of the 3-5 actual choices I must make, each as `- [ ]`, phrased as a concrete fork (e.g. "Modal vs Drawer for the create flow → I recommend Drawer because <reason>"). This is the point of the whole doc: I should be able to check boxes and be done.
4. **Ranked Options Table** — # | Name | Approach | Reuses | New surface | A11y verdict | Effort | Risk | Score(1-10). Sorted best-first.
5. **Full option detail** — the per-option sections from Stage 1, in ranked order, each with its code sketch / wireframe in a fenced block.
6. **Open questions for me** — anything you assumed that I should confirm, max 5, each with the assumption you made so nothing blocks until morning.
7. **What I'd do next** — the exact first 30 min of implementation for your #1 pick.

# RULES
- Work on a NEW branch `design/{{FEATURE_SLUG}}-{{DATE}}`. Commit the markdown + any throwaway code sketches. NEVER push, NEVER touch main, NEVER edit production components.
- Stay strictly inside the real design system. Reusing an existing token beats inventing a "nicer" one — always.
- Opinionated > neutral. I want a recommendation with a reason, not a menu I have to evaluate from scratch.
- If you hit a budget/time limit, ship the artifact with however many options are done and mark it `PARTIAL` at the top — a ranked 7 I can act on beats a perfect 12 I never get.
````

---

## 🛠️ Claude workflow + cron/launchd setup (run it while you sleep)

1) Save the prompt as a slash command (the {{...}} become $ARGUMENTS / env you fill at call time):

  mkdir -p ~/.claude/commands
  $EDITOR ~/.claude/commands/design-proposal.md   # paste the_prompt, swap {{FEATURE}} for $ARGUMENTS

Test it interactively first (this is the whole game — a slash command that works by hand):

  cd /path/to/repo
  claude "/design-proposal create-team invite flow"

2) Wrap it in a headless runner script that fills the placeholders and runs unattended. Save as ~/bin/design-overnight.sh:

  #!/usr/bin/env bash
  set -euo pipefail
  REPO="$1"; FEATURE="$2"
  SLUG=$(echo "$FEATURE" | tr '[:upper:] ' '[:lower:]-' | tr -cd 'a-z0-9-')
  DATE=$(date +%F)
  cd "$REPO"
  git fetch -q && git switch -c "design/$SLUG-$DATE" 2>/dev/null || git switch "design/$SLUG-$DATE"
  DIFF=$(git log --since=midnight -p | head -c 8000)   # today's work as context
  mkdir -p design-review
  claude -p "$(sed -e "s|{{REPO_PATH}}|$REPO|g" -e "s|{{FEATURE}}|$FEATURE|g" \
      -e "s|{{FEATURE_SLUG}}|$SLUG|g" -e "s|{{DATE}}|$DATE|g" \
      -e "s|{{TODAYS_DIFF}}|$DIFF|g" ~/.claude/commands/design-proposal.md)" \
    --permission-mode acceptEdits --allowedTools "Read,Grep,Glob,Write,Edit,Bash(git*)" \
    > "design-review/$DATE-$SLUG.log" 2>&1
  git add -A && git commit -q -m "design: $FEATURE — overnight proposals ($DATE)"
  command -v terminal-notifier >/dev/null && terminal-notifier -title "Designs ready" -message "$FEATURE"

  chmod +x ~/bin/design-overnight.sh

3) Schedule it overnight with macOS launchd (survives sleep better than cron; replace YOURUSER + repo + feature). Save as ~/Library/LaunchAgents/com.example.design-overnight.plist:

  <?xml version="1.0" encoding="UTF-8"?>
  <plist version="1.0"><dict>
    <key>Label</key><string>com.example.design-overnight</string>
    <key>ProgramArguments</key><array>
      <string>/Users/YOURUSER/bin/design-overnight.sh</string>
      <string>/Users/YOURUSER/code/myapp</string>
      <string>team invite flow</string>
    </array>
    <key>StartCalendarInterval</key><dict><key>Hour</key><integer>2</integer><key>Minute</key><integer>0</integer></dict>
    <key>StandardErrorPath</key><string>/tmp/design-overnight.err</string>
    <key>StandardOutPath</key><string>/tmp/design-overnight.out</string>
  </dict></plist>

  launchctl load ~/Library/LaunchAgents/com.example.design-overnight.plist   # runs 02:00 nightly

Cron equivalent (Linux / Mac mini that stays awake): `0 2 * * * /Users/YOURUSER/bin/design-overnight.sh /Users/YOURUSER/code/myapp "team invite flow" >> /tmp/design.log 2>&1`

Morning: `git switch design/<slug>-<date> && open design-review/<date>-<slug>.md`, check the Decision Queue boxes, done.

---

## 🌅 What you wake up to

````text
# Design Proposals — Team Invite Flow (2026-06-11)

**TL;DR:** Ship **Option 1 (Inline Row Expander)** — it reuses your existing `<DataTable>` + `Button` and adds zero new components. Runner-up: **Option 3 (Right Drawer)** if invites grow past 3 fields. Bold bet: **Option 7 (Bulk Paste + Chips)** — power-user magic, but needs a new `<TokenInput>`.

**System Snapshot:** Next.js + Tailwind · tokens in `tailwind.config.ts` (`brand-500`, `space-4`, `radius-md`) · reuses `DataTable`, `Button`, `Field` · a11y baseline: `eslint-jsx-a11y` on, focus-trap in `Modal` · anti-pattern to avoid: this codebase never nests modals.

**Decision Queue (check and you're done):**
- [ ] Surface: Inline-row vs Drawer vs Modal → I recommend **Inline-row** (zero new surface, matches your settings tables)
- [ ] Roles: dropdown vs segmented control → **segmented** (only 3 roles, faster, fewer clicks)
- [ ] Error model: per-row inline vs top banner → **per-row** (matches `Field` error pattern you already ship)

| # | Name | Approach | Reuses | New surface | A11y | Effort | Risk | Score |
|---|------|----------|--------|-------------|------|--------|------|-------|
| 1 | Inline Row Expander | Expand a table row to invite | DataTable, Button, Field | none | PASS (2.4.7,1.4.3) | S | low | 9 |
| 2 | Classic Modal | Centered form | Modal, Field | none | PASS | S | low | 7 |
| 3 | Right Drawer | Slide-over multi-invite | Drawer, Field | none | PASS | M | med | 8 |
| … | (7 more) | … | … | … | … | … | … | … |

**Open questions (assumed, confirm):** (1) Assumed max 25 invites/batch — no pagination built. (2) Assumed roles = Admin/Member/Viewer from `roles.ts`.

**Next 30 min for #1:** copy `SettingsTableRow.tsx` → add expand state → drop `Field` x2 + role `SegmentedControl` → wire `useInviteMember` mutation.
````

---

## Why the morning review pays off

You replace the worst part of design work — the blank-canvas "what should this even look like" stall that eats your freshest morning hours — with a checklist. The agent already did the boring 80%: read your design system, generated 10+ options, killed the inaccessible ones, ranked them, and wrote a Decision Queue. You sit down with coffee, do the one thing only you can do (taste + judgment), check 3 boxes, and you're in flow on implementation by minute 10 instead of hour 2. Because it's grounded in YOUR tokens and components (Stage 0), the winning option is paste-ready, not a Dribbble fantasy you'd have to translate. One overnight run = a half-day of design exploration compressed into a 5-minute decision.
