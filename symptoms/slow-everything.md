# It loads, but everything is slow

**DON'T:** scale up, restart, raise timeouts, clear a cache.
Scaling up hides the cause and doubles the bill. Raising a timeout turns a fast failure into a slow queue, which is worse. A restart empties the connection table you are about to read.

**First:** is every endpoint slow, or one? One endpoint is a query or a dependency, never capacity.

1. `curl -sS -o /dev/null --max-time 20 -w 'dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' https://YOUR.SITE`
   → `ttfb` high, rest low → server side, go to 2 · `tls` high → handshake CPU or a long chain → `cert-expired.md` · `dns` high → `dns-suspect.md` · `total` high but `ttfb` low → response body or bandwidth, not logic
2. `for i in $(seq 8); do curl -sS -o /dev/null --max-time 20 -w '%{time_starttransfer} %{remote_ip}\n' https://YOUR.SITE; done`
   → all uniformly slow → systematic: a dependency or the DB, go to 4 · wildly variable → contention or one bad node, go to 3
3. `for ip in ORIGIN_IP_1 ORIGIN_IP_2; do curl -sS -o /dev/null --max-time 20 -w "$ip %{time_starttransfer}\n" --resolve YOUR.SITE:443:$ip https://YOUR.SITE; done`
   → one node much slower → it is that node; removing it from the LB is a write, so decide it out loud · all equal → 4
4. `psql -h DB_HOST -c "select pid, now()-query_start as age, state, left(query,60) from pg_stat_activity where backend_type='client backend' and state <> 'idle' order by age desc limit 5;"` · `[mysql]` `SHOW FULL PROCESSLIST;`
   → ages in seconds → `db-connections-exhausted.md` · nothing running long → 5 · `state` and `query` blank on every row you did not open → you are not superuser and cannot see other sessions; `GRANT pg_monitor` and re-run, because zero rows here is not an answer
5. `curl -sS -o /dev/null --max-time 20 -w 'dep_ttfb=%{time_starttransfer}\n' https://THIRD_PARTY_ENDPOINT`
   → the dependency is slow too → `is-it-us.md` · dependency is fine → 6
6. `vmstat 1 5` — **ignore row 1, it is an average since boot** `[linux]` · `vm_stat 1 5` and `iostat 1 5` `[macos]`
   → `si`/`so` non-zero in rows 2-5 → `oom-killed.md` · high IO wait → `disk-full.md` · CPU pinned with nothing queued → you are simply out of capacity

**Nothing matched?** Open an issue with the exact `curl -w` line you got.

*Verified: macOS 26.5, system tools · 2026-08. Steps 1-3, 5 and the macOS half of 6 run as written. Step 4 and `vmstat` are not runtime-verified here — see the [run matrix](../README.md#where-each-file-has-been-run).*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
