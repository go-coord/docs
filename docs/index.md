# go-coord documentation

**Pure-Go (no cgo) cross-host coordination on etcd v3** — host liveness with TTL
leases, an up/down host watcher, and leader election (single and pooled).

`go-coord/coord` gives a fleet of agents the primitives they need to coordinate
high-availability behaviour across hosts. The module path is
`github.com/go-coord/coord`.

!!! success "Status: complete — 100% coverage, 6 arches"
    `HostLiveness` (TTL leases + self-healing keep-alive), `HostWatcher`
    (synthetic-`Up` snapshot + `Up`/`Down` stream), and `Election` /
    `ElectionPool` (leader election) are implemented over
    `go.etcd.io/etcd/client/v3` and `/concurrency`. CGO-free, no vendoring,
    `gofmt` + `go vet` clean, **100% test coverage** including every error
    branch, CI green across the six 64-bit Go targets.

## What it is

- **HostLiveness** registers a short-TTL lease at `/coord/hosts/<host_uuid>` and
  refreshes it in the background. Lease expiry is the cluster-wide signal that a
  host is down (process death, network partition, kernel panic). The keep-alive
  loop **self-heals** if etcd drops the renewal stream.
- **HostWatcher** replays every existing host as a synthetic `Up` event, then
  streams `Up`/`Down` events; `Down` carries the host's last-seen metadata.
- **Election / ElectionPool** wrap etcd-concurrency leader election scoped to a
  key, so cross-host work coalesces to one leader per key. The pool keeps one
  long-lived session per key.

> **The caller owns the etcd connection.** go-coord takes an open
> `*clientv3.Client`; it never dials or fans out extra connections.

## Quick taste

```go
hl, _ := coord.RegisterHostLiveness(ctx, cli, coord.HostMetadata{HostUUID: "h1"}, coord.LivenessOptions{})
defer hl.Stop(ctx)

w, _ := coord.NewHostWatcher(ctx, cli, coord.WatcherOptions{})
for ev := range w.Events() { /* HostUp / HostDown */ }

el, _ := coord.NewElection(ctx, cli, coord.ElectionOptions{Key: "/coord/elect/reconcile"})
_ = el.Campaign(ctx, "h1") // blocks until leader
```

## Repositories

| Repo | What it is |
| --- | --- |
| [`coord`](https://github.com/go-coord/coord) | the library — liveness, watcher, election |
| [`docs`](https://github.com/go-coord/docs) | this documentation site (MkDocs Material, versioned with mike) |
| [`go-coord.github.io`](https://github.com/go-coord/go-coord.github.io) | the organization landing page (Hugo) |
| [`brand`](https://github.com/go-coord/brand) | logo and brand assets |

## Principles

- **Pure Go, `CGO_ENABLED=0`** — trivial cross-compilation, a single static
  binary, no C toolchain.
- **No vendoring.** The etcd client is a normal build-from-source dependency.
- **Caller owns the connection.** go-coord takes an open `*clientv3.Client`.
- **100% test coverage** is the target, enforced as a CI gate.

## Where to go next

- [Concepts](concepts.md) — leases, watches, elections and how go-coord uses them.
- [Usage & API](api.md) — the full surface with examples.
- [Failure handling](failure-handling.md) — the self-heal loop, backoff, and
  what happens on partitions and crashes.
- [Roadmap](roadmap.md) — what is done and what may come next.

Source lives at [github.com/go-coord/coord](https://github.com/go-coord/coord).
