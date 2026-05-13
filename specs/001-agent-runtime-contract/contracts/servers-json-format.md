# Contract: /arc/mcp/servers.json Format

**Version**: v1alpha1
**Compatible with**: The `mcpServers` de facto standard used by Claude Code, Claude Desktop, Cursor, Windsurf, Gemini, LM Studio, and ToolHive

## Format

JSON file with a top-level `mcpServers` object. Each key is a server name, value is an object with at minimum a `url` field.

## Schema

```json
{
  "mcpServers": {
    "<server-name>": {
      "url": "<endpoint-url>"
    }
  }
}
```

## Examples

### Gateway + individual servers

```json
{
  "mcpServers": {
    "gateway": {
      "url": "http://mcp-gateway-broker.mcp-system:8080/mcp"
    },
    "weather-tool": {
      "url": "http://weather-tool-service.team1:8080/mcp"
    },
    "github-tool": {
      "url": "http://github-tool-service.team1:8080/mcp"
    }
  }
}
```

### Gateway only

```json
{
  "mcpServers": {
    "gateway": {
      "url": "http://mcp-gateway-broker.mcp-system:8080/mcp"
    }
  }
}
```

### No servers configured

```json
{
  "mcpServers": {}
}
```

## Generation Rules

1. **MCP Gateway**: If an MCP Gateway is deployed (configurable via Helm values), add a `gateway` entry with the gateway Service URL.
2. **Individual MCP tools**: Controller discovers Deployments with label `kagenti.io/protocol=mcp` in the agent's namespace. For each, add an entry keyed by the tool name with the Service URL.
3. **Empty state**: If no gateway and no tools, the file MUST contain `{"mcpServers": {}}`.
4. **Regeneration**: Controller regenerates when MCP tool Deployments are created/deleted in the namespace or when gateway config changes.

## URL Convention

- Gateway: `http://<gateway-service>.<gateway-namespace>:<port>/mcp`
- Individual tool: `http://<tool-service>.<agent-namespace>:<port>/mcp`
- Transport: Streamable HTTP (default for all Kagenti MCP tools)
