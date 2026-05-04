# Godot MCP Agents

[![](https://badge.mcpx.dev?type=server 'MCP Server')](https://modelcontextprotocol.io/introduction)
[![Made with Godot](https://img.shields.io/badge/Made%20with-Godot-478CBF?style=flat&logo=godot%20engine&logoColor=white)](https://godotengine.org)
[![](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white 'TypeScript')](https://www.typescriptlang.org/)
[![](https://img.shields.io/badge/License-MIT-red.svg 'MIT License')](https://opensource.org/licenses/MIT)

A [Model Context Protocol](https://modelcontextprotocol.io/introduction) server that gives Claude Code (and other MCP clients / agent runtimes) full control over the Godot 4.x engine — **158 tools** spanning headless scene operations, runtime introspection, gameplay debugging, asset I/O, project configuration, and live game manipulation over a TCP bridge.

This is **ywkuno's** Godot MCP build, tailored for agentic workflows: tier-aware tool budgeting so the surface fits any agent's tool limit, Claude-Code-friendly install instructions, and a few raid-bloom-specific helpers we needed in our own Godot projects.

## Why this fork?

There are two well-known Godot MCP servers in the wild:

| Project | Approach | Tools | Editor or Runtime | License |
|---------|----------|-------|-------------------|---------|
| [Coding-Solo/godot-mcp](https://github.com/Coding-Solo/godot-mcp) | Headless CLI + runtime TCP bridge | ~20 | Both | MIT |
| [tugcantopaloglu/godot-mcp](https://github.com/tugcantopaloglu/godot-mcp) | Same architecture, large surface area | 149+ | Both | MIT |
| [youichi-uda/godot-mcp-pro](https://github.com/youichi-uda/godot-mcp-pro) | Editor plugin, JSON-RPC over WebSocket :6505 | 172 | Editor (with UndoRedo) | Paid ($5) |
| **ywkuno/godot-mcp-agents** *(this repo)* | Headless CLI + runtime TCP bridge :9090 | **158** | Both | MIT |

What this fork adds on top of the upstream:

- **Tier-aware tool budget** — `GODOT_MCP_TIER=full|lite|minimal` plus `GODOT_MCP_DISABLED_TOOLS` / `GODOT_MCP_ENABLED_TOOLS` env vars so MCP clients with stricter per-server tool limits (Claude Code, Cursor) don't choke on a 158-tool surface.
- **Claude-Code-first install path** — `claude mcp add` snippet and `.mcp.json` example below.
- **Project-specific helpers** — e.g. `raid_bloom_probe` for our Raid Bloom prototype's smoke tests.
- **Cleaner runtime autoload identity** — the autoload is registered as `CodexMcpInteractionServer` (legacy internal name kept for compatibility with existing project installs) so it doesn't collide with other forks running side-by-side.
- **Quieter smoke runs** — benign Godot diagnostics (e.g. Vulkan device noise) are filtered out of `smoke_scene` output.

If you want UndoRedo integration, an editor plugin, or WebSocket transport, use [godot-mcp-pro](https://github.com/youichi-uda/godot-mcp-pro). If you want a slimmer 20-tool baseline, use [Coding-Solo/godot-mcp](https://github.com/Coding-Solo/godot-mcp). This fork sits in the middle: full-coverage runtime control under MIT, with knobs to fit the Claude Code budget.

## Requirements

- [Godot Engine](https://godotengine.org/download) 4.x (4.4+ for UID features)
- [Node.js](https://nodejs.org/) >= 18.0.0
- An MCP-capable client (Claude Code, Claude Desktop, Cursor, Cline, Continue, etc.)

## Install

```bash
git clone https://github.com/ywkuno/godot-mcp-agents.git
cd godot-mcp-agents
npm install
npm run build
```

The compiled entry point ends up at `build/index.js`.

## Configure for Claude Code

The fastest way:

```bash
claude mcp add godot-mcp-agents node /absolute/path/to/godot-mcp-agents/build/index.js
```

Or pin it to a single project via `.mcp.json` at the project root:

```json
{
  "mcpServers": {
    "godot-mcp-agents": {
      "command": "node",
      "args": ["/absolute/path/to/godot-mcp-agents/build/index.js"],
      "env": {
        "GODOT_PATH": "/path/to/godot",
        "GODOT_MCP_TIER": "lite"
      }
    }
  }
}
```

### Configure for other clients

<details>
<summary>Cline (VS Code)</summary>

```json
{
  "mcpServers": {
    "godot-mcp-agents": {
      "command": "node",
      "args": ["/absolute/path/to/godot-mcp-agents/build/index.js"],
      "disabled": false
    }
  }
}
```
</details>

<details>
<summary>Cursor</summary>

Create `.cursor/mcp.json` in your project:

```json
{
  "mcpServers": {
    "godot-mcp-agents": {
      "command": "node",
      "args": ["/absolute/path/to/godot-mcp-agents/build/index.js"]
    }
  }
}
```
</details>

## Tool tiers

158 tools is more than some MCP clients want loaded at once. Pick a tier with the `GODOT_MCP_TIER` env var:

| Tier | Tools exposed | Use when… |
|------|---------------|-----------|
| `full` *(default)* | 158 | You want the whole surface and your client doesn't care. |
| `lite` | ~110 | You want runtime + scene control without advanced 3D, networking, or audio bus plumbing. |
| `minimal` | ~45 | You only need project lifecycle, scene CRUD, file I/O, and core runtime introspection. |

Fine-grained overrides:

- `GODOT_MCP_DISABLED_TOOLS=tool_a,tool_b` — drops named tools *in addition to* the tier filter.
- `GODOT_MCP_ENABLED_TOOLS=tool_a,tool_b` — exposes **only** the named tools (overrides tier and disabled-list).

Disabled tools also get rejected at the call layer with `MethodNotFound`, so a tier-restricted server won't accidentally execute a tool the client tried to call.

## Runtime tools setup (TCP bridge)

The `game_*` tools talk to a running Godot game via TCP `127.0.0.1:9090`. To enable that, the project needs the bridge autoload:

1. Copy `build/scripts/mcp_interaction_server.gd` into your Godot project (e.g. `res://addons/codex_mcp/`).
2. Open **Project → Project Settings → Autoload**.
3. Add the script with the autoload name `CodexMcpInteractionServer`.

Or just call the `install_runtime_bridge` MCP tool against your project path and it will copy the file and register the autoload for you.

## Environment variables

| Variable | Description |
|----------|-------------|
| `GODOT_PATH` | Path to the Godot executable (overrides auto-detection). |
| `DEBUG` | Set to `"true"` for detailed server-side logging on stderr. |
| `GODOT_MCP_TIER` | `full` (default), `lite`, or `minimal`. |
| `GODOT_MCP_DISABLED_TOOLS` | Comma-separated tool names to drop. |
| `GODOT_MCP_ENABLED_TOOLS` | Comma-separated tool names; if set, only these are exposed. |

## Architecture

Two communication channels, just like the upstream:

1. **Headless CLI** — operations that don't need a running game (scene reading/modification, resource creation, project settings) are run as `godot --headless --script godot_operations.gd <op> <json_params>`.
2. **TCP socket** — the `mcp_interaction_server.gd` autoload listens on `127.0.0.1:9090` and processes JSON commands from the TypeScript MCP server while the game is running.

### Source layout

| Path | Description |
|------|-------------|
| `src/index.ts` | MCP server, tool definitions, all handlers, tier filter |
| `src/utils.ts` | Pure utility functions (parameter mapping, validation, error helpers) |
| `src/scripts/godot_operations.gd` | Headless CLI side — all `--headless` operations |
| `src/scripts/mcp_interaction_server.gd` | Runtime side — TCP bridge autoload that handles `game_*` commands |
| `tests/` | Vitest suite (unit tests, tool-definition contract, handler validation) |

## Development

```bash
npm run build       # tsc + copy gd scripts into build/
npm run watch       # tsc --watch
npm test            # vitest run
npm run inspector   # @modelcontextprotocol/inspector against build/index.js
```

## Acknowledgments

This project stands on the shoulders of two prior works:

- **[Coding-Solo/godot-mcp](https://github.com/Coding-Solo/godot-mcp)** by [Solomon Elias](https://github.com/Coding-Solo) — the original. Provided the foundational MCP server architecture, headless GDScript operations system, and TCP runtime bridge. Thank you for the open-source baseline.
- **[tugcantopaloglu/godot-mcp](https://github.com/tugcantopaloglu/godot-mcp)** by [Tugcan Topaloglu](https://github.com/tugcantopaloglu) — the upstream of this fork. Extended the original to ~150 tools spanning networking, 3D/2D rendering, UI controls, audio effects, animation trees, file I/O, runtime code execution, node manipulation, signals, project creation, camera control, and physics.

Differences from those upstreams (Claude Code tier filter, autoload renaming, raid-bloom helpers, doc rewrite) are MIT-licensed under [LICENSE](LICENSE) and authored by [ywkuno](https://github.com/ywkuno).

## License

MIT — see [LICENSE](LICENSE).
