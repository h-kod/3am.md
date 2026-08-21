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

The record lives in the [run matrix](README.md#where-each-file-has-been-run), not in a tracker. It is a table of what has been executed and what has not, and it is the thing to update when you run something.

Two shapes of debt sit in it. **Linux**: every `[linux]` branch was written from documentation on a macOS machine, which means the unverified half serves most of the actual audience. **Infrastructure**: nothing needing PostgreSQL, pgbouncer, Kubernetes or a broker has been run at all.

Every unrun command has at least been read against its documented output, and that pass caught four routing lines that could not fire — `vmstat`'s first row is an average since boot, `kubectl get pods -o wide` has no image column, SQS counts do not refresh on demand, and `nc -w` prints nothing on timeout. Those are fixed. Reading is not running, no footer counts it, and the matrix still shows them unrun.

Turning one row of that table green is worth more than a new symptom file. Run the commands, fix what the output contradicts, update the file footer and the matrix row in the same PR.

## What gets rejected

Anything on the list in [NON-GOALS.md](NON-GOALS.md), plus: a seventh step, a paragraph of theory, a step with no exit, and a `Verified` footer for a command you did not run.
