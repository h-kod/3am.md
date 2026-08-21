# It fixed itself

**DON'T:** close the incident, silence the alert, and go back to bed.
Nothing fixes itself. Something fixed it, and in six hours it will happen again — except by then the logs have rotated, the high-resolution metrics have been rolled up, and the process that held the answer has been replaced. You have minutes, not hours.

**First:** capture, then understand. Every step below is about evidence that is actively expiring.

1. Write down two timestamps in UTC: the first failure and the first success. Take them from the alert or the log, never from memory.
   → you have both → the gap length is itself a clue: seconds means a retry, minutes means a restart, exactly 60 minutes means a TTL or a rate-limit window
2. Copy the logs out before rotation: `journalctl --since '-2h' > ~/incident-LOGS.txt` `[linux]` · `kubectl logs POD --since=2h --previous > ~/incident-LOGS.txt` `[k8s]`
   → `--previous` is the important flag; if the process was replaced, the interesting log is in the dead container, and it is deleted when the pod is
3. Export the metric window now. Most stacks keep per-second or per-minute resolution for hours and then roll it up forever.
   → done → do not rely on being able to zoom into this window tomorrow, because you will not be able to
4. Did something restart at the recovery minute? `kubectl get pods` and read the `RESTARTS` column `[k8s]` · `ps -o pid,lstart,comm -p PID` · `uptime`
   → a restart at that minute → it did not fix itself, something killed and replaced it → `oom-killed.md` · no restart → 5
5. Did recovery land on a round boundary — an exact minute, an exact hour?
   → yes → a scheduled thing: a token refresh, a certificate reload, a rate-limit window resetting, a cache TTL expiring · no → 6
6. `crontab -l` and `kubectl get cronjobs -A` `[k8s]`, then compare their schedules with your failure timestamp.
   → a job overlaps the failure window → your own scheduled work is the cause, and it will run again on schedule · nothing overlaps → write the two timestamps and the captured logs into the incident, and say plainly that the cause is unknown

**Nothing matched?** Open an issue with the two timestamps and the gap length. That pattern alone is often diagnosable.

*Verified: macOS 26.5, system tools · 2026-08. `crontab -l`, `ps -o lstart`, `uptime` run as written. `journalctl` and `kubectl` paths are NOT runtime-verified here.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
