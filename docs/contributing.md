# Contributing

Contributions are welcome. go-coord holds to a few non-negotiables that keep the
library dependable and portable.

## Ground rules

- **Pure Go, `CGO_ENABLED=0`.** No cgo, ever. The library must cross-compile and
  embed anywhere.
- **No vendoring.** Dependencies build from source; the etcd client is a normal
  module dependency.
- **100% test coverage**, including every error branch, is a CI gate — not an
  aspiration. New code lands with the tests that cover it.
- **`gofmt` + `go vet` clean.**
- **All six 64-bit Go targets** stay green: amd64, arm64, riscv64, loong64,
  ppc64le, s390x.

## The two test layers

The suite is deliberately split so it is both deterministic and realistic:

- **Seam-based unit tests** (`coord_test.go`, no build tag) use in-package fakes
  — a fault-injecting `etcdKV`, a fake session and election backend — plus
  controllable time/marshal seams to drive **every** branch (including the etcd
  grant/put/keepalive/revoke/watch error paths and the keep-alive backoff)
  without any network. These build and run on every arch, including the qemu
  lanes.
- **Embedded-etcd integration tests** (`coord_integration_test.go`, behind the
  `etcd_integration` build tag) boot a real `embed.Etcd` on loopback and exercise
  the whole thing end-to-end. They run on the ubuntu/macos lanes, where coverage
  is measured.

When you add a branch, cover it in the seam tests (so it runs everywhere) and,
where it exercises real etcd semantics, in the integration tests too.

## Running the tests

```sh
# full suite + coverage (needs a Go toolchain that can build the embedded server)
COVERPKG=$(go list ./... | paste -sd, -)
go test -tags etcd_integration -race -coverpkg="$COVERPKG" -coverprofile=cover.out ./...
go tool cover -func=cover.out | tail -1   # must be 100.0%

# the network-free lane the qemu arches run
go test ./...
```

## License

By contributing you agree your contribution is licensed under the project's
**BSD-3-Clause** license.
