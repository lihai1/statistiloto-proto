# Statistiloto Proto

Shared gRPC protobuf contract between the Java BFF and the Go lottery statistics service.

## Tech Stack

- **Protocol Buffers v3** (`proto3`) — message and service definitions
- **gRPC** — RPC transport between the Java BFF and the Go service
- **gRPC-Gateway** — REST/JSON mappings via `google.api.http` annotations

## Service Definition

Package: `lottery.v1`

`LotteryService` is the contract between the Java BFF and the Go lottery-stats-server. The Go service owns the tree-based analysis engine (`internal/lottery-tree`) and the `lottery_results` store. It is stateless beyond the historical draws table: no user data, no saved forms.

| RPC | Request | Response | HTTP Mapping | Description |
|-----|---------|----------|--------------|-------------|
| `HealthCheck` | `HealthCheckRequest` | `HealthCheckResponse` | `GET /health` | Returns the health status of the service. |
| `GenerateForm` | `GenerateFormRequest` | `GenerateFormResponse` | `POST /api/generate/form` | Generates lottery number combinations based on historical-draw patterns over an optional date window. |
| `GetStatistics` | `GetStatisticsRequest` | `GetStatisticsResponse` | `POST /api/generate/pares` | Calculates frequent number pairs/groups over a date window. |
| `Analyze` | `AnalyzeRequest` | `AnalyzeResponse` | `POST /api/generate/analyze` | Evaluates user-selected numbers against historical winning draws. |

All `POST` endpoints use `body: "*"` (the entire JSON request body maps to the request message).

## Message Reference

### `DateWindow`
An optional historical-draw window. If both bounds are unset, the service uses the full archive.

| Field | Type | Description |
|-------|------|-------------|
| `from` | `google.protobuf.Timestamp` | Lower bound of the draw window. |
| `to` | `google.protobuf.Timestamp` | Upper bound of the draw window. |

### `GenerateFormRequest`
Mirrors the original `FormCalculations` model.

| Field | Type | Description |
|-------|------|-------------|
| `how_many` | `int32` | Number of combinations to generate. |
| `form_type` | `int32` | Lottery form type/variant (matches original `type` field). |
| `will_be` | `repeated int32` | Numbers the user wants included in every generated form ("willBe"). |
| `window` | `DateWindow` | Optional historical window. Unset = full archive. |
| `strength` | `Strength` | Strength mode for statistics-aware generation. |

### `NumberSet`
One generated combination (regular numbers + optional strong number).

| Field | Type | Description |
|-------|------|-------------|
| `numbers` | `repeated int32` | The regular numbers in the combination. |
| `strong` | `optional int32` | The optional strong/bonus number. |

### `GenerateFormResponse`
A list of generated combinations.

| Field | Type | Description |
|-------|------|-------------|
| `forms` | `repeated NumberSet` | The generated number combinations. |

### `GetStatisticsRequest`
Mirrors the original `statsCalculations` model.

| Field | Type | Description |
|-------|------|-------------|
| `how_many` | `int32` | How many pairs/groups to compute. |
| `form_type` | `int32` | Lottery form type/variant. |
| `window` | `DateWindow` | Optional historical window. |
| `strength` | `Strength` | Strength mode (strong/weak pair statistics). |

### `Pair`
A frequent number group with its occurrence count.

| Field | Type | Description |
|-------|------|-------------|
| `numbers` | `repeated int32` | The numbers in the group. |
| `count` | `int32` | Occurrence count in the archive. |

### `GetStatisticsResponse`
The list of frequent pairs/groups.

| Field | Type | Description |
|-------|------|-------------|
| `pairs` | `repeated Pair` | Frequent number pairs/groups. |
| `total_draws_in_range` | `int32` | Number of historical draws in the requested date window. |

### `AnalyzeRequest`
Mirrors the original `FormAnalyzeCalculations` model.

| Field | Type | Description |
|-------|------|-------------|
| `form` | `repeated int32` | The user's selected numbers to evaluate. |
| `window` | `DateWindow` | Optional historical window. |

### `AnalyzeResponse`
The analysis of the user's form against history.

| Field | Type | Description |
|-------|------|-------------|
| `frequency_groups` | `repeated FrequencyGroup` | Grouped frequency results, one entry per group size (1–6). Each group contains the number combinations of that size and their occurrence counts, mirroring the legacy 2D-array analyze output. |
| `archive_size` | `int32` | Number of historical draws used for the analysis. |

### `FrequencyEntry`
One number combination and its occurrence count.

| Field | Type | Description |
|-------|------|-------------|
| `numbers` | `repeated int32` | The numbers in this combination (length = group size). |
| `count` | `int32` | How many times this combination appeared in the archive. |

### `FrequencyGroup`
Holds all frequency entries for a specific group size.

| Field | Type | Description |
|-------|------|-------------|
| `size` | `int32` | Group size: 1 = single number, 2 = pair, 3 = triple, etc. |
| `combos` | `int32` | Total possible combinations for this group size (C(37, size)). |
| `entries` | `repeated FrequencyEntry` | Frequency entries in this group, sorted by count descending. |

### `HealthCheckRequest`
Empty request for the health check. No fields.

### `HealthCheckResponse`
Contains the health status of the service.

| Field | Type | Description |
|-------|------|-------------|
| `status` | `string` | Service status string. |
| `version` | `string` | Service version. |
| `draws_loaded` | `int32` | Number of historical draws loaded. |

## Enum Reference

### `Strength`
Selects whether strong-ball statistics are used.

| Value | Number | Description |
|-------|--------|-------------|
| `STRENGTH_UNSPECIFIED` | 0 | Unspecified / default. |
| `WEAK` | 1 | Weak-ball statistics. |
| `STRONG` | 2 | Strong-ball statistics. |

## Code Generation

### Go
```bash
make proto
```
Generates Go gRPC stubs and gRPC-Gateway reverse-proxy into `pkg/gen` (package `lotteryv1`), as specified by:
```
option go_package = "github.com/lihai1/stat-tree-server/pkg/gen;lotteryv1";
```

### Java
```bash
gradle generateProto
```
Generates Java gRPC stubs into package `com.statistiloto.lottery.v1` (multiple files), as specified by:
```
option java_package = "com.statistiloto.lottery.v1";
option java_multiple_files = true;
```

### Python
```bash
python -m grpc_tools.protoc \
  -I. \
  -I third_party \
  --python_out=./gen/python \
  --grpc_python_out=./gen/python \
  lottery.proto
```
Generates Python gRPC stubs into `./gen/python`.

## Project Structure

```
proto/
├── README.md                  # This file
├── docs/
│   ├── REQUIREMENTS.md        # Contract and compatibility requirements
│   └── FLOWS.md               # Mermaid diagrams for proto and call flows
├── lottery.proto              # The single source of truth (service + messages)
└── third_party/               # Vendored Google protos
    ├── google/
    │   ├── api/
    │   │   ├── annotations.proto   # gRPC-Gateway REST annotations
    │   │   └── httpbody.proto
    │   └── protobuf/
    │       ├── any.proto
    │       ├── api.proto
    │       ├── compiler/plugin.proto
    │       ├── descriptor.proto
    │       ├── duration.proto
    │       ├── empty.proto
    │       ├── field_mask.proto
    │       ├── source_context.proto
    │       ├── struct.proto
    │       ├── timestamp.proto
    │       ├── type.proto
    │       └── wrappers.proto
```

## Usage

### Go Lottery Service (Server)
The Go service implements `LotteryService` and serves gRPC. It also runs the gRPC-Gateway reverse-proxy so the same contract is exposed over REST/JSON. Generated stubs live in `pkg/gen` (package `lotteryv1`).

### Java BFF (Client)
The Java BFF consumes the generated Java stubs (`com.statistiloto.lottery.v1`) as a gRPC client, calling the Go service for all lottery operations (form generation, statistics, analysis). The BFF handles HTTP-facing concerns (auth, session, presentation) and delegates business logic to the Go service via this contract.

### Python (Optional Client)
Python stubs can be generated for scripting, testing, or data-analysis tooling that needs to call the Go service directly over gRPC.

## Versioning Policy

- The proto package is `lottery.v1`. Breaking changes require a new major version (`lottery.v2`) with a new package and new generated-code paths.
- Within `v1`, changes must be backward compatible:
  - New fields must use new (unused) field numbers and be optional/singular.
  - Existing field numbers must never be reused or re-typed.
  - New RPCs may be added; existing RPC signatures must not change.
  - New enum values may be appended; existing values must not be renumbered or removed.
- Field numbers are a permanent part of the wire format and must never be recycled.

## Documentation Links

- [Contract Requirements](docs/REQUIREMENTS.md)
- [Proto and Call Flows](docs/FLOWS.md)
- [Protocol Buffers Language Guide (proto3)](https://protobuf.dev/programming-guides/proto3/)
- [gRPC-Gateway Documentation](https://grpc-ecosystem.github.io/grpc-gateway/)
