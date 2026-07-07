# Concepts

go-coord is a thin, opinionated layer over three etcd v3 primitives. This page
explains each and how the library uses it.

## Leases and host liveness

etcd **leases** are time-bounded: a key attached to a lease is deleted when the
lease expires. `HostLiveness` grants a lease (default TTL 10s), puts the host's
metadata under `/coord/hosts/<host_uuid>` attached to that lease, and then keeps
the lease alive with a background `KeepAlive` stream.

The consequence is simple and robust: **as long as an agent is healthy and
connected, its key exists; the moment it dies or is partitioned long enough, the
lease expires and etcd deletes the key.** Every other agent watching the prefix
learns the host is gone. TTL is the failover budget — short enough to react,
long enough to absorb a single missed refresh (a GC pause, say).

```go
hl, err := coord.RegisterHostLiveness(ctx, cli, coord.HostMetadata{
    HostUUID: "host-uuid-1", Hostname: "dc1-h1", Hypervisor: "qemu",
}, coord.LivenessOptions{LeaseTTLSec: 10})
...
defer hl.Stop(ctx) // revokes the lease immediately — no TTL wait on graceful exit
```

`HostMetadata` is a small JSON value — keep it small, since every watcher
decodes it.

## Watches and the host view

`HostWatcher` turns the liveness prefix into an event stream. It does two things:

1. **Snapshot.** On start it does a prefix `Get` and emits a synthetic
   `HostUp` for every host already registered, so a fresh agent has a baseline
   view without waiting for the next renew.
2. **Watch.** It then watches the prefix from the snapshot's revision (so no
   event is missed in the gap) and emits `HostUp` on PUT and `HostDown` on
   DELETE. `HostDown` carries the host's *last-seen* metadata (via etcd's
   `WithPrevKV`) so you can name the host without a separate inventory lookup.

```go
w, _ := coord.NewHostWatcher(ctx, cli, coord.WatcherOptions{IncludeSelf: "host-uuid-1"})
for ev := range w.Events() {
    switch ev.Kind {
    case coord.HostUp:   // ev.Metadata is populated
    case coord.HostDown: // ev.HostUUID (+ last-seen metadata)
    }
}
```

`IncludeSelf` suppresses events for your own host UUID so you don't react to
yourself.

## Elections and single-leader work

When a host goes down, you often want exactly **one** surviving agent to react
(claim its work), not all of them at once. `Election` wraps etcd-concurrency
leader election scoped to a key:

- `Campaign` blocks until you win (or ctx is cancelled);
- `TryCampaign` is non-blocking — `(true, nil)` if you won, `(false, nil)` if
  someone else holds it;
- `Resign` gives up leadership so a successor can take over;
- `Observe` streams the current leader's identity to followers.

```go
el, _ := coord.NewElection(ctx, cli, coord.ElectionOptions{Key: "/coord/elect/reconcile"})
defer el.Close()
if err := el.Campaign(ctx, "host-uuid-1"); err == nil {
    // we are the single leader for this key; do the coordinated work
}
```

### The pool

Creating a fresh session (a lease grant) for every election is wasteful when you
have many keys and frequent events. `ElectionPool` keeps **one long-lived
session per key**, so the first encounter of a key grants a session and every
later borrow reuses it. Sessions are auto-revoked by etcd within one TTL if the
agent crashes.

```go
pool := coord.NewElectionPool(cli, coord.PoolOptions{TTLSec: 30})
defer pool.Close()

el, _ := pool.Election(ctx, "/coord/elect/rule-42")
won, _ := el.TryCampaign(ctx, "host-uuid-1")
```

An Election borrowed from the pool has a no-op `Close()` — the pool owns the
session lifetime; call `pool.Close()` to release everything.
