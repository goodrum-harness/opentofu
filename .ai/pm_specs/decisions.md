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

---

# GitHub Issue Prioritization Decisions
# Source: opentofu/opentofu open issues analysis — 2026-05-15
# Method: Ranked by reactions + comment engagement across all open issues

---

## 2026-05-15: `tofu lint` — Spec as standalone v1.13.x feature (#2213, #3999)

**Context:** `tofu lint` (#2213) has 185 reactions — second only to Stacks (#931, 187 reactions). RFC #3999 already exists. This is the single highest-signal actionable feature request with a clear, bounded scope.
**Decision:** Write a full spec for `tofu lint` targeting v1.13.x delivery. This closes the first-party linting gap and is implementable without provider-protocol changes.
**Rationale:** Lint is a DX-tier feature that CI pipelines can adopt immediately. It does not depend on the new runtime, provider protocol, or any blocked architectural work. The existing RFC provides a clear starting point.
**Impact:** Spec `spec-tofu-lint.yaml` created.
**Provisional?** No.

---

## 2026-05-15: Variables in lifecycle attributes — unified lifecycle expressiveness spec (#1329, #2525)

**Context:** Issue #1329 (variables in `prevent_destroy`/`ignore_changes`, 171 reactions) and #2525 (arbitrary expressions in `ignore_changes`, 86 reactions) both stem from the same root limitation: lifecycle blocks are evaluated in a static context that does not allow variable references or computed expressions.
**Decision:** Spec both issues under a single "lifecycle expressiveness" feature. The evaluation-context fix in `internal/configs/` and the plan engine covers both, and solving them together avoids inconsistent semantics.
**Rationale:** These are the top two pure-language enhancement requests. Solving them together is less work than two separate PRs and produces a coherent user story.
**Impact:** Spec `spec-lifecycle-expressiveness.yaml` created covering #1329 and #2525.
**Provisional?** No.

---

## 2026-05-15: Module system — deduplication and cache dir before locking (#1086, #1199, #586)

**Context:** Module deduplication (#1086, 74 reactions) and `TF_MODULE_CACHE_DIR` (#1199, 57 reactions, already accepted) are technically clear and non-breaking. Module locking (#586, 54 reactions, 38 comments) has unresolved design questions around version semantics.
**Decision:** Spec #1086 and #1199 together as `spec-module-system.yaml` for near-term delivery. Module locking deferred pending RFC resolution.
**Rationale:** #1199 is accepted with no implementation. #1086 is a pure performance/correctness improvement. #586 needs stakeholder alignment on version semantics before engineering begins.
**Impact:** Spec `spec-module-system.yaml` created. Module locking tracked as a future spec dependency on RFC #586 resolution.
**Provisional?** No.

---

## 2026-05-15: Stacks concept (#931) — deferred, unblocking analysis required

**Context:** #931 (Implement the stack concept) has the most reactions (187) and comments (45) of any open issue, but is explicitly labelled `blocked` and `pending-decision`. No unblocking path is documented.
**Decision:** Do not spec Stacks for v1.13.x. Instead, schedule a scoping session to identify the specific blockers and produce a decision document for the v1.14.0 planning cycle.
**Rationale:** Stacks is an architectural initiative, not a bounded feature. Building it on the old engine while the new runtime is in progress creates duplicate work. The right moment is after the new runtime reaches beta and the execution model is stable.
**Impact:** No spec created. Tracked as a strategic future initiative in the decisions log.
**Provisional?** Yes — revisit at v1.14.0 planning kickoff.

---

## 2026-05-15: `tofu test` enhancements — JUnit output and mock files as fast-follow (#2501, #1778, #2814)

**Context:** `tofu test` JUnit output (#2501, 31 reactions, accepted, core-team) and mock files (#1778, 8 reactions, accepted) are both accepted with no implementation. Code coverage (#2814, 13 reactions) is requested but not yet accepted.
**Decision:** Bundle JUnit output and mock files into the existing `spec-test-coverage-hardening.yaml` as must-have additions. Code coverage added as nice-to-have. Parameterised test cases (#3004) deferred.
**Rationale:** JUnit output is a core-team accepted item — it should ship. Mock files unlock better provider testing workflows. Bundling into the existing test spec avoids a proliferation of small spec files.
**Impact:** `spec-test-coverage-hardening.yaml` updated to include JUnit output and mock file requirements.
**Provisional?** No.

---

## 2026-05-15: `action` blocks / `action_trigger` lifecycle (#3309) — deferred pending provider-protocol dependency

**Context:** #3309 (80 reactions) proposes `action` blocks and `action_trigger` lifecycle properties. This requires provider-protocol changes (new capability in gRPC protocol v6+).
**Decision:** Defer to v1.14.0. Write a pre-spec note tracking the provider-protocol dependency. Cannot ship without coordination with provider authors across the ecosystem.
**Rationale:** Provider-protocol changes have long lead times — providers need to adopt the new capability before users can consume it. Speccing the language feature without the protocol spec creates an incomplete user experience.
**Impact:** Noted here as deferred. Will require a joint spec covering both language syntax and provider-protocol extension.
**Provisional?** Yes — re-evaluate if a provider-protocol RFC is accepted before v1.14.0 planning.

---

## 2026-05-15: Skip-refresh optimization (#1703) — unfreeze tied to new runtime

**Context:** #1703 (123 reactions, "skip refreshing unchanged resource instances") is marked `frozen`. The new language runtime (internal/engine/) is in development with several sub-issues open (#4057–#4065).
**Decision:** Do not implement selective refresh on the old engine. Unfreeze #1703 and link it as a delivery target for the new runtime once it reaches beta. The new runtime's planning model is the right place for lazy/selective refresh.
**Rationale:** Adding selective refresh to the old engine creates a second code path to maintain. The new runtime should subsume this naturally.
**Impact:** No spec created. Tracked as a new-runtime milestone dependency.
**Provisional?** Yes — revisit when #3414 runtime reaches feature-complete beta.
