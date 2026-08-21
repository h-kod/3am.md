<h1 align="center">3am.md</h1>

<p align="center">
  <b>It's 3am. Prod is down. You can't think.</b><br>
  Open the file that matches what you are seeing. Run the commands in order.
</p>

<p align="center">
  <img alt="13 symptoms" src="https://img.shields.io/badge/symptoms-13-informational">
  <img alt="6 steps max" src="https://img.shields.io/badge/steps%20per%20file-6%20max-informational">
  <img alt="changes nothing" src="https://img.shields.io/badge/changes%20nothing-guaranteed-success">
  <img alt="licence" src="https://img.shields.io/badge/prose-CC%20BY%204.0-lightgrey">
</p>

---

## What are you seeing?

| What you see | Open |
|---|---|
| Nothing loads, but the server is up | **[site-down-but-server-up](symptoms/site-down-but-server-up.md)** |
| It loads, but everything is slow | **[slow-everything](symptoms/slow-everything.md)** |
| Some requests fail, some don't | **[intermittent-5xx](symptoms/intermittent-5xx.md)** |
| Data is missing or wrong | **[data-looks-wrong](symptoms/data-looks-wrong.md)** |
| Jobs or messages aren't being processed | **[queue-backlog](symptoms/queue-backlog.md)** |
| Is it us, or is it our provider? | **[is-it-us](symptoms/is-it-us.md)** |
| It fixed itself | **[it-fixed-itself](symptoms/it-fixed-itself.md)** |

Reached from inside the files above, never directly: [dns-suspect](symptoms/dns-suspect.md) · [cert-expired](symptoms/cert-expired.md) · [db-connections-exhausted](symptoms/db-connections-exhausted.md) · [disk-full](symptoms/disk-full.md) · [oom-killed](symptoms/oom-killed.md) · [deploy-made-it-worse](symptoms/deploy-made-it-worse.md)

---

## What a file looks like

One screen. No theory. Abridged from [site-down-but-server-up.md](symptoms/site-down-but-server-up.md):

```
# Nothing loads, but the server is up

DON'T: deploy, restart, roll back, clear a cache, run a migration.
A restart wipes the process state and the connection table — the two things
that tell you what happened. You get one look at them.

First: can you reproduce it right now, from your own machine? If no → is-it-us.md

1. curl -sS -o /dev/null --max-time 10 -w 'dns=%{time_namelookup}
   tcp=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer}
   code=%{http_code}\n' https://YOUR.SITE
   → code=000 dns=0 → 2 · code=000 tcp=0 → 3 · code=000 tls=0 → cert-expired.md
   · code=5xx → intermittent-5xx.md · ttfb>2 → slow-everything.md · code=200 → 5

2. dig +short A YOUR.SITE @1.1.1.1  then  dig +short A YOUR.SITE
   → both empty → dns-suspect.md · answers differ → dns-suspect.md · same → 3

3. nc -vz -G 5 YOUR.SITE 443 [macos] · nc -vz -w 5 YOUR.SITE 443; echo "exit=$?" [linux]
   → refused → nothing is listening: LB target group / the process is gone
   · [macos] timed out, or [linux] no output and exit=1 → packets dropped:
   firewall, security group, WAF · succeeded → 4
   ...
```

That branch table in step 1 is not guesswork. Each of the three failure modes was triggered on purpose and the output read: DNS failure gives `dns=0.000000`, a refused connection gives `dns>0 tcp=0`, a broken handshake gives `dns>0 tcp>0 tls=0`. All three return `code=000`, so the status code alone tells you nothing — the timing fields are what route you.

---

## Two promises

**No command here changes the system you are debugging.**
Nothing restarts, deploys, rolls back, drops a table, kills a process, scales, or clears a cache. Reading is allowed; so is copying evidence out before it expires. That is the whole promise — you can run every line on this repo while you are still deciding what happened.

**Every file starts with what *not* to do.**
Most of the damage in an incident comes from the first reflex, not from the failure. The reflex destroys the evidence you need sixty seconds later. So each file names the reflex, and says what it destroys.

---

## How the commands are verified

Each file's footer records which of its steps were executed and on what — and where a step could not be run, it says so plainly instead of implying otherwise. The table further down is that record collected in one place, and it is not all green.

That is not ceremony. Two commands in the first draft were wrong in ways only running them reveals:

- **`curl --dns-servers 1.1.1.1`** — a standard suggestion, and macOS system curl *rejects the flag outright*: it is built without c-ares. The file now resolves with `dig` and pins the answer with `--resolve`.
- **`lsof +L1`** for deleted-but-open files — correct, and it returned **570 harmless matches on a healthy machine** next to the one that mattered, a 101 MB browser temp file. At 3am, 570 false positives is the same as no answer. The step now filters by size.

Known gaps are written down rather than papered over. Every unrun command has at least been read against its documented output, which caught four routing lines that could not fire: `vmstat`'s first row is an average since boot, so a swap branch tripped on healthy machines; `kubectl get pods -o wide` has no image column and was asked to count images; SQS message counts do not refresh on demand, so "measure twice, ten seconds apart" read a filling queue as flat; and `nc -w` prints nothing on timeout, so that branch had no output to match. Those are fixed. Reading is not running, and the table below still shows them unrun.

## Where each file has been run

The honest version of the badge at the top. A row is only as good as its **Not run** column.

| File | Run as written | Not run |
|---|---|---|
| [cert-expired](symptoms/cert-expired.md) | macOS 26.5, LibreSSL and Homebrew OpenSSL — all 6 | — |
| [data-looks-wrong](symptoms/data-looks-wrong.md) | — | steps 1-5 · PostgreSQL 16 |
| [db-connections-exhausted](symptoms/db-connections-exhausted.md) | — | all 6 · PostgreSQL 16, pgbouncer 1.22 |
| [deploy-made-it-worse](symptoms/deploy-made-it-worse.md) | macOS, git 2.x — steps 1, 2, 4, 6 | steps 3, 5 · Kubernetes |
| [disk-full](symptoms/disk-full.md) | macOS — steps 1-5 | step 2 on GNU and busybox `du` · `journalctl` in 6 |
| [dns-suspect](symptoms/dns-suspect.md) | macOS — steps 1-4, 6 | `[linux]` half of 4 · step 5 (DNSSEC) |
| [intermittent-5xx](symptoms/intermittent-5xx.md) | macOS — steps 1-3, 5, 6 | step 4 · Kubernetes, AWS |
| [is-it-us](symptoms/is-it-us.md) | macOS — steps 1-2, against a live Statuspage API | steps 3-6 · the footer does not say either way |
| [it-fixed-itself](symptoms/it-fixed-itself.md) | macOS — `crontab -l`, `ps -o lstart`, `uptime` | `journalctl` in 2 · `kubectl` in 2, 4, 6 |
| [oom-killed](symptoms/oom-killed.md) | macOS — steps 3-5, macOS half of 6 | step 1 · Linux · step 2 · Kubernetes · `vmstat` in 6 |
| [queue-backlog](symptoms/queue-backlog.md) | — | steps 1-4, 6 · Redis, RabbitMQ, SQS |
| [site-down-but-server-up](symptoms/site-down-but-server-up.md) | macOS 26.5, system tools only — all 6 | `[linux]` half of 3 |
| [slow-everything](symptoms/slow-everything.md) | macOS — steps 1-3, 5, macOS half of 6 | step 4 · PostgreSQL · `vmstat` in 6 |

Two things that table says out loud. **The repo was written on macOS and almost all production is Linux**, so the verified half serves the smaller audience. And **`is-it-us` steps 3-6 have no record at all** — a gap in the bookkeeping rather than in the testing, and the first thing worth closing because it costs nothing.

Turning a row green is the most useful contribution here — [CONTRIBUTING.md](CONTRIBUTING.md#verification-debt).

---

## The six rules

The format is the product. A file that reads beautifully but breaks a rule makes the repo worse — [SPEC.md](SPEC.md).

1. **One file = one symptom, never one technology.** Nobody thinks *"Postgres"* at 3am.
2. **Every file opens with `DON'T`,** and says what each reflex destroys.
3. **Six steps maximum.** Step seven turns it into a runbook, and runbooks do not get read at 3am.
4. **Nothing changes the system being debugged.** Copying evidence out is the one exception.
5. **No theory.** Someone reading for background is not in an incident and is not the reader.
6. **Every step has an exit** — another step, another file, or a sentence naming the culprit.

---

## When it's over

The acute phase ends before the incident does. **[AFTER-THE-FIRE.md](AFTER-THE-FIRE.md)** is the twenty minutes that decide whether any of it was worth something: the evidence that is on a clock, and the one line of a postmortem nobody ever writes — what you *nearly* did, and chose not to.

## What this is not

Not your runbook. Not a monitoring guide. Not a list of links. Not exhaustive, on purpose. It is the first five minutes, when you cannot think — [NON-GOALS.md](NON-GOALS.md).

## Contributing

The most valuable contribution is not a new symptom, it is a **command that failed on your platform**. Platform differences are the usual cause and the most useful fix.

One symptom per file, six steps maximum, nothing that changes the system, and every command run before it ships — or a footer that says plainly which ones were not — [CONTRIBUTING.md](CONTRIBUTING.md).

---

<p align="center">
  <sub>v0.3 — 13 symptoms, one exit page · prose <a href="LICENSE">CC BY 4.0</a>, commands <a href="LICENSE-SNIPPETS">MIT</a></sub>
</p>
