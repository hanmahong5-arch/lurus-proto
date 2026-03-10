# lurus-proto

Cross-language protobuf definitions for Lurus service-to-service gRPC APIs.
Generates Go code (Dart code added in Sprint D for Flutter mobile).

## Structure

```
lurus-proto/
  proto/identity/v1/identity.proto   # Account, Wallet, Entitlement gRPC service
  gen/go/identity/v1/                # Generated Go code
  gen/dart/                          # Generated Dart code (Sprint D)
  buf.yaml                          # Buf configuration
  buf.gen.yaml                      # Code generation config
```

## Commands

```bash
# Generate code (requires buf CLI)
buf generate

# Lint protos
buf lint

# Check breaking changes
buf breaking --against .git#branch=main
```

## Usage in Go services

```go
import identityv1 "github.com/hanmahong5-arch/lurus-proto/gen/go/identity/v1"

// Server
identityv1.RegisterIdentityServiceServer(grpcServer, myServer)

// Client
client := identityv1.NewIdentityServiceClient(conn)
```
