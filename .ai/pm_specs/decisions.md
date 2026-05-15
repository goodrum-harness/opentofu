# Decisions: OpenTofu v1.13.0-dev — Next Feature Cycle

Source: PM review of `/harness/.ai/reports/pm_handoff.yaml` (generated 2026-05-15)
Analyst: Harness AI / project-manager skill v1.4.1

---

## 2026-05-15: Experimental Engine Promotion Path

**Context:** `internal/engine/` contains a new execution engine gated behind `experimentalRuntimeEnabled()`. The handoff report explicitly flags this as unresolved — GA vs continued experiment in v1.13.0.
**Options:**
- A: Graduate engine to GA in v1.13.0, deprecate old `internal/tofu/` path
- B: Keep gated as opt-in experiment, add explicit flag in CLI (`--engine=new`)
- C: Promote to default with opt-out escape hatch; remove in v1.14.0
**Decision:** Spec written for Option B (explicit CLI flag, staged rollout). Full GA promotion deferred to v1.14.0 unless regression suite passes completely. Rationale: reduces blast radius; preserves community trust as an OSS fork.
**Rationale:** OpenTofu's community contract requires no silent breaking changes. A named flag lets power users test and report issues without impacting the default path.
**Impact:** Spec `spec-engine-graduation.yaml`. Requires new CLI flag, documentation, and migration guide.
**Provisional?** Yes — revisit after integration test suite results. Owner: core-engine team. Deadline: v1.13.0-rc.1 cut.

---

## 2026-05-15: OCI Provider Distribution — Spec Priority

**Context:** OCI artifact distribution (`application/vnd.opentofu.provider`) is integrated via ORAS-Go but has no formal spec artifact. The handoff report flags it as a follow-on suggestion.
**Options:**
- A: Treat as complete; document only
- B: Write a hardening spec (error handling, fallback, auth flows)
- C: Full feature spec covering auth, mirrors, caching, and CI tooling
**Decision:** Option B — hardening spec. The integration is in place; the gaps are in error UX, auth method coverage, and mirror fallback. A full rewrite is out of scope for v1.13.0.
**Rationale:** Community feedback on OCI support will come quickly post-release. Hardening now prevents a wave of bug reports and builds confidence in the distribution model.
**Impact:** Spec `spec-oci-provider-hardening.yaml`.
**Provisional?** No.

---

## 2026-05-15: Encryption Subsystem — External Subprocess Protocol v2

**Context:** The external subprocess key provider uses a versioned header, suggesting planned v2. No spec exists. Handoff notes flag this as PM-worthy.
**Options:**
- A: Leave at v1, document as stable, defer v2 indefinitely
- B: Spec a v2 protocol with structured JSON envelope, capability negotiation, and timeout contract
- C: Deprecate external subprocess in favour of native KMS integrations only
**Decision:** Option B — write a v2 protocol spec for v1.14.0 planning. v1 remains supported and stable in v1.13.0.
**Rationale:** External subprocess is the escape hatch for organizations with HSMs or custom KMS systems. A versioned protocol upgrade enables richer capability negotiation without breaking existing scripts.
**Impact:** Spec `spec-encryption-subprocess-v2.yaml`. No code changes in v1.13.0; spec targets v1.14.0 handoff.
**Provisional?** No.

---

## 2026-05-15: Diagnostic UX — Structured Error Surfacing

**Context:** `tfdiags.Diagnostics` is the primary error accumulator throughout the codebase. Currently, human-readable output and JSON output share the same diagnostic path, but there is no structured severity routing, no error code catalogue, or machine-readable IDs for integration with CI/CD platforms.
**Options:**
- A: Add error code IDs to tfdiags entries (numeric or namespaced string)
- B: New `--output=sarif` mode for IDE/CI integration
- C: Both A and B; error codes are a prerequisite for SARIF
**Decision:** Option C, in two phases: Phase 1 (v1.13.0) — error code catalogue for the top-20 most frequent user-facing diagnostics. Phase 2 (v1.14.0) — SARIF output mode consuming those codes.
**Rationale:** Error codes are the foundation for IDE integrations, documentation links, and automated remediation. This is a quality-of-life win that compounds across the entire toolchain.
**Impact:** Spec `spec-diagnostic-error-codes.yaml` covers Phase 1.
**Provisional?** No.

---

## 2026-05-15: Test Framework — Equivalence Test Coverage Gaps

**Context:** The `testing/equivalence-tests/` framework exists but the handoff scan found no explicit open-work entries flagging coverage gaps. Given 1,867 source files vs 635 test files (ratio ~0.34), coverage of edge cases in the DAG engine and encryption subsystem may be thin.
**Options:**
- A: Accept current coverage; add tests organically
- B: Structured gap analysis + targeted test additions for DAG, encryption, and backend locking paths
**Decision:** Option B — targeted test coverage spec for the three highest-risk subsystems (DAG walk, encryption migration, backend lock races).
**Rationale:** These subsystems have the highest blast radius if broken. Structured coverage closes the feedback loop before v1.13.0 ships.
**Impact:** Spec `spec-test-coverage-hardening.yaml`.
**Provisional?** No.
