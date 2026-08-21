# Is it us, or is it our provider?

**DON'T:** post "we are investigating" before you know the blast radius, and do not fail over to another region as your first move.
A failover during a provider incident often moves you *into* the broken region. A premature status post you have to walk back costs more trust than five minutes of silence.

**First:** does it fail from a network you do not control? If you cannot answer that, you cannot answer anything else.

1. Bypass your own resolver and use a public one's answer: `IP=$(dig +short A YOUR.SITE @1.1.1.1 | head -1); curl -sS -o /dev/null --max-time 10 -w "code=%{http_code} ttfb=%{time_starttransfer}\n" --resolve YOUR.SITE:443:$IP https://YOUR.SITE`
   → fine from outside, broken from your desk → it is your side → `dns-suspect.md` · broken from both → 2
2. `curl -sS --max-time 8 https://status.PROVIDER.com/api/v2/status.json | jq -r '.status.indicator + " | " + .status.description'`
   → anything other than `none` → they know; read `.../api/v2/incidents/unresolved.json` for the affected components · `none` → 3
3. Read the provider's status *API*, never the HTML page. The page is cached at the edge and lies during exactly the incidents you care about.
   → API says healthy but you disagree → they may not have noticed yet; go to 4 and build the evidence you will send them
4. Test one thing you own end to end that does not touch the provider at all — a static asset from a different host, a health endpoint with no dependencies.
   → that also fails → it is you, go back to the router · that works → the failure is confined to the provider path, go to 5
5. Pin down the exact boundary: which call, to which endpoint, with which error, from which region. Get one reproducible command.
   → you have it → this is what the provider's support needs; anything less gets you a templated reply
6. Capture the first failing timestamp from your own logs before they rotate, in UTC.
   → done → you can now correlate with their post-incident timeline, which is the only way you ever get a credit

**Nothing matched?** Open an issue with the provider and the status API response you saw.

*Verified: macOS 26.5, system tools · 2026-08. Steps 1-2 run as written against a live Statuspage API (`.../api/v2/status.json` is the Atlassian Statuspage standard path). Note: `curl --dns-servers` is deliberately not used — macOS system curl is built without c-ares and rejects the flag outright, so step 1 resolves with `dig` and pins the answer with `--resolve` instead.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
