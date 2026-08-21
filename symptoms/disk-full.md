# The disk is full

**DON'T:** delete logs, empty `/tmp`, or truncate a file a process still has open.
Deleting a file that is still open frees nothing until the process closes it — you get no space back and you destroyed the log you were about to read. Truncating with `>` while it is open can leave a sparse file that reports the old size.

**First:** is it out of bytes, or out of inodes? Same symptom, unrelated causes.

1. `df -h /` and `df -i /`
   → bytes at 100% → 2 · inodes at 100% while bytes are fine → millions of small files, go to 4 · neither at 100% → the full filesystem is a different mount; run `df -h` with no argument and read every line
2. `du -xhd1 / 2>/dev/null | sort -h | tail -10` — one form for all three `du`s; GNU, BSD and busybox all take `-d`, where `--max-depth` is GNU-only
   → one directory dominates → descend into it by repeating the command · nothing dominates → the space is held by deleted-but-open files, go to 3 · it is still running after a minute → `du` walks the whole filesystem and costs real IO on a box that is already struggling; narrow it to `/var` and repeat
3. `lsof +L1 2>/dev/null | awk '$8==0 && $7>50000000'`
   → a hit → those bytes are held hostage by a running process and come back only when it closes the file, not when you delete it again · nothing → 4
4. `find /var -xdev -type f 2>/dev/null | wc -l` then repeat for other suspects
   → one tree holds millions of files → usually a session, cache, or mail spool directory with no cleanup
5. `ls -lSh /var/log 2>/dev/null | head -10`
   → one file far larger than the rest → log rotation is broken or the writer reopens by inode; note the writer before touching anything
6. `docker system df` `[docker]` · `journalctl --disk-usage` `[linux]` · `du -xhd1 /var/lib 2>/dev/null | sort -h | tail -5`
   → image layers or the journal dominate → the space is not "yours"; it belongs to a runtime with its own eviction settings, which is a `SystemMaxUse=` or image-GC decision and not a command you run now

**Nothing matched?** Open an issue with the `df -h` and `df -i` lines.

*Verified: macOS 26.5, system tools · 2026-08. Steps 1, 2 (macOS form), 3, 4 and 5 run as written — `lsof +L1` does work on macOS despite being widely documented as Linux-only. The size filter in step 3 is not cosmetic: unfiltered, `+L1` returned 570 harmless matches on a healthy machine and exactly one real one, a 101 MB deleted-but-open browser temp file. Step 2 is now the single portable form, verified on macOS but not on GNU or busybox `du`; `journalctl --disk-usage` is unrun. See the [run matrix](../README.md#where-each-file-has-been-run).*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
