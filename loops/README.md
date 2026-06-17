# agents in a loop

the overnight agents in the root run once and hand you an artifact. these are different: an agent that runs **again and again against a goal**, in an isolated environment, until the goal is actually met — then stops.

the trick that makes it safe: the agent never touches the place you work. it runs in a sandbox (a git worktree, a docker container, a throwaway branch). it can install junk, break the build, nuke the db — none of it reaches your machine. you wake up / come back to a branch that either passes the check or a log telling you why it couldn't.

the trick that makes it *finish*: the goal has to be **machine-checkable**. one command that exits 0 when done (tests green, lint clean, the page renders, the endpoint returns 200). no exit-0 check = the loop can't tell it's done = it runs forever or quits on vibes. write the check first.

## the loop, in one shape

```
loop:
  agent makes one change toward the goal
  run the check
  check passes?  -> stop, report success
  check fails?   -> feed the failure back in, go again
  hit max iterations? -> stop, report where it got stuck
```

that's it. everything below is just *where* the loop runs and *how* the check is wired.

## the guides

| # | guide | what it sets up |
|---|-------|-----------------|
| 01 | [agents in a loop](01-agents-in-a-loop.md) | the core loop runner + the exit-0 check, in a git worktree |
| 02 | [docker sandbox](02-docker-sandbox.md) | run the loop inside a container so the agent can yolo without risking your host |
| 03 | [playwright loop](03-playwright-loop.md) | close the loop with a real browser — the check *is* the app working |

read 01 first — it's the engine. 02 and 03 are the two ways i isolate it and the two ways i let it check its own work.

## how to use any of them

each file has a **copy-paste prompt**. you don't run the prompt to do the audit — you paste it into your agent and it **builds the setup in your project**: writes the runner, the check, the dockerfile, the playwright spec, wires it together, runs it once to prove it works. fill the `{{PLACEHOLDERS}}`, paste, let it cook.

self-hosted on your own machine with your claude subscription. no api key.

---

running it *is* a workflow. the first version will be clunky. fine-tune the check and the iteration prompt every time you use it — a sharp loop is worth more than a fancy model.
