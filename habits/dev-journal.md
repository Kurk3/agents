# dev journal

> after every work block, i write down what broke, why it broke, and the one rule that stops it happening again. that's it. it's the highest-return habit in my whole setup and it costs five minutes.

if you take one thing from this folder, take this. tools change every six months. the journal is the thing that turns four years of mistakes into judgment instead of four years of repeating them.

## why it works

a mistake you don't write down, you make again. you fix the bug, the relief erases the lesson, and three weeks later you're debugging the same shape of problem from scratch. writing it converts a painful hour into a one-line rule you own forever.

it also does something quieter: it forces you to find the *root cause*, not the patch. "i added a null check" is a patch. "i assumed the API always returns the field — it doesn't on the empty case, so never trust optional fields from external calls" is a rule that protects you on every future API. you can't write the rule without understanding the concept underneath, so journaling drags you from syntax up to concepts whether you meant to or not.

and when a senior reviews your code or a review catches something — that's free, expensive knowledge someone paid for. write it down the same day, then deliberately apply it on the next problem. that loop is most of how i actually leveled up.

## the ritual (how to actually do it)

- **when:** end of each work block, or right after a review / a hard bug / a meeting where you learned something. don't batch it to friday — the detail's already gone by then.
- **what:** one entry, three lines. what broke (or what i learned), the root cause / concept, the rule for next time. that's the whole thing. keep it brutal and short or you'll stop doing it.
- **the rule must be reusable.** not "fixed the cart bug" — that's a changelog. "state that two components share has to live above both or they desync" — that's a rule you'll use for years.
- **mistakes over wins.** wins feel good and teach little. log what hurt.
- **review it.** skim the last month before you start something in the same area. the whole point is to read it *before* you repeat the mistake, not after.

one file, `DEVJOURNAL.md`, append-only, newest at top. that's the entire system. don't build infrastructure for it.

## the template

```markdown
## YYYY-MM-DD — <short title of the thing>

- **what happened:** <the mistake, the bug, or what someone taught me>
- **root cause / concept:** <the real reason — the thing one level under the symptom>
- **rule for next time:** <the reusable sentence i'll carry forward>
- **study:** <optional — concept i was shaky on and should read up on>
```

## let your agent write it (AI-leverage version)

you won't journal consistently from a blank page — nobody does. so don't. at the end of a session, point your agent at what you actually did and have it draft the entry. you edit and keep what's true. that's the difference between a habit that survives and one that dies in week two.

copy this:

````text
You are my dev-journal keeper. At the end of a work session, turn what I actually did into a sharp, reusable journal entry. No praise, no recap of features shipped — I want mistakes and lessons I can carry forward.

CONTEXT
- Today's diff (what I changed):
{{TODAYS_DIFF}}
- What I was working on / what went wrong today (my own words, messy is fine): {{NOTES}}
- Journal file to append to: {{JOURNAL_PATH}}   default ./DEVJOURNAL.md

DO THIS
1. From the diff and my notes, find the real lessons — the bugs, the wrong assumptions, the things I clearly fixed by trial and error (look for reverts, repeated edits to the same spot, defensive code I added late). Infer the root cause one level under the symptom.
2. For each, write the reusable RULE — a sentence general enough to apply to the next project, not a changelog line about this one. "Never trust an optional field from an external call" — not "added null check to user.email".
3. Flag concepts I looked shaky on (kept guessing, copied without understanding) under "study".
4. Be honest and a little harsh. If I made the same kind of mistake more than once, say so. If today was clean and there's no real lesson, write one line saying that — do not invent filler.

OUTPUT
Append ONE entry to {{JOURNAL_PATH}} (create it if missing, newest entry at the top), using exactly:

## <today's date> — <short title>
- **what happened:** ...
- **root cause / concept:** ...
- **rule for next time:** ...
- **study:** ... (omit if none)

Then print just the entry you added. Don't touch any other file.
````

## wire it as a slash command

1. save the prompt as `~/.claude/commands/journal.md`. prefill the diff with a `!`-line so the agent always has today's changes:
   ```
   ---
   description: turn today's work into a dev-journal entry
   allowed-tools: Bash(git *), Read, Write, Edit
   ---
   Today's diff:
   !`git log --since=midnight -p --no-color 2>/dev/null; git diff HEAD --no-color`

   <paste the prompt body, with {{TODAYS_DIFF}} = the diff above and {{NOTES}} = $ARGUMENTS>
   ```
2. end of day: `/journal "spent two hours on the auth redirect, kept getting infinite loops"`
3. read the entry, fix what's wrong, keep it. five minutes.

the discipline you keep is "review the diff it gives you" — way easier to sustain than "write from scratch." the agent does the lift; you keep the judgment. that's the deal in everything in this repo, and it's the deal here too.
