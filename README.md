[中文](README.zh-CN.md) | English

> **SUNSET (2026-09-29):** this repository has zero downstream consumers monorepo-wide
> (verified by grepping every `go.mod`/import across the Lurus monorepo) and its generated
> Go stubs were never a real wire-compatible protobuf build (hand-maintained `.pb.go`
> layout, no `protoimpl`). The single source of truth for `identity.v1` is now
> `2l-svc-platform/proto/proto/identity/v1/identity.proto` inside the platform repo. Do not
> add new consumers here — this repo is pending archival.

# lurus-proto

Cross-language protobuf source of truth for the `identity.v1` gRPC contract used across Lurus internal services.

This repository holds the `.proto` definition and [buf](https://buf.build)-driven code generation for the `IdentityService` API (accounts, wallet, entitlements). It is Go-only today; Dart generation is scaffolded but disabled. **It currently has zero active downstream Go consumers in the monorepo** — the platform service that owns identity keeps its own copy of this same proto in-tree and vendors generated Go types from a separate module (`lurus-proto-go`), not from this repository. See "Relationship to other repos" below before wiring anything to this repo.

## Core contents

- `identity.v1.IdentityService` gRPC definition — account lookup/upsert, entitlements, wallet debit/credit, usage reporting (`proto/identity/v1/identity.proto:11`)
- Generated Go client/server stubs via `buf generate`, checked into `gen/go/` (`gen/go/identity/v1/identity.pb.go`, `gen/go/identity/v1/identity_grpc.pb.go`)
- `buf` lint (`STANDARD` ruleset) and breaking-change detection (`FILE` category) configuration (`buf.yaml:5-10`)
- Dart output is commented out in the generator config, not yet implemented (`buf.gen.yaml:11-13`)

## Quick start

Requires the [`buf`](https://buf.build/docs/installation) CLI and Go 1.25+ (`go.mod:3`).

```bash
# generate all configured languages (currently: Go) into gen/
buf generate

# lint proto style against the STANDARD ruleset
buf lint

# check for breaking changes against the default branch (master)
buf breaking --against '.git#branch=master'

# compile the generated Go package
go build ./...
```

There are no unit tests in this repository (no `_test.go` files) — correctness is enforced by `buf lint`/`buf breaking` plus downstream consumers compiling against the generated types.

## Architecture

```
proto/
  identity/v1/
    identity.proto      # single service definition, source of truth for this repo
gen/
  go/
    identity/v1/
      identity.pb.go        # generated: message types (protoc-gen-go)
      identity_grpc.pb.go   # generated: client/server stubs (protoc-gen-go-grpc)
buf.yaml                 # module + lint/breaking config
buf.gen.yaml              # codegen plugin pins (remote buf.build plugins)
```

`gen/` is checked into git (not gitignored) so consumers can `go get` this module without running `buf` themselves. Do not hand-edit files under `gen/` — they are regenerated wholesale by `buf generate`.

## Configuration

No environment variables or runtime config — this is a schema-only repository, nothing here executes as a service. The only "configuration" is the `buf.gen.yaml` plugin version pins:

| Plugin | Version | Output |
|--------|---------|--------|
| `protocolbuffers/go` | `v1.36.6` | `gen/go` (message types) |
| `grpc/go` | `v1.5.1` | `gen/go` (client/server stubs) |

## Service definition (IdentityService)

| RPC | Purpose |
|-----|---------|
| `GetAccountByZitadelSub` | Look up an account by OIDC `sub` claim. **Deprecated name** — kept for wire back-compat; canonical replacement `GetAccountByIDPSubject` has not been added to this proto yet (`proto/identity/v1/identity.proto:12-16`) |
| `UpsertAccount` | Create/update an account from OIDC login |
| `GetEntitlements` | Product entitlements for an account (platform-side Redis-cached) |
| `GetAccountOverview` | Aggregated read model (account + VIP + wallet + subscription) |
| `ReportUsage` | Record LLM spend for VIP point accumulation |
| `WalletDebit` / `WalletCredit` | Adjust an account's LB wallet balance |

All RPCs are internal service-to-service calls, authenticated via a bearer `INTERNAL_API_KEY` at the transport layer (not enforced by this proto itself — see `proto/identity/v1/identity.proto:9-10`).

Two fields carry vendor-coupling debt that is intentionally *not* cleaned up here: `Account.zitadel_sub` (field 3) and the `GetAccountByZitadelSub` RPC name leak the identity-provider vendor into the wire contract. They are pinned for backward compatibility; do not reuse field number 3 (`proto/identity/v1/identity.proto:41-44`).

## Relationship to other repos — read before you wire a consumer

Per the platform-wide contract registry (`doc/coord/contracts.md`), **this repository is not currently the proto source consumed in production**:

- The canonical, actively-served copy of `identity.proto` lives self-contained inside the platform service repo (`2l-svc-platform/proto/proto/identity/v1/identity.proto`), not here.
- Generated Go types actually imported by consumers (`2b-svc-newhub`, `shared/eventkit`) come from a **separate** module/repo, `lurus-proto-go` (`github.com/hanmahong5-arch/lurus-proto-go`), which is generated from platform's copy — not from this repo's `gen/go/`.
- A grep across the monorepo finds no `.go` file importing this repo's module path (`github.com/hanmahong5-arch/lurus-proto`, without the `-go` suffix).

Before changing anything here or pointing a new consumer at it, check `doc/coord/contracts.md` for the current source-of-truth designation — it may have changed since this README was written, and the two proto copies can drift out of sync in the meantime.

## Development conventions

Full local conventions are in this repo's root convention doc. Summary relevant to this repo: contract changes must be checked against `doc/coord/contracts.md` for consumers before merging, and breaking changes must be listed in `doc/coord/changelog.md` under an "Affects" entry naming every downstream service.

## License

No `LICENSE` file is present in this repository at the time of writing.
