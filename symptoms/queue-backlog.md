# Jobs or messages aren't being processed

**DON'T:** purge the queue, restart the workers, or scale consumers to fifty.
Purging deletes the only copy of the work. Restarting loses everything in flight. Scaling ten times on a poison message multiplies one failure into fifty.

**First:** is the queue growing, or have the consumers stopped? Those look identical on a dashboard and need opposite fixes.

1. Measure depth twice, ten seconds apart: `redis-cli -h HOST llen QUEUE` `[redis]` · `rabbitmqctl list_queues name messages messages_unacknowledged` `[rabbitmq]` · `aws sqs get-queue-attributes --queue-url URL --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible` `[sqs]`
   → growing → producers are outrunning consumers, go to 2 · flat and non-zero → consumers are stuck, go to 3
2. Count live consumers: `kubectl get pods -l app=WORKER` `[k8s]` · `rabbitmqctl list_consumers` `[rabbitmq]`
   → zero or fewer than expected → they crashed → `oom-killed.md` · the expected number → 3
3. Watch the in-flight / unacknowledged count across two samples.
   → constant and non-zero → one message is being retried forever: a poison message, go to 4 · rising then falling → they are working, just slow, go to 5
4. Peek without consuming: `redis-cli -h HOST lindex QUEUE 0` `[redis]` · `aws sqs receive-message --queue-url URL --visibility-timeout 0` `[sqs]`
   → you can now read the payload that kills the worker · caveat: SQS `receive-message` increments the message's receive count even at `--visibility-timeout 0`, so if it is one attempt from the redrive limit, peeking is what sends it to the DLQ
5. Check what the consumer waits on: its DB and its HTTP dependency.
   → DB slow → `db-connections-exhausted.md` · dependency slow → `slow-everything.md`
6. Read the dead letter queue depth and one message from it.
   → growing → messages are failing, not stuck, and that one message contains the real error · empty while the main queue grows → nothing is even being attempted; go back to 2

**Nothing matched?** Open an issue naming your broker and what the two depth samples showed.

*Commands written against vendor docs; no broker available on the authoring machine, so steps 1-4 and 6 are NOT runtime-verified.*

← [back to the router](../README.md) · [after the fire](../AFTER-THE-FIRE.md)
