# Tasks: Agent Runtime Contract (ARC)

**Input**: Design documents from `specs/001-agent-runtime-contract/`
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/

**Tests**: Not explicitly requested. Test tasks omitted. E2E validation covered in Phase 7 (Polish).

**Organization**: Tasks are grouped by user story. This feature spans two repos: `kagenti-operator` (Go) and `kagenti` (this repo, Python/Helm). Tasks are tagged with their target repo.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **kagenti-operator** (Go): `api/v1alpha1/`, `internal/webhook/`, `internal/controller/`
- **kagenti** (this repo): `kagenti/backend/app/`, `charts/kagenti/`, `docs/`

---

## Phase 1: Setup

**Purpose**: Define the ARC contract formats and CRD extension. No code changes yet, just the artifacts that all implementation tasks reference.

- [ ] T001 Define AGENTS.md template file with YAML frontmatter schema and all body sections per contracts/agents-md-format.md, create as a Go template in kagenti-operator at internal/controller/arc/templates/agents.md.tmpl
- [ ] T002 [P] Define servers.json schema per contracts/servers-json-format.md, create as Go template in kagenti-operator at internal/controller/arc/templates/servers.json.tmpl
- [x] T003 [P] Write ADR document ODH-ADR-AgentOps-0004 at docs/ODH-ADR-AgentOps-0004-agent-runtime-contract.md using prose plugin with reasoning voice

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: CRD extension and webhook changes in kagenti-operator. MUST be complete before any user story can be validated on a cluster.

**Target repo**: kagenti-operator

- [ ] T004 Add `ARC` struct and `ARCTracing`, `ARCOBO` sub-structs to AgentRuntime types in kagenti-operator api/v1alpha1/agentruntime_types.go
- [ ] T005 Add `spec.arc` field to AgentRuntimeSpec struct with JSON/YAML tags and kubebuilder validation markers in kagenti-operator api/v1alpha1/agentruntime_types.go
- [ ] T006 Run controller-gen to regenerate CRD manifests in kagenti-operator config/crd/bases/ and deepcopy in api/v1alpha1/zz_generated.deepcopy.go
- [ ] T007 Update webhook Pod mutator to inject `ARC_CONTRACT_VERSION`, `ARC_AGENT_ID`, `ARC_NAMESPACE`, `ARC_MCP_CONFIG`, `ARC_SKILLS_DIR` env vars in kagenti-operator internal/webhook/pod_mutator.go
- [ ] T008 [P] Update webhook Pod mutator to inject `SERVICE_BINDING_ROOT` env var (default `/bindings`) in kagenti-operator internal/webhook/pod_mutator.go
- [ ] T009 Update webhook Pod mutator to add `/arc/` volume mount (ConfigMap-backed volume for AGENTS.md and MCP config) in kagenti-operator internal/webhook/pod_mutator.go
- [ ] T010 Add conditional `OTEL_EXPORTER_OTLP_ENDPOINT` injection: read `spec.arc.tracing.enabled` from AgentRuntime CR and inject only when true, in kagenti-operator internal/webhook/pod_mutator.go
- [ ] T011 Update RBAC ClusterRole to grant controller permissions for ConfigMap create/update/delete in kagenti-operator config/rbac/role.yaml
- [ ] T012 Update Python backend `_build_agentruntime_manifest()` to include `spec.arc` fields (tracing, obo, contractVersion) in kagenti/backend/app/routers/agents.py

**Checkpoint**: CRD extended, webhook injects ARC env vars and volume mounts. Backend passes ARC config through to AgentRuntime CR.

---

## Phase 3: User Story 1 - Agent Developer Discovers Platform Capabilities (Priority: P1) MVP

**Goal**: Every managed agent Pod has `/arc/AGENTS.md` mounted with the full contract, conforming to the AGENTS.md open standard with YAML frontmatter.

**Independent Test**: Deploy an agent to Kind, exec into Pod, verify `/arc/AGENTS.md` exists with correct contract version, env var table, mount paths, and auth behavior sections.

**Target repo**: kagenti-operator

- [ ] T013 [US1] Create AGENTS.md Go template engine in kagenti-operator internal/controller/arc/agents_md.go: function that takes AgentRuntime CR + cluster config and renders the AGENTS.md template with YAML frontmatter and all body sections
- [ ] T014 [US1] Create ConfigMap builder in kagenti-operator internal/controller/arc/configmap.go: function that wraps rendered AGENTS.md into a `<agent>-arc-contract` ConfigMap with ownerReference to AgentRuntime CR
- [ ] T015 [US1] Add AGENTS.md ConfigMap generation to AgentRuntime controller reconcile loop in kagenti-operator internal/controller/agentruntime_controller.go: on AgentRuntime create/update, generate and apply the contract ConfigMap
- [ ] T016 [US1] Ensure webhook mounts the `<agent>-arc-contract` ConfigMap at `/arc/AGENTS.md` as a subPath volume mount in kagenti-operator internal/webhook/pod_mutator.go
- [ ] T017 [US1] Implement credential redaction in AGENTS.md generation: env var table shows retrieval instructions (file paths under `$SERVICE_BINDING_ROOT`) instead of cleartext values, in kagenti-operator internal/controller/arc/agents_md.go

**Checkpoint**: Agent Pod has `/arc/AGENTS.md` with full contract. AI-native agents can read and self-configure.

---

## Phase 4: User Story 2 - Platform Injects Identity and Proxy Transparently (Priority: P1)

**Goal**: Agent receives plain HTTP inbound, outbound through `HTTP_PROXY`. Sidecar handles mTLS and JWT. AGENTS.md documents the auth behavior.

**Independent Test**: Deploy agent, make outbound HTTP request through proxy, verify sidecar attaches SPIFFE SVID. Verify AGENTS.md auth section is accurate.

- [ ] T018 [US2] Ensure existing webhook injects `HTTP_PROXY`, `HTTPS_PROXY`, `SPIFFE_ENDPOINT_SOCKET` env vars correctly and document the injection in the ARC env var table. Fix any gaps in kagenti-operator internal/webhook/pod_mutator.go
- [ ] T019 [US2] Ensure AGENTS.md auth behavior section accurately documents: inbound (sidecar terminates TLS), outbound (proxy presents SVID), agent-to-agent (mTLS), user-to-agent (JWT validation) in kagenti-operator internal/controller/arc/agents_md.go
- [ ] T020 [P] [US2] Add SPIFFE identity binding under `$SERVICE_BINDING_ROOT/identity/` with type file, mapping existing SPIRE CSI volume outputs (cert, key, bundle) to ServiceBinding convention paths in kagenti-operator internal/webhook/pod_mutator.go

**Checkpoint**: Auth is transparent. Agent code never handles TLS or tokens. AGENTS.md documents the model.

---

## Phase 5: User Story 3 - Controller Generates Target-Specific Contracts (Priority: P2) + User Story 4 - Contract Versioning (Priority: P2)

**Goal**: Controller detects target type and generates different AGENTS.md content. Contract version is injected and consistent.

**Independent Test**: Deploy agents to Deployment and Sandbox targets, verify AGENTS.md content differs. Verify `ARC_CONTRACT_VERSION` matches AGENTS.md frontmatter.

- [ ] T021 [US3] Implement target type detection in kagenti-operator internal/controller/arc/agents_md.go: read `spec.targetRef.kind` and `openshell.ai/managed-by` annotation to determine Deployment/StatefulSet/Sandbox
- [ ] T022 [US3] Create light AGENTS.md variant for Sandbox/OpenShell targets: no proxy sidecar docs, identity noted as managed by OpenShell supervisor, OTEL env vars still documented, in kagenti-operator internal/controller/arc/agents_md.go
- [ ] T023 [P] [US3] Skip sidecar injection for Pods with `openshell.ai/managed-by` annotation in kagenti-operator internal/webhook/pod_mutator.go (verify existing behavior or add guard)
- [ ] T024 [US3] Ensure webhook still injects OTEL env vars for Sandbox targets when `spec.arc.tracing.enabled: true` in kagenti-operator internal/webhook/pod_mutator.go
- [ ] T025 [US4] Implement contract version injection: `ARC_CONTRACT_VERSION` env var value matches AGENTS.md frontmatter `contract_version` field, sourced from `spec.arc.contractVersion` or platform default, in kagenti-operator internal/controller/arc/agents_md.go and internal/webhook/pod_mutator.go
- [ ] T026 [US4] Add unsupported target type rejection: controller sets AgentRuntime status condition to error with clear message for unrecognized `spec.targetRef.kind` values, in kagenti-operator internal/controller/agentruntime_controller.go

**Checkpoint**: Target-specific contracts work. Contract version is consistent across env var and AGENTS.md.

---

## Phase 6: User Story 5 - OBO Delegation (Priority: P3) + User Story 6 - ServiceBinding (Priority: P3) + User Story 7 - AgentRuntime CR Configuration (Priority: P2)

**Goal**: OBO delegation is opt-in via AgentRuntime CR. ServiceBinding conventions adopted. AgentRuntime CR `spec.arc` controls all ARC features.

**Independent Test**: Deploy agent with `spec.arc.obo.mode: token-forwarding`, verify AGENTS.md OBO section changes. Deploy agent with default CR, verify client-credentials auth.

- [ ] T027 [US7] Read `spec.arc.obo.mode` in controller and pass to AGENTS.md template: generate appropriate OBO section (none, token-forwarding, correlation-header) in kagenti-operator internal/controller/arc/agents_md.go
- [ ] T028 [P] [US7] Read `spec.arc.tracing.enabled` in controller and include/exclude tracing section in AGENTS.md template, in kagenti-operator internal/controller/arc/agents_md.go
- [ ] T029 [US5] Document OBO token-forwarding mechanism in AGENTS.md template: agent forwards `Authorization` header, sidecar performs RFC 8693 exchange, in kagenti-operator internal/controller/arc/templates/agents.md.tmpl
- [ ] T030 [P] [US5] Document OBO correlation-header mechanism in AGENTS.md template: agent forwards `X-Kagenti-Request-Id`, sidecar looks up stored user token, in kagenti-operator internal/controller/arc/templates/agents.md.tmpl
- [ ] T031 [US6] Document ServiceBinding credential paths in AGENTS.md template: `$SERVICE_BINDING_ROOT/identity/`, `$SERVICE_BINDING_ROOT/model-access/`, `$SERVICE_BINDING_ROOT/trace-collector/` with retrieval instructions, in kagenti-operator internal/controller/arc/templates/agents.md.tmpl
- [ ] T031a [P] [US6] Create `$SERVICE_BINDING_ROOT/model-access/` binding directory with type file (`model`), projected from model endpoint Secret (url, api-key, provider files) in kagenti-operator internal/webhook/pod_mutator.go
- [ ] T031b [P] [US6] Create `$SERVICE_BINDING_ROOT/trace-collector/` binding directory with type file (`otel`), projected from OTEL collector config (endpoint, sampling files) in kagenti-operator internal/webhook/pod_mutator.go
- [ ] T032 [US7] Implement ConfigMap regeneration on AgentRuntime CR changes: controller watches AgentRuntime updates and regenerates AGENTS.md ConfigMap, in kagenti-operator internal/controller/agentruntime_controller.go

**Checkpoint**: Full ARC feature set configurable via AgentRuntime CR. OBO delegation and ServiceBinding documented in contract.

---

## Phase 7: MCP Server Discovery

**Goal**: `/arc/mcp/servers.json` populated with gateway and individual MCP tool endpoints in standard format.

**Independent Test**: Deploy agent + MCP tool, verify `/arc/mcp/servers.json` contains the tool's Service URL. Delete tool, verify it's removed from servers.json.

- [ ] T033 Implement MCP tool discovery via label selector (`kagenti.io/protocol=mcp`) in agent's namespace, in kagenti-operator internal/controller/arc/mcp_discovery.go
- [ ] T034 Implement servers.json generation: build standard `mcpServers` JSON from discovered tools + configured gateway URL, in kagenti-operator internal/controller/arc/mcp_discovery.go
- [ ] T035 Create `<agent>-arc-mcp` ConfigMap with ownerReference to AgentRuntime CR and mount at `/arc/mcp/servers.json` as subPath, in kagenti-operator internal/controller/arc/configmap.go
- [ ] T036 Add watch for Deployments with `kagenti.io/protocol=mcp` label changes to trigger MCP ConfigMap regeneration, in kagenti-operator internal/controller/agentruntime_controller.go
- [ ] T037 Handle empty state: generate `{"mcpServers": {}}` when no tools or gateway configured, in kagenti-operator internal/controller/arc/mcp_discovery.go

**Checkpoint**: MCP discovery works. Agents read servers.json to find gateway and individual tools.

---

## Phase 8: Helm + Backend Integration

**Goal**: Helm charts and Python backend updated for ARC. Agent creation API passes `spec.arc` to AgentRuntime CR.

**Target repo**: kagenti (this repo)

- [ ] T038 Add ARC default values to charts/kagenti/values.yaml: `arc.contractVersion`, `arc.tracing.enabled`, `arc.obo.mode`, `arc.mcpGateway.url`
- [ ] T039 [P] Update charts/kagenti/templates/agent-namespaces.yaml to include any ARC-related namespace-level configuration
- [ ] T040 Add ARC configuration options to agent creation API: accept `arc` object in create/update agent request body in kagenti/backend/app/routers/agents.py
- [ ] T041 Update `_build_agentruntime_manifest()` to merge ARC config from API request with Helm defaults, in kagenti/backend/app/routers/agents.py

**Checkpoint**: End-to-end flow works: UI/API creates agent with ARC config, operator injects contract.

---

## Phase 9: Polish and Cross-Cutting Concerns

**Purpose**: Documentation, validation, and cleanup

- [ ] T042 [P] Update docs/kagenti-agent-identity-architecture.md to reference ARC and the `/arc/` mount convention
- [ ] T043 [P] Update docs/components.md to document ARC as a platform component
- [ ] T044 Validate quickstart.md scenarios on a Kind cluster: deploy agent, read AGENTS.md, use MCP servers, verify auth
- [ ] T045 Run pre-commit checks on all changed files in this repo
- [ ] T046 Bump kagenti-operator-chart version in charts/kagenti/Chart.yaml to include ARC-enabled operator

---

## Dependencies and Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies, start immediately
- **Phase 2 (Foundational)**: Depends on Phase 1 (templates defined)
- **Phase 3 (US1)**: Depends on Phase 2 (CRD + webhook ready)
- **Phase 4 (US2)**: Depends on Phase 2. Can run in parallel with Phase 3.
- **Phase 5 (US3+US4)**: Depends on Phase 3 (AGENTS.md generation exists)
- **Phase 6 (US5+US6+US7)**: Depends on Phase 3 (AGENTS.md generation exists)
- **Phase 7 (MCP)**: Depends on Phase 2. Can run in parallel with Phases 3-6.
- **Phase 8 (Helm/Backend)**: Depends on Phase 2 (CRD extended)
- **Phase 9 (Polish)**: Depends on all prior phases

### User Story Dependencies

- **US1 (Contract Discovery)**: Independent after Phase 2
- **US2 (Transparent Auth)**: Independent after Phase 2. Parallel with US1.
- **US3 (Target-Specific)**: Depends on US1 (AGENTS.md generation)
- **US4 (Versioning)**: Depends on US1 (AGENTS.md generation). Parallel with US3.
- **US5 (OBO)**: Depends on US1 (AGENTS.md generation). Parallel with US3/US4.
- **US6 (ServiceBinding)**: Depends on US1 (AGENTS.md generation). Parallel with US3/US4/US5.
- **US7 (AgentRuntime CR)**: Depends on US1. Parallel with US3-US6.

### Cross-Repo Sequencing

1. **kagenti-operator**: Phases 2-7 (CRD, webhook, controller, MCP)
2. **kagenti**: Phase 8 (backend + Helm), Phase 9 (docs)
3. Operator must be released and chart version bumped before end-to-end validation

### Parallel Opportunities

```
Phase 2: T007 + T008 can run in parallel (different env var groups)
Phase 3 + Phase 4: US1 and US2 can run in parallel (different concerns)
Phase 5: T021 + T023 can run in parallel (controller vs webhook)
Phase 6: T027 + T028 + T030 can run in parallel (different template sections)
Phase 7: Can run entirely in parallel with Phases 5-6
Phase 8: T038 + T039 can run in parallel (different files)
Phase 9: T042 + T043 can run in parallel (different doc files)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (templates + ADR)
2. Complete Phase 2: Foundational (CRD + webhook)
3. Complete Phase 3: US1 (AGENTS.md generation)
4. **STOP and VALIDATE**: Deploy agent on Kind, exec into Pod, verify `/arc/AGENTS.md`
5. This alone delivers the core value: agents can read their contract

### Incremental Delivery

1. Setup + Foundational -> CRD and webhook ready
2. US1 (AGENTS.md) -> Contract discoverable (MVP)
3. US2 (Auth docs) -> Auth behavior documented in contract
4. US3+US4 (Target-specific + Versioning) -> Multi-target support
5. US5+US6+US7 (OBO + ServiceBinding + CRD config) -> Full feature set
6. MCP Discovery -> Tool endpoints in contract
7. Helm/Backend -> End-to-end UI flow
8. Each increment is independently deployable and testable

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Cross-repo tasks are tagged with target repo in description
- Operator changes (Go) must be released before backend/Helm changes can be validated end-to-end
- Commit after each task or logical group
- Stop at any checkpoint to validate independently
