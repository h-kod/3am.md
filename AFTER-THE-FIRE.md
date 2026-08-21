# After the fire

The acute phase is over. What you do in the next twenty minutes decides whether this incident teaches you anything at all.

**DON'T:** close the incident, archive the channel, tidy up the debug artifacts, or promise a postmortem for tomorrow.
Tomorrow you will have the story but not the evidence. Logs rotate, high-resolution metrics get rolled up, containers are replaced, queues drain. And the story you remember will be tidier than what happened, in exactly the places that matter.

**First:** capture what is on a clock. Everything else can wait until morning; this cannot.

1. Copy out the perishable things: application logs, the *dead* container's logs, load balancer access logs, queue contents, the high-resolution metric window.
   → most stacks keep per-second data for hours and per-minute for days, so tomorrow you will not be able to zoom into this window at all
2. Write the timeline in UTC, and next to every line write where the timestamp came from.
   → "02:14 first alert (pager)" is evidence · "around 2am" is a memory, and memories lose arguments
3. Write down what you actually did, including everything that did nothing.
   → the failed attempts are the most useful part of the record; without them the next person tries all of them again, in the same order
4. Write down what you nearly did and chose not to do — the restart, the rollback, the purge.
   → this is the most valuable line in any postmortem and the one nobody writes, because not doing something does not feel like an event
5. Name the one signal that would have told you sooner, and where it would have had to appear to be noticed.
   → if the answer is "a dashboard nobody was watching", the fix is routing and alerting, not more instrumentation
6. Choose exactly one change you will actually ship. One.
   → a postmortem with twelve action items ships zero of them, and the review meeting is where twelve becomes zero

**Then** close it — with the timeline attached, not the story.

*Nothing on this page changes any system. Only steps 1 and 2 have a deadline.*

← [back to the router](README.md)
