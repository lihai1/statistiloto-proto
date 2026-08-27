# Contract Requirements

This document defines the requirements for the `statistiloto-proto` contract — the shared gRPC/protobuf definition between the Java BFF and the Go lottery statistics service.

## 1. Contract Requirements

### 1.1 Single Source of Truth
- `lottery.proto` is the **single source of truth** for the service contract.
- No service may duplicate or hand-write message definitions that exist in the proto. All clients and servers must consume generated code.
- The proto must be the only place where RPC names, request/response shapes, field numbers, and REST mappings are defined.

### 1.2 Package and Language Options
- Package: `lottery.v1`.
- Go option: `go_package = "github.com/lihai1/stat-tree-server/pkg/gen;lotteryv1"`.
- Java options: `java_package = "com.statistiloto.lottery.v1"`, `java_multiple_files = true`.
- Imports limited to `google/api/annotations.proto` and `google/protobuf/timestamp.proto` plus vendored Google protos under `third_party/`.

### 1.3 No Duplication
- The Java BFF must not re-declare request/response DTOs that mirror proto messages; it must use the generated Java classes.
- The Go service must not re-declare request/response structs that mirror proto messages; it must use the generated Go types.
- Mapping between proto types and any domain/internal types must happen in an explicit adapter layer, not by redefining the contract.

## 2. RPC Requirements

### 2.1 HealthCheck
- **Signature:** `rpc HealthCheck(HealthCheckRequest) returns (HealthCheckResponse)`
- **REST:** `GET /health`
- **Request:** `HealthCheckRequest` — empty.
- **Response:** `HealthCheckResponse` — `status` (string), `version` (string), `draws_loaded` (int32).
- **Requirement:** Must always return successfully when the service is up. Must report the number of historical draws loaded.

### 2.2 GenerateForm
- **Signature:** `rpc GenerateForm(GenerateFormRequest) returns (GenerateFormResponse)`
- **REST:** `POST /api/generate/form` (`body: "*"`)
- **Request:** `GenerateFormRequest` — `how_many`, `form_type`, `will_be` (repeated), `window` (optional), `strength`.
- **Response:** `GenerateFormResponse` — `forms` (repeated `NumberSet`, each with `numbers` and optional `strong`).
- **Requirement:** Generates `how_many` combinations honoring `will_be` inclusions and the optional `DateWindow`. `strength` controls whether strong-ball statistics are used.

### 2.3 GetStatistics
- **Signature:** `rpc GetStatistics(GetStatisticsRequest) returns (GetStatisticsResponse)`
- **REST:** `POST /api/generate/pares` (`body: "*"`)
- **Request:** `GetStatisticsRequest` — `how_many`, `form_type`, `window` (optional), `strength`.
- **Response:** `GetStatisticsResponse` — `pairs` (repeated `Pair`, each with `numbers` and `count`).
- **Requirement:** Computes the top `how_many` frequent number pairs/groups over the optional date window.

### 2.4 Analyze
- **Signature:** `rpc Analyze(AnalyzeRequest) returns (AnalyzeResponse)`
- **REST:** `POST /api/generate/analyze` (`body: "*"`)
- **Request:** `AnalyzeRequest` — `form` (repeated int32), `window` (optional).
- **Response:** `AnalyzeResponse` — `frequency_groups` (repeated `FrequencyGroup`), `archive_size` (int32).
- **Requirement:** Evaluates the user's selected numbers against historical winning draws. `frequency_groups` contains one entry per group size (1–6), each with `size`, `combos` (C(37, size)), and `entries` sorted by count descending.

### 2.5 Simulate
- **Signature:** `rpc Simulate(SimulateRequest) returns (SimulateResponse)`
- **REST:** `POST /api/generate/simulate` (`body: "*"`)
- **Request:** `SimulateRequest` — `form` (repeated int32, 6/8/10/12 numbers for systematic forms), `strong` (int32, 1–7 or 0 = none), `window` (optional `DateWindow`), `ticket_cost` (double, default 3.0 ILS), `prize_amounts` (repeated double, length 0 or 8 — per-tier prize overrides).
- **Response:** `SimulateResponse` — `draws` (repeated `SimulateDrawResult`, ordered by `draw_date` ascending), `summary` (`SimulateSummary`).
- **Messages:**
  - `SimulateTierHit` — `tier` (int32, 1–8), `hits` (int32), `amount_per_hit` (double), `total` (double).
  - `SimulateDrawResult` — `draw_number`, `draw_date` (Timestamp), `winning_numbers` (repeated int32), `winning_strong` (int32), `tier_hits` (repeated `SimulateTierHit`), `prize_won` (double), `ticket_cost` (double), `used_real_prizes` (bool).
  - `SimulateTierSummary` — `tier` (int32), `label` (string), `total_hits` (int32), `total_amount` (double).
  - `SimulateSummary` — `total_draws`, `total_combinations`, `total_spent`, `total_won`, `net` (all double/int32), `tier_summaries` (repeated `SimulateTierSummary`, always 8 entries), `draws_with_real_prizes` (int32).
- **Requirement:** Backtests a user's ticket against every historical draw in the optional date window. For systematic forms (N > 6), all C(N,6) combinations are played per draw. Prize amounts use scraped per-draw data when available (Go `lottery_results.prize_amounts`), falling back to service defaults or user-supplied `prize_amounts` overrides. `used_real_prizes` / `draws_with_real_prizes` indicate the prize source.

## 3. Backward Compatibility Requirements

- **Field numbers are permanent.** Never reuse or renumber an existing field number.
- **Adding fields:** New fields must use new, unused field numbers and be singular/optional. Existing clients must continue to work.
- **Removing fields:** Mark as `reserved` with the old field number and name; never reuse the number.
- **Changing types:** Changing a field's wire type is a breaking change and is prohibited within `v1`.
- **RPCs:** New RPCs may be added. Existing RPC request/response types may only be extended in a backward-compatible way.
- **Enums:** New values may be appended with new numbers. Existing enum values must not be renumbered or removed. `STRENGTH_UNSPECIFIED = 0` must remain the zero value.
- **Breaking changes** (removing fields, changing types, renumbering) require a new major version (`lottery.v2`) with a new package and new generated-code paths.

## 4. Code Generation Requirements

### 4.1 Go
- Command: `make proto`.
- Output package: `lotteryv1` at `github.com/lihai1/stat-tree-server/pkg/gen`.
- Must generate: gRPC stubs (`*.pb.go`), gRPC-Gateway reverse-proxy (`*.pb.gw.go`), and message types.
- Proto include paths must include both `.` and `third_party/` so `google/api/annotations.proto` and `google/protobuf/timestamp.proto` resolve.

### 4.2 Java
- Command: `gradle generateProto`.
- Output package: `com.statistiloto.lottery.v1`, multiple files (`java_multiple_files = true`).
- Must generate: gRPC stubs and message types consumable by the Java BFF.
- The BFF build must depend on this proto module so generated code is always in sync with `lottery.proto`.

### 4.3 Python
- Command: `python -m grpc_tools.protoc` with `-I.` and `-I third_party`.
- Output: `--python_out` and `--grpc_python_out` into `./gen/python`.
- Used for scripting, testing, and data-analysis tooling.

### 4.4 General
- All three generators must be run from the same `lottery.proto` revision. Generated code must never be hand-edited.
- Regeneration is required after any change to `lottery.proto` before merging.

## 5. REST Mapping Requirements

- All REST mappings are defined via `google.api.http` annotations inside `lottery.proto` — never in a separate config file.
- `HealthCheck` maps to `GET /health`.
- `GenerateForm`, `GetStatistics`, `Analyze`, and `Simulate` map to `POST` endpoints under `/api/generate/...` with `body: "*"`.
- The gRPC-Gateway reverse-proxy must be run by the Go service so the same contract is exposed over REST/JSON for HTTP clients.
- REST paths are part of the public contract; changing a path is a breaking change requiring a major version bump.
- `third_party/google/api/annotations.proto` must be vendored so the annotations resolve without external dependencies.
