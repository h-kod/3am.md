# It broke right after a deploy

**DON'T:** roll back before you capture what shipped, and never roll back a migration reflexively.
A rollback without the diff means you ship the same bug tomorrow, having learned nothing. Rolling back a schema change can destroy data the new code already wrote in the minutes it was live.

**First:** did *code* ship, did *config* ship, or did a dependency move underneath you? All three look like "the deploy broke it".

1. `git log --oneline PREV_SHA..CURRENT_SHA` and `git diff --stat PREV_SHA..CURRENT_SHA`
   → commits you recognise → read them, go to 4 · empty → no code shipped; it is config or a dependency, go to 2
2. `git diff PREV_SHA..CURRENT_SHA -- '*lock*' '*.lock' go.sum | head -40`
   → a lockfile moved → a transitive dependency changed without anyone deciding to change it; this is the most under-suspected cause on this page
3. `kubectl rollout history deployment/NAME` then `kubectl rollout history deployment/NAME --revision=N` `[k8s]`
   → env vars, limits, or an image tag changed → config shipped, not code; the git diff was always going to be empty
4. `git diff --name-only PREV_SHA..CURRENT_SHA | grep -iE 'migrat|schema|alembic|flyway|liquibase'`
   → a migration is in the diff → **a rollback is not safe by default.** Decide explicitly, out loud, whether the old code can read the new schema · nothing → a rollback is probably safe
5. `kubectl get pods -o wide` and check how many pods run each image `[k8s]`
   → both versions serving → half your traffic is on each, which is why the failure looks random → `intermittent-5xx.md`
6. Compare the first error timestamp with the deploy timestamp, in UTC, to the minute.
   → more than a few minutes apart → the deploy may be a coincidence and you are about to roll back the wrong thing; go back to the router · same minute → you have your cause

**Nothing matched?** Open an issue with the `git diff --stat` output and the two timestamps.

*Verified: macOS 26.5, git 2.x · 2026-08. Steps 1, 2, 4 and 6 run as written. `kubectl rollout` paths need a live cluster and are not runtime-verified here.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
