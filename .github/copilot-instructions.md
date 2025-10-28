## Project overview

This repository contains Go types and helpers for CyVerse job submissions (analyses).
Primary responsibilities:
- Define data models for jobs, steps, containers, volumes, and I/O (see `jobs.go`, `step.go`, `container.go`, `io.go`, `interapps.go`).
- Provide helper methods used by higher-level services to build Condor submissions and prepare container run arguments.

The code is a library consumed by other CyVerse services; it does not contain a main binary. Expect callers to provide configuration via `viper` and JSON payloads that map onto the types here (for example `Job`, `Step`, `Container`).

## Key files to reference
- `go.mod` — project dependencies and Go version (module path `github.com/cyverse-de/model/v8`).
- `jobs.go` — central Job type and many helpers (DirectoryName, OutputDirectory, FinalOutputArguments, resource request helpers).
- `step.go` — Step, StepConfig, params/arguments ordering, and env/exec helpers used when assembling runtime commands.
- `container.go` — Container, Volume, ContainerImage and container-specific helpers (WorkingDirectory, UsesVolumes).
- `io.go` — Input/Output types and porklock argument builders used for iRODS transfers.
- `interapps.go` — Interactive app reverse-proxy and websocket configuration fields.

## Patterns & conventions
- Data-first: types are plain structs with JSON tags. External services submit JSON that is unmarshaled into these structs. Example: `NewFromData(cfg, data []byte)` in `jobs.go`.
- Computed accessors: many derived values are exposed via methods (e.g. `OutputDirectory()`, `WorkingDirectory()`, `Arguments()`); prefer these helpers rather than directly reading fields.
- Parameter ordering: step parameters are ordered by `Order` and returned via `StepConfig.Parameters()`; use that function to construct CLI args.
- Backwards compatibility: some logic checks image names for legacy compatibility (see `Step.IsBackwardsCompatible()`).

## Build / test / dev workflows
- This is a Go module. Use the module-aware Go toolchain. Common commands:

  - Run unit tests quickly: `go test ./...`
  - Tidy dependencies: `go mod tidy`
  - Lint with golangci-lint (same as CI): `golangci-lint run ./...`
  - Optionally also run `go vet` if available.

- The repo contains unit tests (files ending with `_test.go`); tests exercise models and helpers. Tests are the recommended way to validate that model changes don't break callers.

### Linting
- CI uses golangci-lint (v1.64+). Run locally before committing:
  - `golangci-lint run ./...`
- Fixes should be minimal and avoid changing public APIs unless intended. Re-run `go test ./...` after fixes.

## Integration points & expectations
- iRODS: the io helpers generate porklock-style arguments and expect a mounted `/configs/irods-config` inside runtime containers.
- HTCondor: job metadata, log paths, and submission content assume Condor-style executions (see `CondorLogDirectory`, `FinalOutputArguments`).
- Containers: this package describes image names, tags and resource requests; other services read `Container` values to build runtime tasks.

## Examples (copyable snippets)
- Build porklock get args for an input:

```go
args := input.Arguments(username, job.FileMetadata)
// returns []string like: ["get","--user",username,"--source", input.IRODSPath(), "--config", "/configs/irods-config", ...]
```

- Get ordered parameters for a step:

```go
params := step.Config.Parameters()
for _, p := range params { // stable order by p.Order
    // use p.Name and p.Value
}
```

## What to watch for when editing
- Preserve JSON tags and method contracts. Changes to struct fields or JSON tags can break callers.
- Keep computed helper behavior stable (return formats of directory paths, trailing slashes for collections, etc.). Tests in the repo exercise many of these behaviors — update/add tests when changing behavior.
- This module is versioned as v8 in the module path. If releasing a new major version, update module path and coordinate with downstream consumers.

## Where to look next / follow-ups
- If you need runtime examples of how these models are consumed, search in the `cyverse-de` organization for imports of `github.com/cyverse-de/model/v8`.
- Add small focused tests for behavioral changes: a happy path + one edge case (e.g., collection trailing slash handling).

If any section is unclear or you'd like the file to include more examples (test examples, typical JSON payloads, or a small quick-start snippet for local testing), tell me which part and I'll iterate.
