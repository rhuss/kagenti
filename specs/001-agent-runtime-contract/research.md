# Research: Agent Runtime Contract (ARC)

**Date**: 2026-05-13
**Spec**: [spec.md](spec.md)

## R1: Webhook Injection Architecture

**Decision**: Extend the existing webhook in `kagenti-operator` (Go) to inject ARC environment variables and `/arc/` volume mounts alongside the current AuthBridge sidecar stack.

**Rationale**: The webhook already injects 5 containers and multiple ConfigMap mounts. ARC env vars and the `/arc/` volume mount are additions to the same mutation path. No new webhook needed.

**Current state**:
- Webhook lives in `kagenti-operator` repo (Go, separate from this repo)
- Triggered by `kagenti.io/type: agent` label on Pods
- Opt-out via `kagenti.io/inject: disabled` label
- Injects: proxy-init, sign-agentcard, envoy-proxy, spiffe-helper, client-registration
- Per-sidecar control via labels: `KAGENTI_ENVOY_PROXY_INJECT_LABEL`, `KAGENTI_SPIFFE_HELPER_INJECT_LABEL`, `KAGENTI_CLIENT_REGISTRATION_INJECT_LABEL`

**ARC changes needed in kagenti-operator**:
- Add `ARC_*` env var injection to the webhook mutation
- Add `/arc/` volume mount (ConfigMap-backed) to the webhook mutation
- Read ARC config from AgentRuntime CR `spec.arc` to determine which vars/mounts to inject
- Conditionally inject `OTEL_EXPORTER_OTLP_ENDPOINT` based on `spec.arc.tracing.enabled`

## R2: AgentRuntime CRD Extension

**Decision**: Add `spec.arc` section to the existing AgentRuntime CRD in `kagenti-operator`.

**Rationale**: AgentRuntime CR already serves as the integration point between the backend and the webhook. Adding `spec.arc` keeps the single-CR-per-agent model and avoids a new CRD.

**Current CRD**:
- API: `agent.kagenti.dev/v1alpha1`
- Fields: `spec.type` (agent/tool), `spec.targetRef` (Deployment/StatefulSet/Sandbox)
- Defined in: `kagenti-operator` repo (Go types)
- Created by: Python backend (`_build_agentruntime_manifest()` in `agents.py:2448`)

**New fields**:
```yaml
spec:
  arc:
    contractVersion: "v1alpha1"    # optional override
    tracing:
      enabled: true/false          # default: false
    obo:
      mode: "none|token-forwarding|correlation-header"  # default: none
```

**Cross-repo coordination**: The Python backend in this repo must be updated to include `spec.arc` when building AgentRuntime manifests.

## R3: AGENTS.md Generation

**Decision**: Generate AGENTS.md in the Go operator controller (not Python backend), mounted via ConfigMap.

**Rationale**: The operator controller already reconciles AgentRuntime CRs and has access to cluster-wide state (MCPServerRegistration CRs, namespace config). The Python backend creates the AgentRuntime CR but doesn't manage Pod-level resources after creation. AGENTS.md must be regenerated on config changes, which is a controller reconciliation concern.

**Current ConfigMap generation pattern**:
- Python backend creates 4 AuthBridge ConfigMaps per namespace (`_ensure_authbridge_configmaps()`)
- Helm charts pre-deploy template ConfigMaps in `kagenti-system`
- Operator copies templates to new agent namespaces

**ARC ConfigMap pattern**:
- Controller generates `<agent>-arc-contract` ConfigMap per AgentRuntime CR
- ConfigMap contains `AGENTS.md` key (contract file) and is mounted at `/arc/AGENTS.md`
- Controller generates `<agent>-arc-mcp` ConfigMap with `servers.json` key, mounted at `/arc/mcp/servers.json`
- Skills init container populates `/arc/skills/` (emptyDir shared volume)
- Controller watches MCPServerRegistration changes and regenerates MCP ConfigMaps

## R4: MCP Server Discovery

**Decision**: Use `/arc/mcp/servers.json` in standard `mcpServers` format. Controller populates from MCP Gateway URL (platform config) plus individual MCP tool Deployments discovered via labels.

**Rationale**: Current MCP tools are plain Deployments with labels (`kagenti.io/type=tool`, `kagenti.io/protocol=mcp`). No MCPServerRegistration CRDs exist. The controller can discover tools via label selector and build the servers.json from Services.

**Current MCP tool state**:
- Tools deployed as Deployments with `kagenti.io/protocol=mcp` label
- Services created with `<tool-name>-service` naming
- Transport: streamable HTTP
- No MCPServer CRDs in codebase (comment in tools.py: "This replaces the MCPServer CRD approach")

**servers.json generation**:
```json
{
  "mcpServers": {
    "gateway": {"url": "http://mcp-gateway-broker.mcp-system:8080/mcp"},
    "weather-tool": {"url": "http://weather-tool-service.team1:8080/mcp"}
  }
}
```

## R5: ServiceBinding Integration

**Decision**: Implement ServiceBinding conventions from scratch. No existing integration to extend.

**Rationale**: Zero ServiceBinding references in current codebase. Credentials are currently via Secrets/ConfigMaps mounted directly. The ARC introduces `$SERVICE_BINDING_ROOT` as a new pattern. For v1alpha1, project the SPIFFE identity binding only; model-access and trace-collector bindings are future work.

**v1alpha1 scope**:
- `$SERVICE_BINDING_ROOT/identity/` with SPIFFE SVID (cert, key, bundle, type file)
- Mount paths align with existing SPIRE CSI driver output
- `model-access/` and `trace-collector/` as stretch goals (credentials currently in Secrets)

## R6: Cross-Repo Implementation Strategy

**Decision**: Implementation spans two repos. Changes must be coordinated.

| Change | Repo | Language |
|--------|------|----------|
| AgentRuntime CRD `spec.arc` fields | kagenti-operator | Go |
| Webhook ARC env var injection | kagenti-operator | Go |
| Webhook `/arc/` volume mount injection | kagenti-operator | Go |
| Controller AGENTS.md ConfigMap generation | kagenti-operator | Go |
| Controller servers.json ConfigMap generation | kagenti-operator | Go |
| Controller MCP tool discovery (label selector) | kagenti-operator | Go |
| Backend `_build_agentruntime_manifest()` update | kagenti (this repo) | Python |
| Helm chart values for ARC defaults | kagenti (this repo) | YAML |
| AGENTS.md template/format definition | kagenti (this repo) | Markdown |
| ADR document | kagenti (this repo) | Markdown |

**Sequencing**: Operator changes first (CRD, webhook, controller), then backend/Helm changes to use the new CRD fields.

## Alternatives Considered

### AGENTS.md generation in Python backend vs Go operator
- **Python backend**: Simpler to prototype, but backend doesn't manage Pod lifecycle after creation. Can't react to config changes (no watch/reconcile loop).
- **Go operator** (chosen): Has reconciliation loop, watches CRs, can regenerate ConfigMaps on changes. Proper Kubernetes controller pattern.

### New CRD for ARC config vs extending AgentRuntime
- **New CRD (ARCConfig)**: Clean separation, but adds a second CR per agent. More complexity for users and the controller.
- **Extend AgentRuntime** (chosen): Single CR per agent. ARC config lives alongside targetRef. Simpler UX and implementation.

### MCP discovery via CRD vs label-based
- **MCPServerRegistration CRD**: Clean API surface, but no existing CRDs for MCP tools. Would need to introduce a new CRD.
- **Label-based discovery** (chosen): Tools already have `kagenti.io/protocol=mcp` labels. Controller can discover via label selector. No new CRD needed for v1alpha1.
