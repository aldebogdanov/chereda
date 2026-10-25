# chereda

Queue effect for Irij. SQLite-backed. At-least-once delivery with optional dedup keys.

## Install

`irij.toml`:

```toml
[seeds]
chereda = "0.1.0"
```

## Use

```irij
use chereda :open
use std.db :open

;; One-time at startup. Opens the SQLite DB, creates schema, binds
;; the connection to chereda's internal slot.
chereda-init "/var/lib/myapp/queue.db"

with default-queue
  ;; Push.
  enqueue "jobs" (json-encode {kind= "resize" path= "/foo"}) ""

  ;; Idempotent push — same dedup-key in the same topic is dropped.
  enqueue "jobs" (json-encode {kind= "resize" path= "/foo"}) "resize-foo"

  ;; Worker loop fragment.
  msg := try-dequeue "jobs"
  if (get "ok" msg)
    process-and-ack (get "id" msg) (get "payload" msg)
  else
    sleep 1000
```

## Effect surface

```
effect Queue
  enqueue     :: Str Str Str ()     ;; topic, payload, dedup-key (empty for none)
  try-dequeue :: Str Map            ;; topic -> {ok, id, payload} | {ok=false}
  ack         :: Int ()             ;; mark inflight -> done
  nack        :: Int ()             ;; mark inflight -> queued (requeue)
  queue-len   :: Str Int            ;; count of `queued` rows
```

`default-queue :: Queue ::: Db` keeps messages in SQLite through std.db,
so a function that installs it declares `::: Db` (as `chereda-init`,
`::: Console Db`, does); code that only enqueues and dequeues declares
`::: Queue`. chereda has no Java of its own.

## Semantics

- **At-least-once.** A message is dequeued (queued → inflight) and held until
  ack. If the worker crashes before ack, the message stays inflight forever
  unless someone nacks it. Make workers idempotent.
- **Optional dedup.** Pass a non-empty `dedup-key` on enqueue. The (topic,
  dedup-key) pair is UNIQUE; repeat enqueues are silently dropped via
  `INSERT OR IGNORE`. Empty key skips dedup entirely.
- **FIFO per topic.** Dequeue picks the smallest id in `status='queued'` —
  monotonic id = insertion order.
- **Topic-partitioned.** Topics are independent; queue-len, dequeue, ack
  all scope to one topic at a time.

## Storage

One SQLite table:

```sql
CREATE TABLE chereda_msgs (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  topic       TEXT NOT NULL,
  payload     TEXT NOT NULL,
  dedup_key   TEXT,
  status      TEXT NOT NULL DEFAULT 'queued',
  enqueued_at TEXT NOT NULL DEFAULT (datetime('now')),
  picked_at   TEXT,
  UNIQUE(topic, dedup_key)
);
CREATE INDEX idx_chereda_topic_status ON chereda_msgs(topic, status);
```

`done` rows accumulate; trim them externally if size matters.

## Planned

- `chereda-redis` — separate seed adding a Redis-backed cap for cross-process
  / cross-host work queues.
- Lease-based at-least-once — `inflight` rows expire after a configurable TTL
  and bounce back to `queued` automatically.
- Worker helper — `fn run-worker :: Str Fn () ::: Queue` that loops
  try-dequeue + ack/nack around a user-supplied handler.

## License

MIT
