# Some requests fail, some don't

**DON'T:** turn up retries, scale the replica count, or restart the instance that is erroring.
Retries turn a partial outage into a full one — that is what a retry storm is. Restarting the erroring instance deletes the only stack trace that names the cause.

**First:** what fraction fails? A small constant fraction is one bad node or one bad key. A growing fraction is capacity or a dependency.

1. `for i in $(seq 20); do curl -sS -o /dev/null --max-time 10 -w '%{http_code} %{remote_ip}\n' https://YOUR.SITE; done | sort | uniq -c | sort -rn`
   → failures all on one `remote_ip` → one bad node, go to 4 · spread across IPs → a shared dependency, go to 3 · all `200` → not reproducible from here → `is-it-us.md`
2. Read *which* 5xx you got in step 1.
   → `502`/`504` → upstream never answered: the app is down or slow behind the LB → `slow-everything.md` · `503` → no healthy target, or you are being rate limited, go to 4 · `500` → your code; the trace is in the app log
3. `for p in / /api/health /api/SOME_ENDPOINT; do curl -sS -o /dev/null --max-time 10 -w "$p %{http_code}\n" https://YOUR.SITE$p; done`
   → one path fails → that path's dependency, not the platform · all paths fail intermittently → shared: DB, cache, or auth → `db-connections-exhausted.md`
4. `kubectl get endpointslices -l kubernetes.io/service-name=SERVICE -o wide` `[k8s]` — `get endpoints` truncates the address list and hides not-ready ones · `aws elbv2 describe-target-health --target-group-arn ARN` `[aws]`
   → fewer ready addresses than replicas → the LB is doing its job; find why that target died → `oom-killed.md` or `disk-full.md` · all ready → 5
5. `curl -sS -o /dev/null --max-time 10 -w '%{http_code} %{http_version}\n' --http1.1 https://YOUR.SITE` then the same without `--http1.1`
   → only HTTP/2 fails → ALPN or proxy mismatch at the edge, not your app · both the same → 6
6. Re-run step 1 prefixing each line with a timestamp and look for a period.
   → failures every N seconds → a health check flapping, a cron, or a GC pause · random → capacity or a poison input

**Nothing matched?** Open an issue with the `uniq -c` table from step 1.

*Verified: macOS 26.5, system tools · 2026-08. Steps 1-3, 5 and 6 run as written. Step 4 needs a live cluster — not runtime-verified here, see the [run matrix](../README.md#where-each-file-has-been-run). It moved from `get endpoints` to `get endpointslices`: the Endpoints API is deprecated as of Kubernetes 1.33, and its `kubectl` view showed only the first few ready addresses, which is the one thing step 4 asks you to count.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
