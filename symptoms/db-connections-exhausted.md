# The database is refusing or queueing connections

**DON'T:** raise `max_connections`, restart the database, or kill connections at random.
Raising the limit moves the wall a few metres further away. A restart rolls back every in-flight transaction. Killing blind can kill the session holding the lock that would have explained everything.

**First:** are connections exhausted, or is one query blocking all the others? A dashboard shows both as "connections at 100%".

1. `psql -h DB_HOST -c "select state, count(*) from pg_stat_activity where backend_type='client backend' group by 1 order by 2 desc;"` · `[mysql]` `SHOW FULL PROCESSLIST;`
   → many `idle in transaction` → a leaked transaction: code opened one and never committed, go to 3 · many `active` → real work, go to 2 · one large unlabelled bucket with a blank state → you are not superuser, so every session but your own reads as NULL; `GRANT pg_monitor TO YOUR_USER` and re-run, because every step below is blind until you do
2. `psql -h DB_HOST -c "select pid, now()-query_start as age, left(query,80) from pg_stat_activity where state='active' and pid <> pg_backend_pid() order by age desc limit 5;"`
   → one query far older than the rest → that is the cause, not a symptom · all young → 4
3. `psql -h DB_HOST -c "select pid, pg_blocking_pids(pid), left(query,60) from pg_stat_activity where wait_event_type='Lock' and cardinality(pg_blocking_pids(pid)) > 0;"` — the `wait_event_type` filter is not cosmetic; `pg_blocking_pids` is expensive and the docs say not to run it across every row
   → a chain → only the head of the chain matters; everything behind it is collateral · empty → nothing is blocked, so it is volume, go to 4
4. `psql -h DB_HOST -c "select client_addr, count(*) from pg_stat_activity group by 1 order by 2 desc;"`
   → one host holds most of them → a deploy that leaks connections or a runaway job → `deploy-made-it-worse.md` · evenly spread → 5
5. `[pgbouncer]` `psql -h PGBOUNCER_HOST -p 6432 -U ADMIN_USER pgbouncer -c "show pools;"` — pgbouncer usually runs on the app side, so this is rarely `DB_HOST`
   → `maxwait` above zero → clients are queueing right now, and that number is how long the oldest one has waited · `cl_waiting` high while `sv_active` sits at the pool limit → the app wants more than the pool allows; the database itself is idle and blameless · permission denied → your user is not in pgbouncer's `admin_users`
6. `psql -h DB_HOST -c "select (select count(*) from pg_stat_activity) as current, name, setting from pg_settings where name in ('max_connections','superuser_reserved_connections');"`
   → `current` well under the limit → this is not a connection problem at all → `slow-everything.md` · `current` at or near it → the wall is real, and nothing you do below it helps until you find what holds the connections

**Nothing matched?** Open an issue with the `state, count(*)` table from step 1.

*Commands written against PostgreSQL 16 and pgbouncer 1.22 syntax; no database available on the authoring machine, so no step is runtime-verified — see the [run matrix](../README.md#where-each-file-has-been-run). Steps 1, 2, 3, 5 and 6 were revised after reading each command against its documented output; reading is not running and none of them count as verified.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
