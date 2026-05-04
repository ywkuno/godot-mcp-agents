# Working in this repo

This is **godot-mcp-agents** — a TypeScript MCP server that bridges Claude Code (and any MCP client) to the Godot 4.x engine. It is a fork of `tugcantopaloglu/godot-mcp`, which itself forks `Coding-Solo/godot-mcp`. See [README.md](README.md) for the full pitch.

The repo was originally cloned as `godot-mcp-codex` (and the runtime autoload still uses that legacy name for backward-compat with existing installs like `raid_bloom_starter`). External identity (npm package, GitHub repo, MCP server name) is `godot-mcp-agents`. If you see `Codex` in the autoload constants (`AUTOLOAD_NAME = 'CodexMcpInteractionServer'`, `addons/codex_mcp/`), that is intentional — leave it alone unless you also plan to migrate every consumer project.

## Layout

- `src/index.ts` — single big file. Contains the `GodotServer` class, every tool definition (158 of them, in one array), every handler (one `case` per tool in a switch), and the Godot path detection logic. ~3,500 lines.
- `src/utils.ts` — pure helpers: parameter snake_case mapping, path validation, error response builder, version checks.
- `src/scripts/godot_operations.gd` — runs inside `godot --headless` for non-runtime ops (scene parsing, project settings, resource creation).
- `src/scripts/mcp_interaction_server.gd` — autoload that runs inside the *user's* Godot game, listens on TCP `127.0.0.1:9090`, and executes runtime `game_*` commands.
- `scripts/build.js` — copies the `.gd` files from `src/scripts/` into `build/scripts/` after `tsc`.
- `tests/` — Vitest. Three suites: `utils.test.ts`, `handlers.test.ts`, `tool-definitions.test.ts`. The last one asserts the tool *count* (currently 158) and that every name in the test list also appears in `src/index.ts`.

## When you add or rename a tool

You must update **four** places or tests will fail:

1. The `tools: [...]` array inside `setupToolHandlers()` in `src/index.ts` (the `name`, `description`, `inputSchema`).
2. The `case` block inside the `CallToolRequestSchema` handler in the same method.
3. A handler method (`handleFooBar`) on the `GodotServer` class.
4. `tests/tool-definitions.test.ts` — `ALL_TOOL_NAMES` array and the `expect(ALL_TOOL_NAMES).toHaveLength(N)` assertion.

If the tool should be excluded from the `lite` tier, add it to `TIER_LITE_DROP` near the top of `src/index.ts`. If it should be in `minimal`, add it to `TIER_MINIMAL_KEEP`.

## When you change runtime behavior (`game_*` tools)

The TypeScript handler usually just forwards a JSON command over TCP to the autoload. The actual logic lives in `src/scripts/mcp_interaction_server.gd`. Both sides must agree on the command name and parameter schema. Parameter names cross the boundary in **snake_case** — see `convertCamelToSnakeCase` and `PARAMETER_MAPPINGS` in `utils.ts`.

## Build & test

```bash
npm run build   # tsc + copy gd scripts. Required after editing index.ts.
npm test        # vitest run. Should be green before any commit.
```

The build emits `build/index.js` (the bin entry point per `package.json`) and `build/scripts/*.gd`.

## MCP wiring sanity check

After `npm run build`:

```bash
npx @modelcontextprotocol/inspector build/index.js
```

…opens the inspector so you can list tools and exercise them without a real client. Useful when verifying tier filtering (`GODOT_MCP_TIER=minimal npx @modelcontextprotocol/inspector build/index.js`).

## Things that are easy to break

- **The tool-definition contract test.** It greps `src/index.ts` for `name: '<tool>'` and `case '<tool>':`. If you rename a tool inconsistently, that's the test that catches it.
- **The autoload name.** It's `CodexMcpInteractionServer` here (was `McpInteractionServer` upstream). The constant `AUTOLOAD_NAME` in `src/index.ts` and the `install_runtime_bridge` handler must agree.
- **`name: 'godot-mcp-agents'` in the `Server({...})` constructor** — that's the MCP server identity. Don't change it casually; clients key off it.
- **Tool tier filter.** `resolveAllowedTools` runs each `ListTools` request. The call handler also rejects disabled tools. Both paths read the env vars fresh, so you don't need to reboot the server to test new tier configs in dev.
