# Usage & API

```sh
go get github.com/go-coord/coord
```

All constructors take an **open** `*clientv3.Client` — go-coord never dials.

## Host liveness

```go
type HostMetadata struct {
    HostUUID   string `json:"host_uuid"`
    Hostname   string `json:"hostname"`
    Hypervisor string `json:"hypervisor"`
    Version    string `json:"version,omitempty"`
    StartedAt  int64  `json:"started_at_unix_ns"`
}

type LivenessOptions struct {
    Prefix      string        // defaults to HostsPrefix ("/coord/hosts/")
    LeaseTTLSec int64         // defaults to DefaultLeaseTTLSec (10)
    Refresh     time.Duration // defaults to DefaultRefreshInterval (3s)
    Logger      *slog.Logger  // defaults to a discard handler
}

func RegisterHostLiveness(ctx context.Context, cli *clientv3.Client, meta HostMetadata, opts LivenessOptions) (*HostLiveness, error)
func (h *HostLiveness) Stop(ctx context.Context) error      // idempotent; revokes the lease
func (h *HostLiveness) LeaseID() clientv3.LeaseID           // tracks the live lease across self-heal
func (h *HostLiveness) Key() string
```

`RegisterHostLiveness` grants a lease, puts the metadata, and launches the
keep-alive goroutine. It fails fast on a nil client, an empty `HostUUID`, or an
etcd grant/put/keepalive error. `Stop` is safe to `defer`.

## Host watcher

```go
type HostEventKind int
const ( HostUp HostEventKind = iota; HostDown )

type HostEvent struct {
    Kind     HostEventKind
    HostUUID string
    Metadata HostMetadata // populated on Up; last-seen on Down
}

type WatcherOptions struct {
    Prefix      string       // defaults to HostsPrefix
    Logger      *slog.Logger // defaults to discard
    IncludeSelf string       // if non-empty, suppress events for this HostUUID
}

func NewHostWatcher(ctx context.Context, cli *clientv3.Client, opts WatcherOptions) (*HostWatcher, error)
func (w *HostWatcher) Events() <-chan HostEvent // closes when ctx is cancelled
func (w *HostWatcher) Wait()                    // blocks until the watch goroutine exits
```

The channel is buffered (32) and closed cleanly when `ctx` is cancelled, so a
`for range w.Events()` loop terminates on shutdown.

## Leader election

```go
type ElectionOptions struct {
    Key      string       // etcd prefix the election locks on
    TTL      int          // session TTL in seconds; default 10
    Identity string       // value written to the leader key
    Logger   *slog.Logger
}

func NewElection(ctx context.Context, cli *clientv3.Client, opts ElectionOptions) (*Election, error)
func (e *Election) Campaign(ctx context.Context, identity string) error         // blocks until leader
func (e *Election) TryCampaign(ctx context.Context, identity string) (bool, error) // non-blocking
func (e *Election) Resign(ctx context.Context) error
func (e *Election) Observe(ctx context.Context) <-chan string                   // current leader per transition
func (e *Election) Close() error                                                // releases the session
```

## Pooled election

```go
type PoolOptions struct {
    TTLSec int          // session lease TTL; defaults to 30s
    Logger *slog.Logger // defaults to a discard handler
}

func NewElectionPool(cli *clientv3.Client, opts PoolOptions) *ElectionPool
func (p *ElectionPool) Election(ctx context.Context, key string) (*Election, error) // reuses one session per key
func (p *ElectionPool) Close() error   // releases every pooled session
func (p *ElectionPool) Stats() PoolStats

type PoolStats struct {
    SessionCount int
    Closed       bool
    TTLSec       int
}
```

A pool-borrowed `Election` has a no-op `Close()`; the pool owns the session.

## Full example

```go
package main

import (
	"context"
	"log"

	"github.com/go-coord/coord"
	clientv3 "go.etcd.io/etcd/client/v3"
)

func main() {
	cli, err := clientv3.New(clientv3.Config{Endpoints: []string{"127.0.0.1:2379"}})
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()
	ctx := context.Background()

	hl, err := coord.RegisterHostLiveness(ctx, cli,
		coord.HostMetadata{HostUUID: "h1", Hostname: "dc1-h1"}, coord.LivenessOptions{})
	if err != nil {
		log.Fatal(err)
	}
	defer hl.Stop(ctx)

	w, err := coord.NewHostWatcher(ctx, cli, coord.WatcherOptions{IncludeSelf: "h1"})
	if err != nil {
		log.Fatal(err)
	}
	go func() {
		for ev := range w.Events() {
			log.Printf("%v %s", ev.Kind, ev.HostUUID)
		}
	}()

	el, err := coord.NewElection(ctx, cli, coord.ElectionOptions{Key: "/coord/elect/reconcile"})
	if err != nil {
		log.Fatal(err)
	}
	defer el.Close()
	if err := el.Campaign(ctx, "h1"); err != nil {
		log.Fatal(err)
	}
	log.Println("leader: doing the coordinated work")
}
```
