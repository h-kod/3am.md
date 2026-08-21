# Something was killed for using too much memory

**DON'T:** raise the memory limit and redeploy, restart in a loop, or add swap.
Raising the limit hides a leak until it takes down something bigger. A restart loop destroys the one log line that names the victim and its size. Swap converts a fast crash into a slow, unkillable crawl.

**First:** was it the kernel OOM killer, a container limit, or the application's own heap? Only the first one appears in `dmesg`.

1. `sudo dmesg -T | grep -iE 'killed process|out of memory|oom' | tail -5` `[linux]`
   → a hit → the kernel picked a victim, and the line gives you the process name and its RSS at death; that number is the whole answer · nothing → 2
2. `kubectl get pod POD -o jsonpath='{range .status.containerStatuses[*]}{.name}{" "}{.lastState.terminated.reason}{"\n"}{end}'` `[k8s]`
   → `OOMKilled` → the container limit killed it, not the host; the host may have had plenty free · `Error` or empty → it exited on its own, so read the app log
3. `kubectl get pods` and read `RESTARTS` `[k8s]` · `uptime` · `ps -o pid,lstart,comm -p PID`
   → restarts at a regular interval → a leak, and the interval tells you how fast · one restart under load → a spike, which is a different fix
4. `ps -eo pid,rss,comm --sort=-rss | head -10` `[linux]` · `ps -Ao pid,rss,comm -r | head -10` `[macos]`
   → one process dwarfs the rest → that is your candidate, note its PID for step 5 · many equal processes → too many workers for the box, not a leak
5. `for i in 1 2 3 4; do ps -o rss= -p PID; sleep 20; done`
   → monotonically rising with steady load → a leak · flat → it is sized wrong, not leaking
6. `vmstat 1 5` and read `si`/`so` `[linux]` · `vm_stat | grep -E 'Swapins|Swapouts|Pageouts'` twice, 10s apart `[macos]`
   → swap activity climbing → the box is thrashing and everything on it is now slow → `slow-everything.md` · no swap activity → it died cleanly at a hard limit

**Nothing matched?** Open an issue with the `dmesg` line or the `lastState.terminated` value.

*Verified: macOS 26.5, system tools · 2026-08. Steps 3-6 run as written in their macOS form. `dmesg` requires root on macOS and the OOM killer is Linux-only, so step 1 is `[linux]` by nature and not runtime-verified here.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
