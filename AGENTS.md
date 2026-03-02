# AGENTS.md - AI Agent Orientation

This document helps AI coding agents understand the Morpheus-Lumerin-Node codebase.

## What This Is

The **Morpheus-Lumerin-Node** is a consumer node for the Morpheus decentralized AI network. It enables users to:

1. **Stake MOR tokens** to access AI models (stake returns after 7 days)
2. **Route inference requests** through the Lumerin P2P network
3. **Interact via desktop UI** (Electron) or CLI

## Architecture

```
User App / Desktop UI
        ↓
   proxy-router (Go, :8082)
        ↓
   Lumerin Router (:9081)
        ↓
   P2P Network → AI Providers
        ↓
   BASE Blockchain (sessions, staking)
```

## Key Directories

| Directory | Purpose |
|-----------|---------|
| `proxy-router/` | Go HTTP server - routes requests, manages sessions |
| `proxy-router/internal/chatstorage/` | Chat history persistence |
| `proxy-router/internal/handlers/httphandlers/` | HTTP middleware and routes |
| `proxy-router/internal/proxyapi/` | Streaming session management |
| `ui-desktop/` | Electron + React desktop application |
| `cli/` | Command-line chat interface |
| `launcher/` | Bootstrap utility (downloads llama.cpp, TinyLlama) |
| `smart-contracts/` | Foundry contracts for marketplace governance |

## Concurrency Safety

This codebase uses Go maps that require explicit synchronization:

### Protected Maps

| File | Map | Protection |
|------|-----|------------|
| `chatstorage/file_chat_storage.go` | `fileMutexes` | `fileMutexesLock sync.Mutex` |
| `proxyapi/streaming_session_manager.go` | `sessions` | `mu sync.RWMutex` |

### Concurrency Limiter

The HTTP server has a configurable concurrency limiter to prevent resource exhaustion:

- **Default**: 100 concurrent requests
- **Override**: Set `PROXY_MAX_CONCURRENT` environment variable
- **Tested stable**: 500 concurrent on M-series Mac

When the limit is reached, requests receive `503 Service Unavailable` with:
- `X-Concurrency-Limit` header
- `X-Concurrency-Current` header
- `Retry-After: 1` header

## Build & Run

```bash
cd proxy-router

# Build
go build -o proxy-router ./cmd/main.go

# Run (requires .env with wallet config)
./proxy-router
```

### Required Environment Variables

```bash
# Blockchain
ETH_NODE_ADDRESS=https://mainnet.base.org
WALLET_PRIVATE_KEY=<hex without 0x prefix>

# Diamond Contract (Base Mainnet)
DIAMOND_ADDRESS=0x6aBE1d282f72B474E54527D93b979A4f64d3030a
MOR_TOKEN_ADDRESS=0x7431aDa8a591C955a994a21710752EF9b882b8e3

# Optional: Override concurrency limit
PROXY_MAX_CONCURRENT=150
```

## API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `:8082/swagger/` | Swagger UI |
| `:8082/v1/chat/completions` | Chat inference (requires active session) |
| `:8082/blockchain/models` | List available models |
| `:8082/blockchain/sessions` | Open/manage sessions |
| `:8082/health` | Health check |

## Common Gotchas

### 1. MAX_UINT256 Allowance Overflow

**NEVER** approve unlimited MOR tokens. The router calls `increaseAllowance()` which overflows if current allowance is MAX_UINT256.

```bash
# WRONG - breaks all session opens
amount = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff

# RIGHT - use reasonable amount
amount = 10000000000000000000000  # 10,000 MOR
```

### 2. Session Per Model

One session = one model. If you use 5 different models, you need 5 sessions (each staking ~2 MOR).

### 3. Chat History Logic

The `file_chat_storage.go` sets metadata (Title, ModelId, IsLocal) only on the first message. Check for `Messages == nil || len(Messages) == 0`.

## Testing

```bash
cd proxy-router
go test ./...

# Stress test (if you have the script)
# Tests concurrent request handling
```

## Contributing

When modifying this codebase:

1. **Map access** - Always protect with mutex if accessed from multiple goroutines
2. **Error handling** - Check bounds before slice access
3. **Concurrency** - Test with high concurrent load (100+ requests)

## Related Repositories

- [Morpheus Network](https://github.com/MorpheusAIs) - Main Morpheus organization
- [Lumerin Protocol](https://github.com/Lumerin-protocol) - P2P routing layer
