中文 | [English](README.md)

# lurus-proto

Lurus 内部服务间 `identity.v1` gRPC 契约的跨语言 protobuf 源。

本仓存放 `.proto` 定义以及基于 [buf](https://buf.build) 的代码生成配置，服务对象是 `IdentityService`（账户 / 钱包 / 权益）。目前只生成 Go，Dart 生成已搭好插件位但被注释关闭。**本仓目前在 monorepo 里没有任何活跃的 Go 消费者**——真正对外提供服务的 platform 仓库自己维护了一份同样的 proto 副本，下游实际 vendor 的生成 Go 类型来自另一个独立模块（`lurus-proto-go`），而不是本仓的 `gen/go/`。改动前请先看下面「与其他仓的关系」一节。

## 核心内容

- `identity.v1.IdentityService` gRPC 定义——账户查询/创建更新、权益、钱包扣减/充值、用量上报（`proto/identity/v1/identity.proto:11`）
- 通过 `buf generate` 生成并已提交入库的 Go client/server 桩代码（`gen/go/identity/v1/identity.pb.go`、`gen/go/identity/v1/identity_grpc.pb.go`）
- `buf` lint（`STANDARD` 规则集）与破坏性变更检测（`FILE` 分类）配置（`buf.yaml:5-10`）
- Dart 输出在生成器配置里被注释掉，尚未实现（`buf.gen.yaml:11-13`）

## 快速开始

需要 [`buf`](https://buf.build/docs/installation) CLI 与 Go 1.25+（`go.mod:3`）。

```bash
# 生成所有已配置语言(当前仅 Go)到 gen/
buf generate

# 按 STANDARD 规则集做 proto 风格检查
buf lint

# 对比默认分支(master)检测破坏性变更
buf breaking --against '.git#branch=master'

# 编译生成的 Go 包
go build ./...
```

本仓没有单测（无 `_test.go` 文件）——正确性由 `buf lint` / `buf breaking` 加上下游消费者编译期校验来兜底。

## 架构

```
proto/
  identity/v1/
    identity.proto      # 唯一的服务定义, 本仓的真源
gen/
  go/
    identity/v1/
      identity.pb.go        # 生成产物: 消息类型 (protoc-gen-go)
      identity_grpc.pb.go   # 生成产物: client/server 桩 (protoc-gen-go-grpc)
buf.yaml                 # module + lint/breaking 配置
buf.gen.yaml              # codegen 插件版本锁定 (buf.build 远程插件)
```

`gen/` 目录已提交入库（未 gitignore），这样消费者可以直接 `go get` 本模块而不必自己跑 `buf`。**禁止手改 `gen/` 下的文件**——它们由 `buf generate` 整体重新生成。

## 配置

无环境变量、无运行时配置——本仓是纯 schema 仓库，这里的任何东西都不作为服务运行。唯一算得上「配置」的是 `buf.gen.yaml` 里的插件版本锁：

| 插件 | 版本 | 输出 |
|------|------|------|
| `protocolbuffers/go` | `v1.36.6` | `gen/go`（消息类型） |
| `grpc/go` | `v1.5.1` | `gen/go`（client/server 桩） |

## 服务定义（IdentityService）

| RPC | 用途 |
|-----|------|
| `GetAccountByZitadelSub` | 按 OIDC `sub` claim 查账户。**名称已废弃**——为线上兼容保留；规范替代名 `GetAccountByIDPSubject` 尚未加入本 proto（`proto/identity/v1/identity.proto:12-16`） |
| `UpsertAccount` | 依据 OIDC 登录创建/更新账户 |
| `GetEntitlements` | 账户的产品权益（platform 侧 Redis 缓存） |
| `GetAccountOverview` | 聚合读模型（账户 + VIP + 钱包 + 订阅） |
| `ReportUsage` | 上报 LLM 用量用于 VIP 积分累计 |
| `WalletDebit` / `WalletCredit` | 调整账户 LB 钱包余额 |

所有 RPC 都是内部服务间调用，鉴权靠传输层的 bearer `INTERNAL_API_KEY`（本 proto 自身不做强制，见 `proto/identity/v1/identity.proto:9-10`）。

有两处字段是故意保留的供应商耦合债：`Account.zitadel_sub`（字段号 3）与 `GetAccountByZitadelSub` 这个 RPC 名把身份提供商厂商名泄漏进了线上契约，为向后兼容而保留，**禁止复用字段号 3**（`proto/identity/v1/identity.proto:41-44`）。

## 与其他仓的关系——接消费者前先读这节

根据平台级契约注册表（`doc/coord/contracts.md`）：**本仓目前不是生产环境实际消费的 proto 源**。

- 真正在跑、被服务实际使用的 `identity.proto` 副本是 platform 仓自带的一份（`2l-svc-platform/proto/proto/identity/v1/identity.proto`），不是本仓。
- 下游实际导入的生成 Go 类型（`2b-svc-newhub`、`shared/eventkit`）来自**另一个独立**模块/仓库 `lurus-proto-go`（`github.com/hanmahong5-arch/lurus-proto-go`），它是从 platform 那份副本生成的，不是从本仓的 `gen/go/` 生成。
- 对整个 monorepo 做过 grep，没有任何 `.go` 文件导入本仓的模块路径（`github.com/hanmahong5-arch/lurus-proto`，不带 `-go` 后缀）。

改动本仓内容或把新消费者接到本仓之前，先查 `doc/coord/contracts.md` 里当前的真源标注——它可能已经变了，且这两份 proto 副本在此期间可能已经彼此漂移不一致。

## 开发约定

本仓根目录的约定文档里有完整本地约定。与本仓相关的摘要：变更契约前须查 `doc/coord/contracts.md` 确认所有消费者；破坏性变更须在 `doc/coord/changelog.md` 里以 "Affects" 条目列出每一个受影响的下游服务。

## 许可证

截至撰写本文档时，本仓没有 `LICENSE` 文件。
