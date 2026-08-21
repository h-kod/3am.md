# Contributing

The format is the product. A file that reads well but breaks the format makes the repo worse, so format review comes before content review.

Read [SPEC.md](SPEC.md) first. All six rules are hard.

## Adding a symptom

1. Name it after **what a human sees**, not what you suspect. `site-down-but-server-up`, not `nginx-502`.
2. Open with `DON'T:` and say *what each reflex destroys*. "Don't restart" is worthless; "a restart empties the connection table you are about to read" is the whole point of the line.
3. Six steps maximum. If you need seven, two of yours are the same step.
4. Every step gets one `→` routing line, branches separated by ` · `, and every branch ends somewhere: another step, another file, or a sentence naming the culprit.
5. No command may change the system being debugged. Copying evidence out is fine.
6. **Run every command before you open the PR**, and write what you ran in the footer. If you could not run one, say so in the footer. An honest gap is fine; a fake "Verified" is not.

## Fixing a command

Open a `command failed` issue with the exact command, the exact output, and your OS. Platform differences are the most common cause and the most valuable fix — `ss` and `timeout` do not exist on macOS, `nc` uses `-G` there and `-w` on Linux, and `curl --dns-servers` is missing from macOS system curl entirely.

## Verification debt

Known and deliberate, recorded here so no footer overstates itself:

- **Linux runtime** — every `[linux]` branch is written from documentation, not run.
- **PostgreSQL and pgbouncer** — `data-looks-wrong` and `db-connections-exhausted` have no runtime-verified step.
- **Brokers** — `queue-backlog` steps 1-4 and 6 are unverified for Redis, RabbitMQ and SQS.

Closing any of these is more valuable than a new symptom file.

## What gets rejected

Anything on the list in [NON-GOALS.md](NON-GOALS.md), plus: a seventh step, a paragraph of theory, a step with no exit, and a `Verified` footer for a command you did not run.
