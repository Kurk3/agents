# overnight agents

ai prompts that do the work while you sleep. you wake up and just review.

**how to use:** copy the folder, or pick the one agent you like → throw it into your agent (claude code / cursor) → fill the blanks → you're good to go :)

i let these run overnight (or in a background tab while i code), and every morning my laptop already has finished work waiting — PRs, designs, perf reports, edge-case audits. then i just review and decide. that's the whole trick.

## the agents

| # | agent | what it does while you sleep |
|---|-------|------------------------------|
| 01 | [edge-case hunter](01-overnight-edge-case-hunter.md) | finds the inputs & states you forgot in today's diff (ranked p0/p1/p2) |
| 02 | [night auditor](02-the-night-auditor-best-practices-pas.md) | best-practices pass on the code you wrote today |
| 03 | [tech-debt sentry](03-tech-debt-sentry-overnight.md) | catches where your new code drifts from your own patterns |
| 04 | [sleep-to-ship](04-sleep-to-ship-10-design-options-wait.md) | 10+ accessible design options in your design system |
| 05 | [morning code review](05-the-morning-code-review.md) | a senior 1:1 on today's code + what to study next |
| 06 | [perf sentinel](06-overnight-perf-sentinel.md) | what blows up in prod at scale, and the exact fix |
| 07 | [overnight analyzer](07-overnight-analyzer-wake-up-to-decisi.md) | turns a hard question into a researched decision brief |
| 08 | [night watch](08-night-watch-production-red-team.md) | a hostile SRE red-teams your diff for prod risks |
| 09 | [co-pilot recon](09-co-pilot-recon-senior-context-while-.md) | a 2nd tab researches the web for whatever you're building right now |

each file has three things: the **prompt** (copy-paste), the **claude slash-command**, and the **cron/launchd** schedule to fire it overnight.

## more in here

the overnight agents above run once. the rest of the repo is the workflow around them:

- [loops/](loops/) — agents that run **again and again against a goal** in an isolated environment until a check passes. how i run agents in a loop, sandbox them in docker, and close the loop with playwright.
- [habits/](habits/) — the daily practices around the tools. start with the [dev journal](habits/dev-journal.md).
- [skills/](skills/) — single-purpose prompts i reuse, like [deload-brain](skills/deload-brain.md) for reading a codebase fast.

## run it overnight (3 moves)

1. save it as a slash command → `~/.claude/commands/<name>.md`
2. wrap it in a tiny runner → `claude -p "/<name> ..."`
3. schedule it → macOS **launchd** (or cron)

> ⚠️ use launchd, not plain cron — cron skips the job if your mac was asleep at 3am, and you wake up to an empty folder. launchd catches up on wake.

self-hosted on your own machine with your claude subscription. no api key.

---

steal these. tweak them for your stack. build a better one? open a PR — i'll run it.
