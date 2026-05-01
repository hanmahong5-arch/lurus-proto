# lurus-proto (2l-proto)

跨语言 protobuf 定义，承载 Lurus 服务间 gRPC 契约。Go + Dart (Flutter) 双端生成。

- Current scope: `identity/v1` (Account / Wallet / Entitlement)
- Tooling: buf
- Consumers: 2b-svc-api (via `shared/lurus-proto-go`), 2l-svc-platform (provider), 2c-app-lutu (Dart)

## Directory

```
proto/identity/v1/identity.proto   # Source of truth
gen/go/identity/v1/                # Generated Go
gen/dart/                          # Generated Dart (Sprint D)
buf.yaml
buf.gen.yaml
```

## Commands

```bash
buf generate                              # Generate all languages
buf lint                                  # Proto lint
buf breaking --against .git#branch=main   # Breaking-change check
```

## Usage (Go)

```go
import identityv1 "github.com/hanmahong5-arch/lurus-proto/gen/go/identity/v1"

// Server
identityv1.RegisterIdentityServiceServer(grpcServer, myServer)

// Client
client := identityv1.NewIdentityServiceClient(conn)
```

## Cross-service Rule

- Proto 变更前读 `doc/coord/contracts.md` 确认消费者。
- 破坏性变更必须在 `doc/coord/changelog.md` "Affects" 列列出所有消费服务。
- 2b-svc-api 通过 `shared/lurus-proto-go` 引用；platform 保留本地副本。
