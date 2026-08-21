# Data is missing or wrong

**DON'T:** re-run the job, fix rows by hand, or delete and re-import.
Each one destroys the two facts you can only measure once: how many rows are affected, and since when. You cannot size a blast radius twice.

**First:** is data *missing*, *duplicated*, or *stale*? Three different causes — pick one before you type anything.

1. `psql -c "select date_trunc('hour', created_at) h, count(*) from TABLE group by 1 order by 1 desc limit 12;"`
   → a cliff at one hour → something changed then → `deploy-made-it-worse.md` · a slow decline → a growing backlog → `queue-backlog.md` · counts normal → the rows exist, so it is wrong-not-missing, go to 3
2. Count the same window at the source and at the destination.
   → source has more → it never arrived → `queue-backlog.md` · counts equal → the write worked; the read or the transform is wrong
3. `psql -c "select KEY_COLUMN, count(*) from TABLE where created_at > now() - interval '6 hours' group by 1 having count(*) > 1 limit 10;"`
   → duplicates → a retry without an idempotency key; find the retrying caller before you delete anything
4. `psql -c "select max(updated_at), now() - max(updated_at) as staleness from TABLE;"`
   → frozen at a timestamp → the writer died at that moment → `oom-killed.md` or `queue-backlog.md`
5. `psql -c "select current_database(), current_user, inet_server_addr(), version();"`
   → not the database you assumed → stop; you were looking at the wrong environment. This is the single most common 3am mistake.
6. Freeze the evidence before anyone touches it: post the row counts, the min and max ids, and the timestamps from steps 1 and 4 into the incident channel.
   → done → now you may plan a fix, and you can prove afterwards whether it worked

**Nothing matched?** Open an issue describing which of the three shapes it was.

*Commands written against PostgreSQL 16 syntax; no database available on the authoring machine, so steps 1-5 are NOT runtime-verified.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
