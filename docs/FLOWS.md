# Flows

Mermaid diagrams for the proto contract lifecycle and runtime call flows.

## 1. Proto Change Flow

How a change to the contract propagates through all consumers. Every change to `lottery.proto` must regenerate all language stubs and test every service before merging.

```mermaid
flowchart TD
    A["Edit lottery.proto"] --> B{"Backward compatible?"}
    B -- No --> B1["Bump to new major version\n(lottery.v2, new package)"]
    B1 --> C
    B -- Yes --> C["Regenerate Go stubs\n(make proto)"]
    C --> D["Regenerate Java stubs\n(gradle generateProto)"]
    D --> E["Regenerate Python stubs\n(grpc_tools.protoc)"]
    E --> F["Update Go service implementation"]
    F --> G["Update Java BFF client calls"]
    G --> H["Test Go service"]
    H --> I["Test Java BFF"]
    I --> J["Test Python scripts (if affected)"]
    J --> K["Merge"]
```

## 2. gRPC Call Flow

The Java BFF calls the Go lottery service using the generated gRPC stubs. The proto contract is the only thing both sides share.

```mermaid
flowchart LR
    Client["HTTP Client / Browser"] --> BFF["Java BFF\n(gRPC client)"]
    BFF -->|"gRPC call\n(LotteryService stubs)"| GoService["Go Lottery Service\n(gRPC server)"]
    GoService --> Engine["internal/lottery-tree\n(analysis engine)"]
    Engine --> Store["lottery_results store\n(historical draws)"]
    Store --> Engine
    Engine --> GoService
    GoService -->|"gRPC response\n(proto message)"| BFF
    BFF -->|"HTTP/JSON response"| Client
```

### RPCs over gRPC
| RPC | Direction | Purpose |
|-----|-----------|---------|
| `HealthCheck` | BFF → Go | Liveness and draw count |
| `GenerateForm` | BFF → Go | Generate number combinations |
| `GetStatistics` | BFF → Go | Frequent pairs/groups |
| `Analyze` | BFF → Go | Evaluate user form vs. history |

## 3. REST Gateway Flow

The Go service runs the gRPC-Gateway reverse-proxy, translating HTTP/JSON requests into gRPC calls using the `google.api.http` annotations in `lottery.proto`. This lets HTTP clients use the same contract without a gRPC client.

```mermaid
flowchart LR
    HTTP["HTTP Client"] -->|"POST /api/generate/form\nPOST /api/generate/pares\nPOST /api/generate/analyze\nGET /health"| Gateway["gRPC-Gateway\n(reverse-proxy, Go)"]
    Gateway -->|"translate JSON → proto\n→ gRPC call"| GoService["Go Lottery Service\n(gRPC server)"]
    GoService -->|"gRPC response\n(proto message)"| Gateway
    Gateway -->|"translate proto → JSON\n→ HTTP response"| HTTP
```

### REST Mappings
| HTTP Method | Path | RPC | Body |
|-------------|------|-----|------|
| `GET` | `/health` | `HealthCheck` | — |
| `POST` | `/api/generate/form` | `GenerateForm` | `*` |
| `POST` | `/api/generate/pares` | `GetStatistics` | `*` |
| `POST` | `/api/generate/analyze` | `Analyze` | `*` |
