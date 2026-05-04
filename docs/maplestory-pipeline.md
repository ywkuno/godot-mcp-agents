# MapleStory asset pipeline recipe

> Pair godot-mcp-agents with [WzBuilder](https://github.com/ywkuno/repacker-aio) (a desktop tool for editing/extracting MapleStory v232 WZ data) to build 2D Godot games using authentic Maple sprites.

This recipe shows the end-to-end agent workflow without wrapping WzBuilder as a second MCP server — the integration is filesystem + CLI shell-out, which keeps the agent's tool budget free for Godot operations.

## Architecture

Two independent processes meet on the filesystem:

```
┌─────────────────────────┐         ┌──────────────────────────┐
│  WzBuilder (separate)   │         │  godot-mcp-agents        │
│  WinForms desktop app + │         │  (this repo, MCP server) │
│  WzBuilder.Tools CLI    │         │                          │
│                         │         │  158 tools — scene ops,  │
│  Sprite-sheet exporter  │         │  scripting, runtime,     │
│  writes:                │         │  asset I/O, debugging    │
│                         │         │                          │
│  out/sprite-sheets/     │         │  Reads from any path     │
│   ├─ mob/100100/        │         │  the agent points it at  │
│   │   ├─ 100100.png     │         │                          │
│   │   ├─ 100100.json    │         │                          │
│   │   └─ 100100.tres    │         │                          │
│   └─ ...                │         │                          │
└──────────┬──────────────┘         └────────────┬─────────────┘
           │      shared filesystem path         │
           └─────────────────────────────────────┘
                            ▲
                            │
                ┌───────────┴────────────┐
                │  Claude Code agent     │
                │  Loads ONLY this MCP   │
                │  Shells out to wzb CLI │
                │  for export steps      │
                └────────────────────────┘
```

Why this shape:

- **Token-cheap.** WzBuilder is not an MCP server; the agent never loads its tool surface. Asset extraction is a single shell-out per export.
- **Composable.** The agent uses godot-mcp-agents tools (e.g. `add_node`, `set_property`, `start_game_for_debugging`) the same way it would for any other Godot project — Maple assets are just another `res://` path.
- **Cacheable.** WzBuilder's per-PNG cache (`out/map-sprites/`) means re-exports are cheap; the bundle is only regenerated when the source WZ changes.

## One-time setup

1. **Install WzBuilder** (separate repo, [github.com/ywkuno/repacker-aio](https://github.com/ywkuno/repacker-aio)). Follow that repo's README to point it at your `Mob.wz` / `Npc.wz` / `Reactor.wz`.

2. **Junction `addons/maple_assets`** inside your Godot project:

   ```powershell
   # Windows (PowerShell, no admin needed):
   New-Item -ItemType Junction `
     -Path "addons/maple_assets" `
     -Target "C:\path\to\repacker-aio\out\sprite-sheets"
   ```

   ```bash
   # macOS / Linux:
   ln -s /path/to/repacker-aio/out/sprite-sheets addons/maple_assets
   ```

   The `.tres` files emitted by WzBuilder reference `res://addons/maple_assets/...`, so the junction makes them drop-in usable with no post-processing.

## Agent recipe

### Step 1 — populate assets (CLI shell-out)

The agent invokes WzBuilder's CLI for either a single entity or a batch. Returns a `_manifest.json` listing what was written:

```bash
# Single entity
dotnet run --project /path/to/repacker-aio/src/WzBuilder.Tools/WzBuilder.Tools.csproj -- \
  sprite-sheet-entity --kind mob --id 100100 \
  --wz-path "/path/to/Mob.wz" --out /path/to/repacker-aio/out/sprite-sheets/

# Bulk (every mob in the WZ, capped at 50)
dotnet run --project /path/to/repacker-aio/src/WzBuilder.Tools/WzBuilder.Tools.csproj -- \
  sprite-sheet-bulk --kind mob \
  --wz-path "/path/to/Mob.wz" --out /path/to/repacker-aio/out/sprite-sheets/ \
  --limit 50
```

After the bulk run, `out/sprite-sheets/mob/_manifest.json` is the discovery surface — listing every `(id, atlas, json, tres)` tuple plus warnings.

### Step 2 — place sprites in the scene (godot-mcp-agents tools)

Once `addons/maple_assets/mob/100100/` exists in the project, the agent drops sprites in directly:

```
# Spawn an animated Snail
add_node parent="root" type="AnimatedSprite2D" name="Snail"
set_property node="root/Snail" property="sprite_frames" \
             value="res://addons/maple_assets/mob/100100/100100.tres"
set_property node="root/Snail" property="animation" value="stand"
set_property node="root/Snail" property="position" value="Vector2(640, 360)"

# Smoke-test in headless runtime
start_game_for_debugging
get_runtime_state
```

The `.tres` already wires per-frame durations from the WZ `delay` field, so the Snail bounces at the correct in-game cadence without any extra tuning.

## Engine-importer contract

WzBuilder's JSON carries a stable `meta.wzbuilder` block agents can rely on:

```json
{
  "meta": {
    "wzbuilder": {
      "schemaVersion": 1,
      "kind": "m",
      "id": "100100",
      "name": "Snail",
      "sourceWz": "/path/to/Mob.wz",
      "sourceImg": "0100100.img",
      "sourceMtimeUtc": "2026-05-04T...",
      "wzVersion": "v232",
      "states": ["stand", "move", "die1", "hit1"]
    }
  }
}
```

- `id` is a string (numeric for mob/npc/reactor; composite for tile/obj/bg).
- `kind` is `m`/`n`/`rea` for entity bundles; `tile`/`obj`/`bg` for map-asset bundles (planned).
- `sourceMtimeUtc` is the cache-invalidation sentinel — when the source WZ's mtime is newer, re-export.

## Don't-do list

- **Don't add a second MCP server for WzBuilder.** It would duplicate the CLI's surface and burn tool budget. The CLI shell-out is enough.
- **Don't loop the agent through "browse every entity in the WZ."** That's a token trap. Either export by known id (`--id 100100`) or by bulk + read the `_manifest.json`.
- **Don't bake atlas paths into scenes by hand.** Let `set_property sprite_frames` carry the `res://` URI; the .tres handles the rest.

## Status

| Asset class | Status | Where |
|---|---|---|
| Mob / NPC / Reactor sprite sheets | ✅ shipped | `wzb sprite-sheet-entity`, `wzb sprite-sheet-bulk` |
| Tile / Object / Background per map | ⏳ planned | follow-up commit on WzBuilder |
| Skill VFX sprite sheets | ⏳ planned | same |
| BGM / SFX | ⏳ Phase 2 | WzBuilder's `docs/PLAN.md` |
| Character composite renderer | ⏳ Phase 4 | same |

For the WzBuilder side of this contract see `docs/GODOT_INTEGRATION.md` in [ywkuno/repacker-aio](https://github.com/ywkuno/repacker-aio).
