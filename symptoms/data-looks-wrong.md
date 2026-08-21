# Data is missing or wrong

**DON'T:** re-run the job, fix rows by hand, or delete and re-import.
Each one destroys the two facts you can only measure once: how many rows are affected, and since when. You cannot size a blast radius twice.

**First:** is data *missing*, *duplicated*, or *stale*? Three different causes — pick one before you type anything.

1. `psql -h DB_HOST -c "select date_trunc('hour', created_at) h, count(*) from YOUR_TABLE group by 1 order by 1 desc limit 12;"`
   → a cliff at one hour → something changed then → `deploy-made-it-worse.md` · a slow decline → a growing backlog → `queue-backlog.md` · counts normal → the rows exist, so it is wrong-not-missing, go to 3
2. Count the same window at the source and at the destination.
   → source has more → it never arrived → `queue-backlog.md` · counts equal → the write worked; the read or the transform is wrong
3. `psql -h DB_HOST -c "select KEY_COLUMN, count(*) from YOUR_TABLE where created_at > now() - interval '6 hours' group by 1 having count(*) > 1 limit 10;"`
   → rows come back → duplicates: a retry without an idempotency key; find the retrying caller before you delete anything · no rows → it is not duplication, go to 4
4. `psql -h DB_HOST -c "select max(updated_at), now() - max(updated_at) as staleness from YOUR_TABLE;"`
   → frozen at a timestamp → the writer died at that moment → `oom-killed.md` or `queue-backlog.md` · staleness of seconds → the writer is alive, so this is wrong data rather than missing data, go to 5
5. `psql -h DB_HOST -c "select current_database(), current_user, coalesce(inet_server_addr()::text,'unix socket') as host, inet_server_port() as port, left(version(),40);"`
   → not the database you assumed → stop; you were looking at the wrong environment. This is the single most common 3am mistake · it is the right one → 6
6. Freeze the evidence before anyone touches it: post the row counts, the min and max ids, and the timestamps from steps 1 and 4 into the incident channel.
   → done → now you may plan a fix, and you can prove afterwards whether it worked

**Nothing matched?** Open an issue describing which of the three shapes it was.

*Commands written against PostgreSQL 16 syntax; no database available on the authoring machine, so steps 1-5 are NOT runtime-verified — see the [run matrix](../README.md#where-each-file-has-been-run). Steps 3, 4 and 5 previously had no exit for the ordinary case, which broke rule 6; that is fixed here, and `TABLE` became `YOUR_TABLE` because `TABLE` is a reserved word and a literal copy-paste failed at the parser rather than at the placeholder.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
