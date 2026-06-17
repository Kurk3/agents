# 03 · playwright loop

> close the loop with a real browser. the check isn't "tests pass" — it's "the page actually works." the agent drives chromium, reads what broke (and a screenshot), fixes, reloads, repeats until the flow goes green.

- **run mode:** `loop` with a browser as the check
- **how to use:** paste the prompt below into your agent → it wires playwright as the loop's verification step in your web app, then runs the loop once against a real flow. fill the `{{PLACEHOLDERS}}` first.

## why a browser closes the loop properly

unit tests prove your functions are right. they don't prove the button is clickable, the form submits, the data shows up, the page doesn't white-screen on load. for a web app, the honest definition of "done" is *a user can do the thing* — and the only way an agent can check that unattended is to actually drive the browser.

playwright gives the loop eyes. the agent navigates the app, asserts the flow, and when it fails it gets back the exact error plus a screenshot of the broken state — real feedback, the same thing you'd see if you opened devtools yourself. that's a far better signal than a green unit test that lies about whether the app works.

two ways to wire it, both in the prompt:
- **playwright test as the check** — a `*.spec.ts` that is the loop's exit-0 command. simplest, runs headless, great for cron.
- **playwright MCP, agent-driven** — the agent itself clicks around live via the MCP server, decides what's broken, and explores flows you didn't script. better when the goal is fuzzy ("make onboarding not broken").

## the prompt — copy this

````text
You are wiring Playwright as the verification step of an autonomous loop on MY web app, so "done" means a real browser can complete the flow — not just that unit tests pass. Build it, then run the loop once against a real flow. Write real files.

CONTEXT
- Repo: {{REPO_PATH}}
- App: how to start it + URL: {{APP_START}}    e.g. "npm run dev, serves http://localhost:5173"
- The user flow that must work (this is the goal): {{FLOW}}    e.g. "land on /, log in with test creds, add an item to cart, see it in /cart"
- Stack: {{STACK}}    e.g. "React + Vite frontend, same-repo Node API"
- Test creds / seed data the flow needs: {{TEST_DATA}}    e.g. "user test@x.com / pw 'test1234', seeded by npm run seed"

DO THIS
1. INSTALL + CONFIG. Add Playwright (npm i -D @playwright/test, npx playwright install chromium). Create playwright.config.ts with a webServer block that boots the app ({{APP_START}}) and waits for the URL before tests run — so the loop is one self-contained command.

2. THE CHECK = THE FLOW. Write e2e/flow.spec.ts that performs {{FLOW}} end to end and asserts the user-visible result (text on screen, URL, an element). On failure, configure Playwright to capture a screenshot + trace (use: 'on-failure'). This spec is the loop's exit-0 check: `npx playwright test` green == goal met. Write it to FAIL today if the flow is currently broken — that's the starting signal.

3. FEED FAILURES BACK WITH EYES. The runner each iteration:
     - runs `npx playwright test`; if green, "DONE" and stop
     - if red, collect the failure: the assertion error, the console/network errors Playwright logged, AND the path to the failure screenshot
     - call the agent headless with all of that, including the screenshot path, so it can SEE the broken state:
         claude -p "Flow `{{FLOW}}` is failing. Playwright error: <error>. Console errors: <errors>. Screenshot at <path> — open it. Make the smallest fix so the flow passes. Don't edit the spec." \
           --permission-mode acceptEdits --allowedTools "Bash" Read Write Edit
     - re-run, loop, stop on green or max iterations
   Reuse the loop engine from guide 01; this just swaps the check for the Playwright run and adds the screenshot to the feedback.

4. OPTIONAL — AGENT-DRIVEN VIA MCP. Also set up the Playwright MCP server (claude mcp add playwright) and write a second prompt where the agent clicks through the live app itself to find what's broken, instead of running a fixed spec. Use this mode when the goal is fuzzy; use the spec mode for cron. Document both, tell me which to use when.

5. PROVE IT. Start the loop against {{FLOW}} now. Show me: the spec, the first failure (with the screenshot it saw), the fix it made, and the run going green. Then break the flow on purpose and show it recover.

GUARDRAILS
- The agent must never edit e2e/flow.spec.ts — it could "pass" by weakening the assertion. The spec is read-only to the loop; it defines truth.
- Assert user-visible outcomes (text, URL, visible element), not implementation details, so the check reflects "the user can do it," not "the DOM has a class."
- Keep the flow deterministic: seed/reset test data each run ({{TEST_DATA}}) so a failure means the code broke, not that yesterday's data is stale.

Output: the files you created, the one command to run the loop, and how to switch between spec-mode and MCP-mode.
````

## pairs with the sandbox

run this loop inside the [docker sandbox](02-docker-sandbox.md): the container boots the app, runs chromium headless, drives the flow, and tears down — so a browser process or a polluted db never sticks around on your host. compose service for the app + the runner with playwright installed, and you've got a self-contained "does the app actually work" loop you can fire on a schedule.

## why eyes beat green checkmarks

i shipped plenty of code with all unit tests green and a homepage that white-screened on load, because nothing in the suite ever opened the page. the browser is the only check that can't be fooled by code that's "correct" but doesn't work. once the loop's definition of done is "a real chromium completed the real flow," the agent can no longer lie to itself — and neither can you. that's the point of giving it eyes.
