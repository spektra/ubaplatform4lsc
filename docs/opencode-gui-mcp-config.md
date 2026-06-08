# OpenCode GUI — MCP Config Guide

## Critical: Array-style `command`

The OpenCode **GUI** (not just CLI) validates MCP config strictly.  
Local MCP servers **must** use the array form:

```jsonc
// ✅ CORRECT — GUI-compatible
"command": ["/opt/homebrew/bin/npx", "-y", "@modelcontextprotocol/server-github"]
```

```jsonc
// ❌ WRONG — GUI will throw ConfigJsonError
"command": "/opt/homebrew/bin/npx",
"args": ["-y", "@modelcontextprotocol/server-github"]
```

The old `command`-string + `args`-array format triggers:
```
ConfigInvalidError: Expected array, got "npx" at ["mcp"]["..."]["command"]
```

## Every MCP entry must include `"enabled": true`

Missing it triggers:
```
ConfigInvalidError: Missing key at ["mcp"]["..."]["enabled"]
```

## Known-good example

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "context7": {
      "type": "remote",
      "url": "https://mcp.context7.com/mcp",
      "headers": {
        "Authorization": "Bearer ${CONTEXT7_API_KEY}"
      },
      "enabled": true
    },
    "github": {
      "type": "local",
      "command": ["/opt/homebrew/bin/npx", "-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      },
      "enabled": true
    },
    "playwright": {
      "type": "local",
      "command": ["/opt/homebrew/bin/npx", "-y", "@playwright/mcp@latest"],
      "enabled": true
    },
    "supabase": {
      "type": "local",
      "command": ["/opt/homebrew/bin/npx", "-y", "@supabase/mcp-server-supabase@latest", "--read-only"],
      "env": {
        "SUPABASE_URL": "${SUPABASE_URL}",
        "SUPABASE_SERVICE_ROLE_KEY": "${SUPABASE_SERVICE_ROLE_KEY}"
      },
      "enabled": true
    }
  },
  "skills": {
    "paths": [".agents/skills"]
  }
}
```

## Config file locations

OpenCode merges **global** and **project** config. A broken project config can break the GUI even if global config is valid.

| Scope | Path |
|---|---|
| Global | `~/.config/opencode/opencode.jsonc` (also mirrored to `~/.config/opencode/opencode.json`) |
| Project | `./opencode.json` |

## Where to check logs

```
~/.local/share/opencode/log/
```

Quick command to inspect the latest log:

```bash
LATEST="$(ls -t "$HOME/.local/share/opencode/log/"*.log | head -n 1)"
echo "$LATEST"
tail -n 200 "$LATEST"
```

## Troubleshooting checklist

When adding a new MCP server or fixing a "failed to load" error:

1. **Validate JSON** — check braces, commas, trailing commas (JSONC allows them, JSON does not)
2. **`enabled: true`** — every MCP entry must have it
3. **Command is an array** — `"command": ["/path/to/npx", "-y", "package"]`, never `"command": "npx"` + `"args": [...]`
4. **Full npx path** — use `/opt/homebrew/bin/npx` on this iMac (Apple Silicon Homebrew)
5. **Check both configs** — project `opencode.json` overrides/merges with global `~/.config/opencode/opencode.jsonc`
6. **Relaunch the GUI** after config changes — the GUI does not hot-reload config
7. **Check the logs** — `LATEST="$(ls -t ~/.local/share/opencode/log/*.log | head -n1)"; tail -n 200 "$LATEST"`
