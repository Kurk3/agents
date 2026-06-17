# 02 · docker sandbox

> run the loop inside a container so the agent can install, run, break, and nuke anything — and none of it reaches your machine. this is what lets you run it with permissions wide open and actually sleep.

- **run mode:** `loop` inside a container
- **how to use:** paste the prompt below into your agent → it writes the Dockerfile + compose + run script that sandboxes the loop in your repo, then builds and runs it once. fill the `{{PLACEHOLDERS}}` first.

## why a container and not just a worktree

a [git worktree](01-agents-in-a-loop.md) isolates the *files*. it does nothing about the *runtime* — the agent can still `rm -rf` your home, `npm install` a malicious package onto your host, hammer your real database, leave services running. once you want the agent to run with permissions wide open (the only way it works unattended), the host becomes the blast radius.

a container moves the blast radius inside a box. mount the repo, give it the toolchain, let it run claude with permissions skipped. worst case you `docker rm` the container and you've lost nothing. that's the trade: a box you can throw away vs. your actual machine.

## the prompt — copy this

````text
You are containerizing an autonomous agent loop for MY repo so it runs FULLY isolated — the agent can do anything inside the container and nothing reaches my host. Build the sandbox, then run the loop inside it once to prove it works. Write real files.

CONTEXT
- Repo: {{REPO_PATH}}
- Stack / base image: {{STACK}}    e.g. "Node 20 -> node:20-bookworm" / "Python 3.12 -> python:3.12-slim"
- Goal the loop chases: {{GOAL}}
- The check command (exits 0 when done): {{CHECK_CMD}}    e.g. "npm test"
- Services the goal needs running: {{SERVICES}}    e.g. "postgres, redis" or "none"

DO THIS
1. DOCKERFILE (Dockerfile.loop). Start from the base image for my stack. Install: my toolchain, git, and the Claude Code CLI (npm i -g @anthropic-ai/claude-code). Create a non-root user and work as it. Do NOT bake any secret or API key into the image.

2. AUTH WITHOUT LEAKING KEYS. I run on my Claude subscription, not an API key. Mount my ~/.claude credentials read-only into the container instead of copying them in:
     -v "$HOME/.claude:/home/agent/.claude:ro"
   Confirm in the run script that auth works inside the container before the loop starts.

3. COMPOSE (compose.loop.yml) if {{SERVICES}} != none. One service per dependency (postgres, redis, etc.) on an internal network, plus the "runner" service built from Dockerfile.loop. The runner depends_on the services and waits for them to be healthy. Mount the repo into the runner at /work.

4. SANDBOX THE PERMISSIONS, NOT THE HOST. Inside the container the loop runs claude with:
     claude -p "<iteration prompt>" --dangerously-skip-permissions --add-dir /work
   This is ONLY acceptable because we're in a throwaway container with no host mounts except the repo and read-only creds. Call this out in a comment so future-me doesn't copy that flag onto a host.

5. RUN SCRIPT (run-loop.sh, chmod +x). It should:
     - build the image
     - bring up services (docker compose up -d) if any
     - run the loop inside the runner: iterate the agent against {{GOAL}}, running {{CHECK_CMD}} each pass, stopping on exit 0 or max iterations (reuse the loop logic from the worktree guide, just executed inside the container)
     - on finish, copy the resulting branch/diff OUT to my host so I can review it, then tear the stack down (docker compose down -v)
   The repo is mounted, so commits the agent makes are visible on my host on a loop/* branch — but nothing else it did (installs, db writes, processes) survives teardown.

6. PROVE ISOLATION. After it runs, show me: the diff the loop produced, and proof that nothing leaked to the host — e.g. the container installed packages / wrote db rows that no longer exist after `docker compose down -v`, while the code changes on the loop/* branch persist.

GUARDRAILS
- Read-only mount for credentials (:ro). Never copy keys into a layer.
- Only the repo and creds are mounted from the host. No bind-mount of $HOME, no docker socket, no host network.
- --dangerously-skip-permissions lives INSIDE the container only. Never in a host run script.

Output: the files you created, the one command to start a sandboxed loop, and where the reviewable result lands on my host.
````

## the mental model

```
your machine
└── docker
    └── runner container         <- agent runs here with permissions OFF
        ├── /work  (your repo, mounted — commits survive)
        ├── ~/.claude (creds, read-only — auth works, can't be stolen)
        ├── node_modules, installs, processes  <- all die on teardown
        └── postgres / redis (compose services) <- all die on teardown
```

the agent thinks it has a whole machine. it does — a disposable one. you keep the only two things that matter: the code it wrote, and your real computer.

## why this is the unlock

without the box, "run an agent unattended with full permissions" is a sentence that should scare you — one bad command and you're restoring from backup. with the box, it's boring: it ran, it broke things, you threw the box away, you kept the diff. boring is exactly what you want from something that runs while you're asleep. the container is what turns "i'd never let an agent loop unsupervised" into "of course i do, it's in a container."
