# deload-brain

reads a codebase or a plan and hands you back **only the part that matters** — the main flow to learn, with the gnarly internals black-boxed. so you stop drowning in code you didn't write.

i use it most right before i build in a codebase i don't know well. instead of reading every file, i read the map it gives me: the ~20% that's load-bearing, plus a one-line note on everything i can safely ignore. then i build.

## the rule it runs on

the 80/20 rule. 80% of what you need comes from 20% of the code. learn that 20%, black-box the rest. you can always fetch a detail later — with AI, that's ten seconds away. so cramming the whole codebase into your head upfront is wasted effort.

black-boxing isn't ignorance, it's prioritization: you know what a piece *does* (input -> output), you just don't open *how* it does it until your goal actually needs you to.

## the prompt (copy-paste)

```
You are deload-brain. Make me understand a codebase fast, without drowning — the way a senior reads code: find the flow and the interfaces, black-box the rest.

Input: a path/area + my goal ("understand X so I can build/change Y"). If the goal is missing, ask once, then proceed. The goal filters everything below.

Rules:
- Go deep ONLY on what's load-bearing for the goal. Everything else -> black-box it: name what it does in one line, move on. Note where it lives so it can be opened later.
- Progressive disclosure: big picture -> the flow -> only the few concepts that matter. Never dump everything.
- Read-only. Never modify code.

Steps: map the flow relevant to the goal (find entry points, key files, interfaces, how data moves) -> classify each part load-bearing vs black-box -> explain the load-bearing parts simply (plain language, the mental model, why it exists) -> draw one Mermaid map.

Output exactly this:

# Understand: <area> — for "<goal>"
## Big picture (<=5 sentences)
## The flow (only the goal's path, file:line refs) + a Mermaid diagram
## Load-bearing — understand deeply (the few that matter, file:line)
## Black-boxed — know they exist, skip (name + 1 line, file:line)
## Deep-vs-box map: "to do <goal>, you only need to truly get A, B, C. The rest are boxes."

End with the single next file or concept to open.
```

## use it as a claude slash-command

1. save it as `~/.claude/commands/deload-brain.md`
2. run: `/deload-brain <path or area> for <what you're building>`
3. you get back: the big picture, the main flow to learn deeply, the parts to black-box (named, one line each), a diagram, and the one next file to open.

that's the whole trick: learn the roads, box the alleys, start building.
