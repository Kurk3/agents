# 01 · agents in a loop

> an agent that keeps working a goal until a check passes — in a throwaway git worktree, so a broken iteration never touches the code you're actually working in.

- **run mode:** `loop` (runs N iterations, stops on success or max)
- **how to use:** paste the prompt below into your agent → it builds the runner + the check in your repo, then runs it once to prove it loops. fill the `{{PLACEHOLDERS}}` first.

## the two things you have to get right

1. **a machine-checkable goal.** one command, exits 0 = done, non-zero = not done. `npm test`, `npm run lint && npm run build`, a curl that greps the response, a script that returns 1 until the feature exists. if you can't write this command, you don't have a goal yet — you have a wish. write the command first.

2. **isolation.** the loop runs in a git worktree on its own branch. the agent can break anything in there; your working tree is untouched. when it's done you `git diff` the branch and merge what you want. (need to isolate the *runtime* too — installs, services, a real db — go to [docker sandbox](02-docker-sandbox.md).)

## the prompt — copy this

````text
You are setting up an autonomous loop that drives an agent to complete ONE goal in MY repo, iterating until a check passes, fully isolated so a bad iteration can't touch my working tree. Build it, then run it once to prove it loops. Be concrete — write real files, not advice.

CONTEXT
- Repo: {{REPO_PATH}}
- Goal (what "done" means, in one sentence): {{GOAL}}    e.g. "every function in src/api has input validation and a passing test"
- Stack / how I run things: {{STACK}}    e.g. "Node 20, TypeScript, npm test = vitest, npm run lint = eslint"
- Max iterations before it gives up: {{MAX_ITERS}}   default 20

DO THIS
1. DEFINE THE CHECK. Write a single command that exits 0 ONLY when the goal is met and non-zero otherwise. If my existing test/lint/build command already expresses the goal, use it. If not, write a tiny check script (e.g. scripts/loop-check.sh) that asserts the goal — a test that fails today and will pass when done. Show me the command and explain in one line why exit 0 == goal met.

2. ISOLATE. Create a git worktree on a fresh branch so the loop never edits my checkout:
     git worktree add ../loop-{{slug}} -b loop/{{slug}}
   All iterations run inside ../loop-{{slug}}. My current directory stays clean.

3. WRITE THE RUNNER. Create scripts/loop.sh (chmod +x) that, each iteration:
     - runs the check; if it passes, prints "DONE in N iterations" and exits 0
     - if it fails, captures the failure output and calls the agent headless to make ONE focused change toward the goal, passing in the failure so it knows what to fix:
         claude -p "<the iteration prompt below>" \
           --permission-mode acceptEdits \
           --allowedTools "Bash" Read Write Edit \
           --add-dir "$PWD"
     - commits the iteration ("loop: iter N") so every step is reviewable and revertible
     - stops at MAX_ITERS and prints the last failure so I can see where it stalled
   Use a real loop with a counter, not recursion. Log each iteration's check result to loop.log.

4. THE ITERATION PROMPT (what the agent gets each pass) should be tight:
     "Goal: {{GOAL}}. The check `<check cmd>` is currently FAILING with:
      <failure output>.
      Make the SMALLEST change that moves the check toward passing. Do not refactor unrelated code. Do not touch the check itself. After your change, the runner will re-run the check."

5. PROVE IT. Run scripts/loop.sh once now. Show me: the check command, the first 2-3 iterations from loop.log, and whether it converged. If it converged in one iteration, deliberately break the goal so I can watch it loop and recover.

GUARDRAILS
- Never edit the check script from inside the loop (the agent could "pass" by deleting the test). The check is read-only to iterations.
- One change per iteration. Small steps converge; big rewrites thrash.
- If the same failure repeats 3 iterations in a row with no progress, stop and report "stuck: <failure>" — don't burn all iterations spinning.

Output: the files you created, the exact command I run to start a loop, and how I merge the result back (git diff loop/{{slug}}, cherry-pick or merge).
````

## run it unattended (workflow / cron)

once `scripts/loop.sh` works by hand, fire it from your last block of the day so it grinds overnight:

```bash
#!/bin/zsh
# ~/bin/loop-cron.sh
cd "$HOME/code/your-repo" || exit 1
./scripts/loop.sh "feature-x" >> loop.log 2>&1
```

schedule with launchd (survives sleep, unlike plain cron):

```
~/Library/LaunchAgents/com.you.loop.plist  ->  StartCalendarInterval Hour 2
launchctl load ~/Library/LaunchAgents/com.you.loop.plist
```

morning: `git log loop/feature-x` to see what it did, `git diff main..loop/feature-x` to review, merge what you trust.

## why the loop earns its keep

the value isn't "agent writes code." it's that the agent gets the one thing you can't give it while you sleep: **feedback**. it tries, the check fails, it reads the exact failure, it tries again — the same tight loop you run when you're heads-down, except it runs 20 times while you're not there. you stop being the thing that runs the test and reads the error; you become the thing that defined the goal and reviews the result. that's the whole shift.
