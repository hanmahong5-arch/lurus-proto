# lurus-proto (2l-proto)

跨语言 protobuf 定义，承载 Lurus 服务间 gRPC 契约（Go + Dart 双端生成）。Lurus 内部基建。当前 scope `identity/v1`（Account / Wallet / Entitlement）。Tooling buf。Source `proto/identity/v1/identity.proto`。

## Commands

```bash
buf generate                              # 生成所有语言
buf lint
buf breaking --against .git#branch=main   # 破坏性变更检查
```

Go import: `github.com/hanmahong5-arch/lurus-proto/gen/go/identity/v1`

> 真源/细节: 变更前读 `doc/coord/contracts.md` 确认消费者；破坏性变更须在 `doc/coord/changelog.md` "Affects" 列列出所有消费服务。2b-svc-api 经 `shared/lurus-proto-go` 引用，platform 保留本地副本。
