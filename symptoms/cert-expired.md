# TLS is failing

**DON'T:** disable certificate verification, point clients at plain HTTP, or renew-and-restart before you know which certificate on which host.
Turning off verification during an incident is how it ships to production permanently. And renewing the wrong certificate wastes the only thing you are short of.

**First:** expired, wrong hostname, or broken chain? Three different fixes, and the browser error message will not tell you which.

1. `echo | openssl s_client -connect YOUR.HOST:443 -servername YOUR.HOST 2>/dev/null | openssl x509 -noout -dates -subject -issuer`
   → `notAfter` in the past → expired · dates valid → 2 · no output at all → nothing completed the handshake, go to 4
2. `echo | openssl s_client -connect YOUR.HOST:443 -servername YOUR.HOST 2>/dev/null | openssl x509 -noout -ext subjectAltName`
   → the name you are requesting is not in the SAN list → the wrong certificate is being served; a CN match alone has not been enough for years
3. `echo | openssl s_client -connect YOUR.HOST:443 -servername YOUR.HOST -showcerts 2>/dev/null | grep -c 'BEGIN CERTIFICATE'`
   → `1` → the intermediate is missing; browsers paper over this and `curl`, Java and Go do not, which is why "it works for me" · `2` or more → the chain is being served
4. Compare with and without SNI: run step 1 again but drop `-servername`.
   → a different certificate comes back → the edge is falling back to a default certificate, so the vhost or the SNI mapping is wrong
5. `for ip in ORIGIN_IP_1 ORIGIN_IP_2; do echo "== $ip"; echo | openssl s_client -connect $ip:443 -servername YOUR.HOST 2>/dev/null | openssl x509 -noout -dates; done`
   → one node has an older certificate → the renewal did not reach every node, which produces intermittent failures → `intermittent-5xx.md`
6. `date -u`
   → your clock is off by more than a few minutes → the certificate is fine and your machine is not; this also explains why only some clients fail

**Nothing matched?** Open an issue with the `-dates -subject -issuer` output.

*Verified: macOS 26.5 with both system LibreSSL (`/usr/bin/openssl`) and Homebrew OpenSSL · 2026-08. All six steps run as written; the output format is identical on both toolchains.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
