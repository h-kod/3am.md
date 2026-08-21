# DNS is giving the wrong answer

**DON'T:** change a record, lower the TTL, or flush caches everywhere.
A record change during an incident propagates for as long as the *old* TTL, so you lose the ability to tell what resolvers actually see — and you cannot undo it faster than that TTL either.

**First:** is the record wrong, or is one resolver wrong? Completely different problems.

1. Ask the authoritative server and a public resolver, and compare: `dig +short A YOUR.NAME @$(dig +short NS YOUR.DOMAIN | head -1)` then `dig +short A YOUR.NAME @1.1.1.1` and `@8.8.8.8`
   → authoritative correct, public stale → it is TTL; you wait it out, go to 2 · authoritative wrong → the record itself is wrong · public resolvers disagree with each other → mid-propagation
2. `dig +noall +answer YOUR.NAME`
   → the number in the second column is the remaining TTL in seconds; that is how long you are stuck, and no amount of flushing changes it for other people
3. `dig YOUR.NAME | grep -E 'status:|ANSWER:'`
   → `status: NXDOMAIN` → the name does not exist at all · `status: NOERROR` with `ANSWER: 0` → the name exists but not for this record type, e.g. you asked A and only AAAA exists · `status: SERVFAIL` → go to 5
4. `scutil --dns | grep -m3 nameserver` `[macos]` · `resolvectl status` `[linux]`, then read `/etc/hosts`
   → a resolver you did not expect, or a stale `/etc/hosts` line → it is only broken for you; a VPN or a corporate proxy is answering
5. `dig +dnssec YOUR.NAME @1.1.1.1` and the same against a non-validating resolver.
   → SERVFAIL only from validating resolvers → DNSSEC is failing, usually after a key rollover; this breaks for roughly half the internet and looks random
6. `dig NS YOUR.DOMAIN @1.1.1.1` and compare with what your registrar shows.
   → they differ → the delegation is wrong at the registrar, which no amount of record editing at your DNS provider will fix

**Nothing matched?** Open an issue with the `dig` output from steps 1 and 3.

*Verified: macOS 26.5, system tools · 2026-08. Steps 1-4 and 6 run as written. Note: `dig +trace` was deliberately left out — it queries the root servers directly and is blocked on many corporate and NAT networks, so it fails in exactly the situation you would reach for it.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
