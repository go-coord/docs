# Failure handling

Coordination libraries earn their keep at the edges — when etcd blips, a leader
changes, or a host is partitioned. This page documents exactly what go-coord
does in each case.

## The self-healing keep-alive loop

A naive keep-alive returns when etcd closes the `KeepAlive` stream (a network
blip, an etcd leader change, a server restart, or an externally-revoked lease).
That is a trap: the host would disappear from the cluster view and **stay gone
for the rest of its uptime**, until someone manually restarted it.

go-coord's keep-alive goroutine instead **re-registers** when the stream closes:

1. It drains the active `KeepAlive` channel until it closes.
2. On close, it logs a warning and re-runs `Grant` + `Put` + `KeepAlive` under a
   fresh lease, with **exponential backoff** (100 ms → doubling → capped at 30 s).
3. The cached metadata (`blob`) and TTL are reused, and the new lease ID is
   published atomically so a concurrent `Stop()` always revokes the current
   lease.

The host does drop out of the cluster view during the gap — that is the intended
HA signal — but it **reappears as soon as connectivity returns**, instead of
staying dark forever.

```go
// Nothing to do on your side: the loop is internal. LeaseID() tracks the live
// lease so diagnostics see the re-registered value after a heal.
id := hl.LeaseID()
```

## Graceful vs. ungraceful stop

- **Graceful:** `hl.Stop(ctx)` cancels the keep-alive loop and **revokes** the
  lease, so the host's key disappears *immediately* — no waiting a full TTL.
  `Stop` is idempotent and safe to `defer`.
- **Ungraceful (crash / partition):** nothing revokes the lease, so etcd expires
  it after the TTL and deletes the key. Watchers see the `HostDown` up to one
  TTL after the failure.

## Watcher robustness

- The watcher watches from the **snapshot revision + 1**, so no event is lost
  between the initial `Get` and the `Watch`.
- Transient watch errors (e.g. a compaction) are logged and skipped; the watch
  continues.
- The event channel is closed cleanly when the context is cancelled, so
  consumers' `for range` loops terminate on shutdown.

## Election safety

- `Campaign` returns an error wrapping `ctx.Err()` on cancellation, so a bounded
  `context.WithTimeout` turns "become leader" into "try to become leader for N".
- A crashed leader's session lease is auto-revoked by etcd within one TTL,
  freeing a successor to win.
- `ElectionPool.Close()` revokes every pooled session, dropping any held
  leadership immediately.

## Tuning the TTL

The lease TTL is the failover budget:

- **Shorter** → faster detection of a dead host, but less tolerance for a missed
  refresh (a long GC pause or a brief network stall could expire a healthy lease).
- **Longer** → more tolerance, but slower failover.

The default of 10 s with a ~3 s refresh (roughly TTL/3) tolerates a single
dropped renew — the same half-period logic as an `sd_notify` watchdog.
