# Amadeus fork changelog

This file tracks Amadeus-local changes to this fork of
[`k8s.io/client-go`](https://github.com/kubernetes/client-go). It is separate from the
upstream [CHANGELOG.md](CHANGELOG.md), which is left untouched so that rebases onto new
upstream releases stay conflict-free.

Versioning scheme: `<upstream version>-amadeus.<n>`. The prerelease suffix keeps the
upstream lineage legible in the version string itself. Note that semver orders a
prerelease *below* its release (`v0.34.0-amadeus.1` < `v0.34.0`), which is why consumers
should pin the fork through a `replace` directive rather than relying on `@latest`.

Each entry records the upstream base, the affected files, and whether the change is
intended to go upstream or is a permanent local divergence.

## v0.34.0-amadeus.1

Upstream base: `k8s.io/client-go` v0.34.0 (tag `kubernetes-1.34.0`, commit `b1c7d7bb6`).

### Added

- **`tools/cache`: pluggable at-rest storage codec for `ThreadSafeStore`.**
  Introduces a `StorageCodec` interface (`Encode`/`Decode`) and a
  `WithThreadSafeStoreStorageCodec` functional option on `NewThreadSafeStore`, allowing
  values to be transformed before they are held in the store's `items` map. Indices
  continue to be computed over *decoded* values, so indexing behaviour is unchanged.

  Files: `tools/cache/thread_safe_store.go`, `tools/cache/thread_safe_store_test.go`.

  Upstreamable: yes, in principle — the `StorageCodec` seam is generic. Would need the
  default-codec change below reverted to a pass-through before proposing.

### Changed

- **`NewThreadSafeStore` now defaults to a gzip codec rather than pass-through.**
  When no `WithThreadSafeStoreStorageCodec` option is supplied, the store uses
  `gzipStorageCodec`, which compresses `[]byte` values and passes all other types
  through unchanged. Because every informer and indexer in client-go constructs its
  store via `NewThreadSafeStore`, this applies fork-wide by default.

  Files: `tools/cache/thread_safe_store.go`.

  Upstreamable: no. Permanent local divergence.

  Behavioural consequences to be aware of:

  - Values that are not `[]byte` are unaffected functionally, but every element
    returned by `Get`, `List`, `Index`, and `ByIndex` now passes through a type
    assertion on client-go's hottest read path.
  - For `[]byte` values, `Get` decompresses into a **freshly allocated slice** on each
    call. Repeated `Get`s therefore no longer return a pointer-identical object, and
    callers that compare by identity will need adjusting.
  - `List`, `Index`, and `ByIndex` decode every element they return, making a `List`
    over a large compressed store O(n) decompressions.

- **Encode/decode failures panic.** `threadSafeMap.encode`/`decode` panic on codec
  error, because the `ThreadSafeStore` interface has no error return on these paths.
  A panic in `Update` or `Delete` occurs while the store lock is held.

  Files: `tools/cache/thread_safe_store.go`.

  Upstreamable: no — an upstream version would need the interface to return errors.

### Verification

`go build ./tools/cache/` clean; full `go test ./tools/cache/` suite passes, including
three new tests covering codec round-tripping with decoded indexing, gzip compression of
`[]byte` values, and pass-through of non-`[]byte` values.

### License notice

This fork is distributed under the Apache License 2.0, unchanged; see [LICENSE](LICENSE).
Per Apache-2.0 §4(b), files modified by Amadeus are to carry a prominent notice stating
that they were changed. Release notes are **not** required by the license — this file
exists for engineering traceability across upstream rebases.
