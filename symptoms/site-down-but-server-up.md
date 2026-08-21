# Nothing loads, but the server is up

**DON'T:** deploy, restart, roll back, clear a cache, run a migration.
A restart wipes the process state and the connection table — the two things that tell you what happened. You get one look at them.

**First:** can you reproduce it right now, from your own machine? If no → `is-it-us.md`

1. `curl -sS -o /dev/null --max-time 10 -w 'dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} code=%{http_code}\n' https://YOUR.SITE`
   → `code=000 dns=0` → 2 · `code=000 tcp=0` → 3 · `code=000 tls=0` → `cert-expired.md` · `code=5xx` → `intermittent-5xx.md` · `ttfb>2` → `slow-everything.md` · `code=200` → 5
2. `dig +short A YOUR.SITE @1.1.1.1` then `dig +short A YOUR.SITE`
   → empty from both → `dns-suspect.md` · different answers → `dns-suspect.md` · same and non-empty → 3
3. `nc -vz -G 5 YOUR.SITE 443` `[macos]` · `nc -vz -w 5 YOUR.SITE 443` `[linux]`
   → `Connection refused` → nothing is listening: LB target group / process is gone · timeout → packets dropped: firewall, security group, WAF · succeeded → 4
4. `curl -sS -o /dev/null --max-time 10 -w 'code=%{http_code} ip=%{remote_ip}\n' --resolve YOUR.SITE:443:ORIGIN_IP https://YOUR.SITE`
   → origin answers → the edge is the problem, not your app: CDN, LB, WAF · origin also fails → your app or its dependencies
5. Ask one person on a different network (phone hotspot, another region) to load it.
   → works for them → it is you: your DNS, VPN, `/etc/hosts`, or corporate proxy · fails for them → it is global, go to 6
6. `git log --oneline --since='2 hours ago' --all` and read your deploy log without touching it.
   → something shipped → `deploy-made-it-worse.md` · nothing shipped → suspect a dependency, a certificate, or a quota, not your code

**Nothing matched?** Open an issue with the exact output you got. That is how this file gets better.

*Verified: macOS 26.5, system tools only (no Homebrew) · 2026-08. All six steps run as written. Linux syntax is portable but not yet runtime-verified.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
