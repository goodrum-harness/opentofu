# Codebase Audit Summary
**Repo:** /harness
**Audited:** 2026-05-15T00:00:00Z
**Primary Language:** Go 1.26.3 | **Framework:** None (CLI tool — OpenTofu v1.13.0-dev)
**Module:** github.com/opentofu/opentofu

---

## Architecture at a Glance

| Layer | Key Packages | Nodes |
|-------|-------------|-------|
| Config | `version/`, `internal/experiments/`, `cmd/tofu/` | 3 |
| Domain | `internal/addrs/`, `internal/configs/`, `internal/plans/`, `internal/states/` | 4 |
| Infrastructure | `internal/backend/`, `internal/providers/`, `internal/getproviders/`, `internal/plugin/`, `internal/plugin6/`, `internal/providercache/`, `internal/encryption/` | 11 |
| Application | `internal/tofu/` (Context + graph builders), `internal/dag/` | 3 |
| Presentation | `internal/command/` (Meta + 4 commands), `internal/command/views/`, `internal/command/arguments/` | 8 |
| Shared | `internal/tfdiags/`, `internal/logging/`, `internal/tracing/`, `internal/terminal/`, `internal/collections/` | 1 (shared) |

**Total:** 30 nodes, 25 edges across 10 scan pages.  
**Codebase size:** 1,867 source files, 635 test files.

---

## Entry Points

### CLI Binary
- `cmd/tofu/main.go` — `main()` → `realMain() int` → `os.Exit`
  - Initializes terminal, OTel tracing, logging, credential helpers, service discovery, OCI/module fetchers, provider source, working directory, backend, command dispatch

### CLI Commands (28+ registered)
| Command | Purpose |
|---------|---------|
| `tofu init` | Download modules, initialize backend, install providers, update lock file |
| `tofu validate` | Validate HCL config (no network/state access) |
| `tofu plan` | Produce a diff of planned changes |
| `tofu apply` | Execute a plan against real infrastructure |
| `tofu destroy` | Destroy all managed resources (`apply -destroy`) |
| `tofu fmt` | Reformat .tf/.tofu files via hclwrite |
| `tofu import` | Import existing infrastructure into state |
| `tofu graph` | Output dependency graph in DOT format |
| `tofu test` | Run .tftest.hcl test files |
| `tofu workspace` | Manage named workspaces |
| `tofu state *` | State manipulation subcommands (list, rm, mv, pull, push, show, replace-provider) |

### Remote State Backends
- AWS S3 (`internal/backend/remote-state/s3/`)
- Google Cloud Storage (`internal/backend/remote-state/gcs/`)
- Azure Blob Storage (`internal/backend/remote-state/azure/`)
- HashiCorp Consul, PostgreSQL, HTTP, Kubernetes, Alibaba OSS, Tencent COS, in-memory

### Provider Plugin Protocols
- `internal/plugin/` — gRPC protocol v5 (tfplugin5)
- `internal/plugin6/` — gRPC protocol v6 (tfplugin6)

---

## Key Architectural Patterns

1. **Walk-based execution engine** — Plan, Apply, and Validate all build a DAG (`internal/dag.AcyclicGraph`) and execute nodes concurrently via `graph.Walk()`. Parallelism capped by a counting semaphore (default 10).

2. **Transformer pipeline** — Graph construction via ~25 ordered `GraphTransformer` steps in `BasicGraphBuilder`. Each step is independently testable. Key transformers: `ConfigTransformer`, `ProviderTransformer`, `ReferenceTransformer`, `TargetingTransformer`, `TransitiveReductionTransformer`.

3. **Dual-form pattern** — Every change/state type has decoded (`cty.Value`) and serialized (`DynamicValue`/`*Src` JSON bytes) forms with explicit `Encode`/`Decode` converters. Applied in `plans`, `states`, and provider plugin RPC.

4. **SyncWrapper pattern** — Mutable shared data structures (`State`, `Changes`) expose a `SyncState`/`ChangesSync` wrapper that adds mutex protection and returns deep copies on every read.

5. **Three-context graceful shutdown** — `ctx` (values only) → `stopCtx` (graceful stop) → `cancelCtx` (immediate abort). Applied throughout backend operations and the engine.

6. **Interface sealing via unexported sigil methods** — `addrs` package uses unexported methods to seal `Referenceable`, `Targetable`, `InstanceKey`, etc., preventing external implementations.

7. **Envelope encryption** — `internal/encryption/` wraps any state/plan bytes in a JSON envelope containing ciphertext + key metadata. Pluggable key providers (PBKDF2, AWS KMS, GCP KMS, Azure Vault, OpenBao, External) + methods (AES-GCM, External, Unencrypted).

8. **Static evaluation at config load time** — `StaticEvaluator` in `internal/configs` resolves module sources, `backend` blocks, and provider `for_each` before the runtime walk, using only locals, const vars, `path.*`, and `terraform.workspace`.

9. **Factory function pattern** — Command factories, provider factories, and graph node factories are all lazy. Shared `command.Meta` passed by value to all commands.

10. **Address type system** — `internal/addrs` defines a rich pair hierarchy: static/dynamic (`Module`/`ModuleInstance`, `ConfigResource`/`AbsResource`) and local/absolute. Generic `addrs.Map[K,V]` and `addrs.Set[T]` use the `UniqueKeyer` interface for non-comparable slice types.

---

## Domain Models (Key Types)

| Type | Package | Purpose |
|------|---------|---------|
| `Config` / `Module` | `internal/configs` | Static module tree node; holds all declarations from one .tf directory |
| `Plan` / `Changes` | `internal/plans` | Top-level plan and container for all planned resource/output changes |
| `State` / `ResourceInstanceObject` | `internal/states` | Post-apply known infrastructure state (decoded form) |
| `ResourceInstanceObjectSrc` | `internal/states` | JSON-serialized state (persisted form, with SchemaVersion) |
| `ResourceInstanceChange` | `internal/plans` | Per-resource-instance change: Action (Create/Update/Delete/Forget…), before/after cty.Values |
| `AbsResourceInstance` | `internal/addrs` | Fully-qualified dynamic resource address (ModuleInstance + ResourceInstance + InstanceKey) |
| `Provider` | `internal/addrs` | Fully-qualified provider address (hostname/namespace/type) |
| `DynamicValue` | `internal/plans` | Msgpack-encoded `cty.Value`; nil ≠ cty.NullVal |
| `SyncState` | `internal/states` | RWMutex-wrapped state for concurrent graph-walk access |
| `ChangesSync` | `internal/plans` | Mutex-wrapped changes container for concurrent graph-walk access |

---

## Service / Package Dependency Map

Top connected nodes (by outgoing + incoming edge count):

| Package | Calls | Called By |
|---------|-------|-----------|
| `internal/addrs` | — | `configs`, `plans`, `states`, `tofu/graph_builders`, `providers` (5 inbound) |
| `internal/tofu::Context` | `dag`, `plans`, `states`, `providers::Interface` | `backend/local` |
| `internal/backend::interfaces` | `encryption` | `command::ApplyCommand`, `command::PlanCommand`, `s3`, `gcs`, `azure` |
| `internal/providers::Interface` | — | `tofu::Context`, `plugin::GRPCProvider`, `plugin6::GRPCProvider` |
| `internal/command::Meta` | — | `cmd/tofu::main`, all command structs |
| `internal/tofu::graph_builders` | `addrs`, `configs` | (part of `tofu::Context` flow) |
| `internal/getproviders` | — | `command::InitCommand`, `cmd/tofu::main`, `providercache::Installer` |
| `cmd/tofu::main` | `command::Meta`, `getproviders` | — (binary entry point) |

---

## Provider Plugin Architecture

| Component | Protocol | Purpose |
|-----------|----------|---------|
| `internal/plugin/GRPCProvider` | tfplugin5 (v5) | Core-side gRPC client; translates providers.Interface → protobuf |
| `internal/plugin6/GRPCProvider` | tfplugin6 (v6) | Same structure, v6 proto; registered under key `6` in VersionedPlugins map |
| `internal/providers/Interface` | Go interface | Canonical type used throughout core; alias for `Configured` |
| `internal/getproviders` | HTTP/OCI/FS | Provider discovery, version resolution, package authentication |
| `internal/providercache/Installer` | — | Full install workflow: query → fetch → cache promotion → lock file update |

Supported plugin protocols: `>= 5, <7`. OCI artifact type: `application/vnd.opentofu.provider`.

---

## External Dependencies

| Library | Purpose |
|---------|---------|
| `hashicorp/hcl/v2` | HCL config parsing and evaluation |
| `zclconf/go-cty` | Type system for HCL values; used throughout for cty.Value |
| `hashicorp/go-plugin` | gRPC subprocess plugin host (provider/provisioner process management) |
| `aws/aws-sdk-go-v2` | AWS S3 backend + AWS KMS key provider |
| `Azure/azure-sdk-for-go` | Azure Blob backend + Azure Key Vault key provider |
| `google.golang.org/api` / `cloud.google.com` | GCS backend + GCP KMS key provider |
| `opentofu/registry-address/v2` | Provider FQN parsing and normalization |
| `oras-go` (CNCF) | OCI Distribution v1.1 registry client for OCI provider mirror source |
| `k8s.io/client-go` | Kubernetes remote state backend |
| `opentelemetry-go` | Distributed tracing (OTLP exporter) |
| `go.uber.org/mock/mockgen` | Test mock generation |
| `mitchellh/cli` | CLI framework (command dispatch, help formatting) |
| `hashicorp/go-version` | Semver parsing and constraint matching |
| `spf13/afero` | Filesystem abstraction in configs parser (testability) |

---

## Naming Conventions

| Concept | Convention | Example |
|---------|-----------|---------|
| Interfaces | Named by capability, no `I` prefix | `Backend`, `Provider`, `Encryption`, `Method` |
| Sync wrappers | `*Sync` suffix | `ChangesSync`, `SyncState` |
| Serialized forms | `*Src` suffix | `ResourceInstanceChangeSrc`, `ResourceInstanceObjectSrc` |
| Graph nodes (unexported) | `node*` prefix | `nodeExpandModule`, `nodeCloseProvider` |
| Graph nodes (exported) | `Node*` prefix | `NodeAbstractResource` |
| Graph transformers | `*Transformer` suffix | `ProviderTransformer`, `ReferenceTransformer` |
| Commands | `*Command` suffix | `ApplyCommand`, `PlanCommand` |
| View implementations | `*Human` / `*JSON` / `*Multi` | `OperationHuman`, `OperationJSON` |
| Remote state backends | `internal/backend/remote-state/<name>/` | `s3`, `gcs`, `azure` |

---

## Testing

- **Framework:** stdlib `testing` package + `go test`
- **Test files:** 635 (`*_test.go` pattern), co-located with production code
- **E2E tests:** `internal/command/e2etest/`
- **Equivalence tests:** `testing/equivalence-tests/`
- **Mock generation:** `go.uber.org/mock/mockgen`
- **Test helpers:** `BuildState()`, `BuildChanges()` convenience constructors; `OverrideForTesting()` for experiments
- **Provider mocking:** `testingOverrides` struct in `command.Meta` injects mock providers without plugin discovery

---

## Error Handling

- **Primary type:** `tfdiags.Diagnostics` — accumulating (not early-exit); errors collected and returned together
- **Config parsing:** `hcl.Diagnostics` at the HCL layer, converted to `tfdiags` above it
- **gRPC layer:** `grpcErr()` helper maps gRPC status codes to `tfdiags.Diagnostics`
- **Provider installation:** Typed sentinel errors (`ErrHostNoProviders`, `ErrProviderNotFound`, etc.)
- **Encryption:** `ErrKeyProviderFailure`, `ErrCryptoFailure` typed errors

---

*Full machine-readable graph: `.ai/knowledge-graph.json`*
*Generated by codebase-auditor@1.0.0 — read-only audit, no changes suggested.*
