# flespi Skills

AI agent skills for the [flespi](https://flespi.com) IoT and telematics cloud platform. Provides comprehensive reference for the flespi REST API, MQTT broker, device management, and analytics.

## Installation

### Claude Code (plugin)

```bash
/plugin marketplace add flespi-software/skills
/plugin install flespi@flespi-software-skills
```

Then use the skill:
```
/flespi:flespi how do I get device messages?
```

### VS Code / GitHub Copilot

Add to your `.vscode/mcp.json`:
```json
{
  "inputs": [
    {
      "id": "flespiToken",
      "type": "promptString",
      "description": "Flespi Token",
      "password": true
    }
  ],
  "servers": {
    "flespi-develop": {
      "type": "http",
      "url": "https://flespi.io/ai/mcp/develop",
      "headers": {
        "Authorization": "FlespiToken ${input:flespiToken}"
      }
    }
  }
}
```

For knowledge context, also copy `.github/copilot-instructions.md` to your project's `.github/` directory.

### Cursor

Add to your `.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "flespi-develop": {
      "url": "https://flespi.io/ai/mcp/develop",
      "headers": {
        "Authorization": "FlespiToken ${FLESPI_TOKEN}"
      }
    }
  }
}
```

For knowledge context, also copy `.cursorrules` to your project root.

### Windsurf

Add to your `.windsurf/mcp.json`:
```json
{
  "mcpServers": {
    "flespi-develop": {
      "serverUrl": "https://flespi.io/ai/mcp/develop",
      "headers": {
        "Authorization": "FlespiToken ${FLESPI_TOKEN}"
      }
    }
  }
}
```

For knowledge context, also copy `.windsurfrules` to your project root.

### OpenCode

Copy `AGENTS.md` to your project root, or add to your `opencode.json`:
```json
{
  "instructions": ["path/to/skills/flespi/SKILL.md"]
}
```

### Manual (any agent)

Copy `skills/flespi/SKILL.md` into your agent's context or instructions file. The content is plain Markdown and works with any LLM-based coding assistant.

## What's Included

- **Platform concepts** — data flow, core entities (channels, devices, streams, calculators, plugins, geofences, webhooks, tokens)
- **REST API reference** — authentication, CRUD, selectors, pagination, response format, error codes, calculate method
- **MQTT broker reference** — topics, categories, filtering, shared/sticky subscriptions, QoS
- **Expression syntax** — operators, functions, parameter access, filtering, templating
- **Common patterns** — device lookup, message retrieval, analytics, stream health, command sending

## License

MIT
