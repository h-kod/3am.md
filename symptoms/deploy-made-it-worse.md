# It broke right after a deploy

**DON'T:** roll back before you capture what shipped, and never roll back a migration reflexively.
A rollback without the diff means you ship the same bug tomorrow, having learned nothing. Rolling back a schema change can destroy data the new code already wrote in the minutes it was live.

**First:** did *code* ship, did *config* ship, or did a dependency move underneath you? All three look like "the deploy broke it".

1. `git log --oneline PREV_SHA..CURRENT_SHA` and `git diff --stat PREV_SHA..CURRENT_SHA`
   → commits you recognise → read them, go to 4 · empty → no code shipped; it is config or a dependency, go to 2
2. `git diff PREV_SHA..CURRENT_SHA -- '*lock*' '*.lock' go.sum | head -40`
   → a lockfile moved → a transitive dependency changed without anyone deciding to change it; this is the most under-suspected cause on this page
3. `kubectl rollout history deployment/NAME` then the same with `--revision=N` and `--revision=N-1`, and compare the two templates by eye `[k8s]`
   → env vars, limits, or an image tag differ → config shipped, not code; the git diff was always going to be empty · `CHANGE-CAUSE` is `<none>` on every row → normal, nobody sets that annotation; the answer is in the `--revision` templates, not this table · the revision you want is missing → history stops at `revisionHistoryLimit`, default 10, and resets if the Deployment was recreated
4. `git diff --name-only PREV_SHA..CURRENT_SHA | grep -iE 'migrat|schema|alembic|flyway|liquibase'`
   → a migration is in the diff → **a rollback is not safe by default.** Decide explicitly, out loud, whether the old code can read the new schema · nothing → a rollback is probably safe
5. `kubectl get pods -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[*].image' | sort -k2` `[k8s]` — `-o wide` has no image column, it adds IP and node
   → two image tags serving at once → half your traffic is on each, which is why the failure looks random → `intermittent-5xx.md` · one tag → the rollout finished, so a half-deployed state is not your explanation, go to 6
6. Compare the first error timestamp with the deploy timestamp, in UTC, to the minute.
   → more than a few minutes apart → the deploy may be a coincidence and you are about to roll back the wrong thing; go back to the router · same minute → you have your cause

**Nothing matched?** Open an issue with the `git diff --stat` output and the two timestamps.

*Verified: macOS 26.5, git 2.x · 2026-08. Steps 1, 2, 4 and 6 run as written. Steps 3 and 5 need a live cluster and are not runtime-verified here — see the [run matrix](../README.md#where-each-file-has-been-run). Step 5 previously read `kubectl get pods -o wide`, which cannot answer the question it was asked: `-o wide` adds IP, node and readiness gates, never the image.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
