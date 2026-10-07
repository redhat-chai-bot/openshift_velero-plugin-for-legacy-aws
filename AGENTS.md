# AGENTS.md — velero-plugin-for-legacy-aws

> Guidance for AI coding agents working in this repository.

## Project Overview

This is the **legacy** Velero plugin for AWS, maintained under
[openshift/velero-plugin-for-legacy-aws](https://github.com/openshift/velero-plugin-for-legacy-aws).
It provides backup/restore object-storage and volume-snapshot capabilities for
Velero running on AWS, using **AWS SDK for Go v1** (`github.com/aws/aws-sdk-go`).

The default development branch is **`oadp-dev`**.

The Go module path is `github.com/vmware-tanzu/velero-plugin-for-aws` (inherited
from the upstream project). The downstream fork replaces the Velero dependency
with `github.com/openshift/velero` via a `replace` directive in `go.mod`.

### Key source layout

```
velero-plugin-for-aws/   # All plugin Go source (single package, package main)
  main.go                # Plugin entry point — registers ObjectStore & VolumeSnapshotter
  object_store.go        # S3-backed ObjectStore implementation
  volume_snapshotter.go  # EBS VolumeSnapshotter implementation
  helpers.go             # Shared AWS session/config helpers
  v1_sign_request_handler.go  # AWS Signature v1 request signing
  *_test.go              # Unit tests
hack/
  build.sh               # Build script invoked by `make local`
  cp-plugin/main.go      # Init-container helper that copies the plugin binary
Makefile                 # Build, test, container, and module targets
Dockerfile               # Multi-stage build producing a scratch-based image
```

## Build & Development Commands

### Prerequisites

- **Go 1.22+** (the module declares `go 1.25.0`; any recent Go toolchain works)
- Docker with BuildKit / `docker buildx` (for container builds only)

### Build locally

```bash
make local
```

Produces the plugin binary at `_output/bin/<GOOS>/<GOARCH>/velero-plugin-for-aws`.

### Run unit tests

```bash
make test
```

This runs:

```bash
CGO_ENABLED=0 go test -v -coverprofile=coverage.out -timeout 60s ./...
```

Tests live alongside the source in `velero-plugin-for-aws/*_test.go`.

### Run a single test

```bash
CGO_ENABLED=0 go test -v -run TestFunctionName ./velero-plugin-for-aws/
```

### CI check (module verification + tests)

```bash
make ci
```

Runs `make verify-modules` (ensures `go.mod`/`go.sum` are tidy) then `make test`.

### Tidy modules

```bash
make modules
# or equivalently:
go mod tidy
```

### Verify modules are clean

```bash
make verify-modules
```

Fails if `go.mod` or `go.sum` have uncommitted changes after `go mod tidy`.

### Build container image

```bash
make container
```

Requires Docker BuildKit (`docker buildx`). Builds a scratch-based image
containing the plugin binary and the `cp-plugin` init-container helper.

### Clean build artifacts

```bash
make clean
```

## Testing Guidelines

- All test files use the standard Go `testing` package with
  `github.com/stretchr/testify` for assertions and mocks.
- Tests are in the same package (`package main`) as the source, so they have
  access to unexported symbols.
- There is no separate integration or e2e test suite in this repository.
  End-to-end testing is handled by the OADP operator test suite.
- When adding or modifying functionality, add or update the corresponding
  `*_test.go` file and ensure `make test` passes before submitting.

## Code Style & Conventions

- Follow standard Go conventions (`gofmt`, `go vet`).
- All source files carry the Apache 2.0 license header — preserve it in new files.
- The plugin uses **AWS SDK v1** (`github.com/aws/aws-sdk-go`). Do **not**
  introduce AWS SDK v2 (`github.com/aws/aws-sdk-go-v2`) dependencies — that
  belongs to the newer `velero-plugin-for-aws` repository.
- Error handling uses `github.com/pkg/errors` for wrapping.
- Logging uses `github.com/sirupsen/logrus` via Velero's logger interface.

## Before Submitting Changes

1. `make modules` — ensure `go.mod` and `go.sum` are tidy.
2. `make verify-modules` — confirm no uncommitted module changes.
3. `make test` — all unit tests pass.
4. `make ci` — full CI check (combines steps 2 and 3).

## Repository-Specific Notes

- The `replace` directive in `go.mod` pins Velero to the OpenShift fork.
  Do not remove it or switch to the upstream `vmware-tanzu/velero` module.
- This is a **legacy** plugin. The actively developed successor is
  `openshift/velero-plugin-for-aws` (AWS SDK v2). Changes here should be
  limited to bug fixes, CVE remediations, and maintenance.
- Container images are built for multiple architectures via `docker buildx`.
  The `ARCH` variable controls the target (default: `linux-amd64`).
