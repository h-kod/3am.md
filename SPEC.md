# SPEC

Six rules. A file that breaks one of them is not merged, however good the content is.
These rules are the product. The commands are just the filling.

### 1. One file = one symptom, never one technology
At 3am nobody thinks *"Postgres"*. They think *"the site won't load"*. File names describe what a human sees, not which component you suspect. There is no `postgres.md` and there never will be.

### 2. Every file opens with `DON'T`
Most of the damage in an incident comes from the first reflex, not from the failure. Restart, deploy, roll back, clear cache — each one destroys evidence you will need sixty seconds later. Name the reflex and say what it destroys.

### 3. `DON'T` block plus at most 6 steps
One screen, no scrolling. Step 7 turns the file into a runbook, and runbooks do not get read at 3am.
*(Originally specced as "15 lines". In practice a step is a command line plus a routing line, so the operational cap is 6 steps — the same budget, countable.)*

### 4. No command changes the system you are debugging
Reading is allowed. Copying evidence *out* — a log to a file, a metric export — is allowed and encouraged, because evidence expires. Everything else is not: nothing restarts, deploys, rolls back, kills, drops, rotates, scales, or clears. That is what makes the repo safe to run before you understand the problem. When the next move is a change to the system, the file says so in words and stops.

### 5. No theory. Every line is an instruction.
Someone reading for background is not in an incident and is not the reader. No "how DNS works", no diagrams, no history.

### 6. Every step has an exit
A step that produces output but does not say where to go next is not a step, it is trivia. Every branch ends in another step number, another file, or a sentence naming the culprit.

---

## Record format

```
# <What the human sees>

**DON'T:** <reflexes> — <what they destroy>
**First:** <one question that halves the search space> → <file if the answer routes away>

1. `<command that changes nothing>`
   → <branch> → <target> · <branch> → <target>
...

**Nothing matched?** Open an issue with the exact output.

*Verified: <os>, <toolchain> · <date>*
```

## Conventions

- **Placeholders are ALL_CAPS**: `YOUR.SITE`, `ORIGIN_IP`, `DB_HOST`. You need to see what to replace without reading.
- **Branches on one line**, separated by ` · `. A table has to be parsed; a line is scanned.
- **Platform tags inline**: `[macos]` `[linux]`. This matters more than it looks — `ss` and `timeout` do not exist on macOS, and `nc` timeout flags differ (`-G` vs `-w`).
- **The footer records where the command was actually run.** "Verified" means someone ran it and read the output, not that it looked right.
