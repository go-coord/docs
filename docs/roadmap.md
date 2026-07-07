# Roadmap

## Done

- **HostLiveness** — TTL-lease registration, background refresh, immediate
  revoke on `Stop`, and a **self-healing** keep-alive loop with exponential
  backoff.
- **HostWatcher** — synthetic-`Up` snapshot, gap-free watch from the snapshot
  revision, `Down` events carrying last-seen metadata, and self-suppression.
- **Election** — `Campaign`, non-blocking `TryCampaign`, `Resign`, `Observe`.
- **ElectionPool** — one long-lived session per key, `Stats` for diagnostics.
- **Quality** — CGO-free, no vendoring, `gofmt` + `go vet` clean, **100% test
  coverage** including every error branch, CI green across the six 64-bit Go
  targets (amd64, arm64, riscv64, loong64, ppc64le, s390x).

## Possible future work

- **Distributed mutex / RWMutex** helpers on top of etcd-concurrency's `Mutex`,
  mirroring the `Election` ergonomics.
- **Configurable refresh cadence** surfaced explicitly (today the keep-alive
  cadence is driven by etcd's lease machinery; `LivenessOptions.Refresh` is
  reserved for a future manual-refresh mode).
- **Metrics hooks** — counters for re-registrations, elected/resigned
  transitions, and watcher lag, for `/metrics` surfacing.
- **Typed metadata** generic over the JSON value, so callers can carry their own
  host descriptor without a second decode.

Nothing here is required for the current use cases; the core primitives are
complete. Suggestions welcome via issues on
[github.com/go-coord/coord](https://github.com/go-coord/coord).
