# Distributed lifecycle checks

Use this checklist for backend queues, workers, leases and persisted workflows.
Check worker/consumer registration and configured topology (bindings, retry
routes, concurrency and replicas) as assumptions behind the trace. Code presence
does not prove execution. Use repository evidence and name remaining assumptions;
this checklist does not authorize production access.

- **Continuation and admission:** Who continues work after an ACK, rejected
  queue admission or an elapsed deadline? Trace durable state, queue ownership,
  retry and deduplication. Can rejection leave a pending record with no worker or
  wakeup? Can timed-out work still finish and apply effects?
- **Terminal meaning:** Separate transport terminal states (ACK, response,
  timeout, cancellation) from domain completion or failure. Trace the transition
  that settles the domain operation and who can still write afterward.
- **State after I/O:** Check whether a worker rereads persisted state after
  external I/O and validates the current status, version, owner and lease where
  relevant. A reread alone leaves a race before the write: verify an atomic write
  predicate or concurrency check fences stale owners and results. Local generation
  counters do not fence other processes.
- **Delivery and commit gaps:** Trace duplicate and reordered delivery, and a
  crash between claim, side effect, result persistence and ACK. Check what the
  next worker can discover and whether retry repeats an effect or loses work.
  Preserve the operation's idempotency contract; a database transaction alone
  does not make an external effect atomic with a commit.
- **Recovery progress and selection:** Identify the durable retry/wakeup or
  reconciliation path after interruption. Does each pass advance or reschedule
  work, including expired leases? Check whether a repeatedly selected failing
  record or fixed first-page limit starves later eligible records. Bounded
  batches must eventually cover the full eligible scope, not silently truncate it.
