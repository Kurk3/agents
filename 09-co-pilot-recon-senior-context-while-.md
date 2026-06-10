# 09 · Co-Pilot Recon: Senior Context While You Code

> A second Claude Code tab that researches the live web for the exact thing you're building right now, then drops a ranked "decisions to make" file beside your code so you stop coding blind.

- **Run mode:** `background-while-coding`
- **How to use:** copy the prompt below → paste it into your agent (Claude Code / Cursor) → fill the `{{PLACEHOLDERS}}` → run. Or wire it to fire overnight with the setup further down.

---

## ⚡ The prompt — copy this

````text
You are my SENIOR ENGINEERING RESEARCH CO-PILOT, running in a background tab while I build a feature in the main tab. Your job is NOT to write production code. Your job is to go research the live web so that when I look up, I have senior-level context waiting and a short list of DECISIONS to make — not a wall of text.

## Context (you derive most of this yourself — don't ask me)
- Repo: {{REPO_PATH}}
- Feature I'm building right now: {{FEATURE}}
- What I've touched today (read this first): {{TODAYS_DIFF}}   ← run `git -C {{REPO_PATH}} diff --stat HEAD && git -C {{REPO_PATH}} diff HEAD` if this is empty
- Stack / framework: infer it from package manifests, lockfiles, and imports in the diff. State what you inferred in one line so I can correct you.
- Target users / surface: {{SURFACE}}  (e.g. "public signup form, web + mobile, must be WCAG 2.2 AA")

## Operating rules (non-negotiable)
1. READ-ONLY on my repo. Do not edit, stage, or push anything. You may run read-only git/grep/cat to understand what I'm building.
2. EVERY claim must be backed by a real fetched source with a URL and a date. No claim from memory. If you can't find a current source, label it `UNVERIFIED` and lower its rank — do not bluff. (Memory is stale; the web is the oracle.)
3. Bias to RECENCY. Prefer sources from the last 18 months; flag anything where the ecosystem likely moved (framework majors, browser/API changes, deprecations). Note the version/date the source assumes.
4. Specific to MY diff, not the topic in general. I don't want "accessibility 101." I want the 6 edge cases THIS feature will hit given how I've written it so far.
5. Opinionated. For every decision, give me YOUR recommended default and the one-line reason. I'm reviewing taste, not collecting links.

## What to actually research (scope to {{FEATURE}})
- The 5–10 edge cases / failure modes a senior would catch in this exact feature that a mid-level dev forgets (empty/loading/error/permission/race/locale/offline/large-input — whichever actually apply to my diff).
- Accessibility + UX correctness for this surface (focus management, keyboard, screen-reader labels, error announcement, reduced-motion, hit targets) — only the ones that apply.
- Current best-practice patterns / well-maintained libraries for this exact problem, with a "use this vs build it" call and why.
- Known gotchas in MY stack/version for this feature (open issues, breaking changes, footguns).
- 2–3 reference implementations or specs worth copying, with the specific link.

## Deliverable — write ONE file, nothing else
Write `{{REPO_PATH}}/.research/{{DATE}}-{{FEATURE_SLUG}}.md`. Make it a MORNING-/LOOK-UP-REVIEW artifact I can act on in under 10 minutes. Exact structure:

```
# Recon — {{FEATURE}}  ({{DATE}})
_Inferred stack: <one line>. Diff scanned: <N files, key files>._

## TL;DR — top 3 things to change before you ship (ranked)
1. <decision> — recommended: <do X>. Why: <1 line>. [src](url, date)
2. ...
3. ...

## Decisions to make (ranked by impact × likelihood-you-hit-it)
For each:
### D1 — <short title>   [impact: high|med|low] [confidence: high|med|UNVERIFIED]
- What I probably have now: <inferred from diff>
- The senior move: <opinionated recommendation>
- Edge cases this prevents: <bullet list, concrete to my feature>
- Effort: <S/M/L>  •  Source: [link](url) (YYYY-MM, version assumed)
- ☐ ACCEPT  ☐ REJECT  ☐ NEEDS-ME   ← I tick one in the morning

## Copy-ready (only if obviously correct)
<tiny snippets / config / a11y attributes I can paste — labelled with the source. If not obviously correct, don't include it.>

## Open questions for me
<≤5 things you genuinely couldn't decide without my intent. One line each.>

## Sources
<flat list: title — url — date — why it mattered>
```

Rank ruthlessly. If something is low-impact, it goes at the bottom or gets cut. I'd rather have 5 sharp decisions than 20 mushy ones. End by printing the file path and the count of high-impact decisions so my notifier can surface it.
````

---

## 🛠️ Claude workflow + cron/launchd setup (run it while you sleep)

SELF-HOSTED, two ways: a live background tab (primary use), and a nightly cron pass (so context is waiting before you even start).

1) Save as a slash command (user-level, available in every repo):
mkdir -p ~/.claude/commands
Then create ~/.claude/commands/recon.md with frontmatter + the prompt:
---
description: Background research co-pilot — ranked "decisions to make" for the feature I'm building
allowed-tools: WebSearch, WebFetch, Bash(git*), Bash(grep*), Bash(cat*), Bash(ls*), Read, Write
argument-hint: "<feature> | <surface>"
---
(paste THE PROMPT here; map $1 -> {{FEATURE}}, $2 -> {{SURFACE}}; set {{REPO_PATH}} to the cwd, {{DATE}} via `date +%F`, {{FEATURE_SLUG}} = slugified feature, {{TODAYS_DIFF}} = leave empty so the prompt's git fallback fills it.)

2) Live "second tab" usage (the video's hero shot) — in your repo, in a separate terminal:
cd ~/code/myapp
claude   # then type:  /recon "multi-step signup form" "public web+mobile, WCAG 2.2 AA"
Keep coding in tab 1. When it finishes you have .research/<date>-<slug>.md waiting.

3) Overnight scheduled pass (context ready before you sit down). Use Claude Code's native scheduler first:
claude   # then:  /schedule   -> create a 06:30 daily routine running: /recon "$(git -C ~/code/myapp log -1 --pretty=%s)" "current feature surface"
Native Remote Tasks > cron because it handles auth and notifications for you.

Cron fallback (if you want pure launchd/cron). Wrapper script ~/bin/recon.sh:
  #!/usr/bin/env bash
  set -euo pipefail
  REPO="$HOME/code/myapp"
  cd "$REPO"
  FEATURE="$(git log -1 --pretty=%s)"
  claude -p "/recon \"$FEATURE\" \"see repo\"" --dangerously-skip-permissions \
    --allowedTools "WebSearch,WebFetch,Read,Write,Bash" >> "$REPO/.research/recon.log" 2>&1
  # notify your phone (Pushover/ntfy) that recon is ready:
  curl -s -d "$(ls -t "$REPO"/.research/*.md | head -1)" ntfy.sh/your-recon-topic >/dev/null
chmod +x ~/bin/recon.sh
launchd plist ~/Library/LaunchAgents/com.example.recon.plist (runs 06:30 daily):
  <?xml version="1.0" encoding="UTF-8"?>
  <plist version="1.0"><dict>
    <key>Label</key><string>com.example.recon</string>
    <key>ProgramArguments</key><array><string>$HOME/bin/recon.sh</string></array>
    <key>StartCalendarInterval</key><dict><key>Hour</key><integer>6</integer><key>Minute</key><integer>30</integer></dict>
    <key>StandardErrorPath</key><string>/tmp/recon.err</string>
  </dict></plist>
Load it: launchctl load ~/Library/LaunchAgents/com.example.recon.plist
Add `.research/` to .gitignore so recon notes never pollute a PR. Done — context lands before you do.

---

## 🌅 What you wake up to

````text
.research/2026-06-10-signup-form.md

# Recon — multi-step signup form  (2026-06-10)
_Inferred stack: Next.js 15 + React 19 + react-hook-form + zod. Diff scanned: 4 files, key: SignupForm.tsx, useSignup.ts._

## TL;DR — top 3 to change before you ship
1. Errors aren't announced to screen readers — your inline <span> won't be read on submit. Recommended: aria-live="assertive" region + aria-describedby on each input. Why: blind users get no feedback on failure. [src](mdn, 2025-11)
2. You validate on every keystroke (mode:"onChange") — annoying + jank on mobile. Recommended: validate onBlur, re-validate onChange only after first error. [src](rhf docs, 2026-02)
3. No "session expired" handling on step 3 — token can die mid-flow. Recommended: catch 401, preserve entered state, soft-redirect. [src](next-auth issue #9921, 2026-01)

## Decisions to make (ranked)
### D1 — Screen-reader error announcement  [impact: high] [confidence: high]
- What you have now: inline <span className="error"> rendered on validation fail.
- Senior move: wrap errors in a single aria-live="assertive" region; link inputs via aria-describedby + aria-invalid.
- Edge cases this prevents: SR users submitting blind; focus lost after failed submit; first error not focused.
- Effort: S • Source: [WAI-ARIA APG, 2025-11]
- ☐ ACCEPT  ☐ REJECT  ☐ NEEDS-ME
### D2 — Autocomplete + password-manager tokens  [impact: med] ...
### D3 — Double-submit / slow-network race  [impact: med] ...

## Copy-ready
aria-live region snippet + the 3 input attributes (from WAI-ARIA APG).

## Open questions for me
- Is step-3 email verification required before account creation, or after? Changes the 401 strategy.

5 high-impact decisions. Tick boxes and go.
````

---

## Why the morning review pays off

You sit down, open one file, and the hardest thinking — "what am I forgetting, what does a senior know here that I don't" — is already done and ranked. You're not context-switching to 14 browser tabs mid-flow; you're ticking ACCEPT/REJECT on 5 sharp, sourced decisions. That's instant flow state: the highest-leverage 10 minutes of the day spent deciding, not Googling. Over a week it kills the slow leak of "I'll look that edge case up later" (you won't) and the rework when QA finds the a11y/race bug you'd have shipped. The killer move is the overnight pass: context is waiting before you've had coffee, so you start the day already operating a level above where you'd start cold.
