# Vendor patches over upstream httpz

Source: https://github.com/karlseguin/http.zig (branch `master`)
Pinned upstream commit: `52eb187cff75e168316b44480cd9f0fae26a1c5e`

This directory is a copy of upstream at the commit above plus the two local
patches below. Upstream now includes the non-blocking shutdown fix that this fork
previously carried. Drop this fork and re-pin to upstream once the shared-queue
and worker-mode patches land there.

## Patch 1 — ThreadPool: shared injector queue replaces per-thread private queues

**File:** `src/thread_pool.zig`

**Symptom upstream:** A pool thread parked inside a long-running job black-holes a
share of all later requests. Each pool thread owned a private ring queue, and both
dispatch paths (`flush` for the non-blocking worker's event batches, `spawnOne` for
the blocking worker's accepts) assigned work to ONE thread by blind round-robin,
with no rebalancing: jobs queued behind a busy thread stay there even while every
other thread idles. Work-stealing was started upstream but never finished — each
worker had a `peer` field for it, wired with an `i + i` typo (`workers[@mod(i + i,
workers.len)]`), so for even pool sizes every peer pointer lands on an
even-indexed worker (worker 0's peer is itself), and the single-peer
`getNext(false)` probe never compensated for the round-robin placement anyway.
Observed in production shape: one handler holding its pool thread (an SSE stream,
before this repo moved streams off the pool) made roughly every `1/count`-th
request hang unserved with idle CPU; it also wedged `server.stop()`, which joins
pool threads. Platform-independent — the queues sit above kqueue/epoll.

**Fix:** One shared bounded multi-producer/multi-consumer queue for the whole pool
(same `Io.Mutex`/`Io.Condition` primitives, same ring-buffer arithmetic): any idle
thread claims the next job, so a parked thread costs exactly one thread, never a
queue share. The `Worker` type and its dead `peer` field are deleted; the queue
state lives in an arena-allocated `Shared` struct because `init` returns the pool
by value while threads hold pointers into it. Public surface preserved verbatim —
`spawn`/`spawnOne`/`flush(batch_size)`/`empty`/`stop`/`deinit` signatures and the
producer-visible `batch_size` field are unchanged, as are the semantics callers
rely on: producer batching to amortize locking, producer blocking when the queue
is full (backpressure to the accept/event loop), drain-before-exit on `stop()`,
and idempotent `stop()`. The `backlog` knob now sizes the single shared queue
rather than `count` private rings — total standing capacity is `backlog`, not
`count × backlog`; the per-dispatch-point bound is what it always was. New
`pending()` exposes the queued-job depth as a load signal for admission control.
Multi-job pushes wake one waiter per queued job (bounded by the pool size)
instead of broadcasting — a broadcast stampeded every idle thread at the shared
mutex on each event-loop batch.

**Tests:** existing pool tests (`batch add`, `small fuzz`, `large fuzz`) pass
unmodified; new `parked thread cannot starve queued jobs` pins the fix (it hangs
forever on the old dispatch) and `pending reports queued depth` pins the new
surface. A follow-up coverage pass adds four more: `stop drains jobs queued
before it`, `stop is idempotent`, `pending counts across ring wraparound`, and
`batch push wakes enough threads for the batch`. Run from this directory:
`zig build test`.

**Upstream PR:** TBD — Indy's call on filing (this rewrite vs a minimal
work-stealing fix are different upstream conversations).

## Patch 2 — Keep the websocket dependency in the selected worker mode

**File:** `build.zig`

**Symptom upstream:** `zig build test -Dforce_blocking=true` fails because
`httpz` selects its blocking worker while `websocket.zig` remains in its default
non-blocking mode. The resulting websocket worker has no `readLoop` method.

**Fix:** Pass the existing `force_blocking` option to `websocket.zig` as
`websocket_blocking`. This keeps both libraries in the same mode and restores the
blocking test suite without changing the default non-blocking build.
