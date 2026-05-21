---
name: gopeak-godot-mcp
description: >-
  Use the GoPeak Godot MCP server (`gopeak`) to drive the Godot editor from
  Claude Code: scene/node edits, script creation, resource authoring, runtime
  inspection of a running game, breakpoint debugging, animation, tilemap,
  asset-store search, screenshot/input testing.

  Trigger this skill whenever the working directory is a Godot project with
  `gopeak` configured in `.mcp.json` and the user asks to: edit a scene/node,
  create or modify a script through the editor, inspect a running game,
  set breakpoints, work with animations or tilemaps, capture screenshots, or
  any task that would otherwise require hand-editing `.tscn`/`.tres` files.

  Common triggers:
  - "add a node to the X scene"
  - "rename this script / move this resource"
  - "what's in the running game's tree?"
  - "set a breakpoint in battle_resolver"
  - "wire up this AnimationTree"
  - "screenshot the title screen"
  - any request involving Godot scenes/resources where the editor is open
---

# gopeak-godot-mcp

Use the GoPeak Godot MCP (`gopeak`) as the primary channel for editor-touching work in a Godot project. Prefer MCP tools over hand-editing `.tscn`/`.tres` whenever a matching tool exists — the bridge keeps UIDs, references, and the live editor in sync.

## When this skill applies

- The project has `mcpServers.godot` (or similar) in `.mcp.json` pointing at `gopeak` / `npx -y gopeak`.
- The Godot editor is open with `godot_mcp_editor` plugin enabled (bridge listens on port 6505 by default).
- The task involves scenes, nodes, scripts (creation/rename/move), resources, animations, tilemaps, runtime state, debugging, or editor automation.

This skill does **not** apply to plain `.gd`/`.json`/data-file edits — those still use `Read`/`Edit`/`Write`. It also does not apply when the editor is closed and the task is just running headless tests or batch builds — use `Bash` for those.

## Profile model

GoPeak upstream exposes tools in a **compact** profile by default:

- **33 core tools** always visible (project, scene basics, script CRUD, runtime status, LSP diagnostics, screenshot, key/mouse injection, etc.).
- **78 additional tools** grouped into **22 dynamic groups** that activate on demand.

Activating only what you need keeps the tool list and token cost low.

### Local fork (`~/Developer/MCP/Gopeak-godot-mcp`) — required for Claude Code

Upstream gopeak ships only 33 compact aliases and defaults `tools/list` page size to 33. Claude Code consumes only the first `tools/list` page when populating its deferred-tool registry, so the 78 dynamic-group tools are unreachable upstream — even with `GOPEAK_TOOL_PROFILE=full` and groups activated.

A local fork at `~/Developer/MCP/Gopeak-godot-mcp` patches both:
1. Adds dotted compact aliases for all 78 dynamic tools (`dap.set_breakpoint`, `lsp.completions`, `runtime.inspect_tree`, `testing.screenshot`, `animation.create`, …).
2. Bumps the default page size to 256 so all 111 tools fit in a single `tools/list` response.

Effect: every gopeak tool is callable from Claude Code under its compact name (sanitized to dashes for OpenAI compat: `mcp__godot__dap-set-breakpoint`, etc.), without group activation. `manage_tool_groups activate` becomes a no-op for visibility — keep it only for documentation/intent.

Project `.mcp.json` must point at the local build:
```json
{
  "mcpServers": {
    "godot": {
      "command": "node",
      "args": ["/Users/nicholas/Developer/MCP/Gopeak-godot-mcp/build/cli.js"],
      "env": { "GODOT_PATH": "/Applications/Godot_mono.app/Contents/MacOS/Godot" }
    }
  }
}
```

Rebuild after pulling upstream or editing the fork: `cd ~/Developer/MCP/Gopeak-godot-mcp && npm run build`.

Re-evaluate this once gopeak ships compact aliases for dynamic-group tools upstream — at which point the fork can be retired and `.mcp.json` switched back to `"command": "gopeak"`.

### Dynamic groups (compact mode)

| Group | Purpose |
|---|---|
| `scene_advanced` | Duplicate / reparent nodes, load sprites |
| `uid` | UID resource management |
| `import_export` | Import pipeline, reimport, validate project |
| `autoload` | Autoload singletons, main scene |
| `signal` | Disconnect signals, list connections |
| `runtime` | Live scene tree, runtime properties, metrics (needs `godot_mcp_runtime` plugin on) |
| `resource` | Materials, shaders, custom resources |
| `animation` | Animations, tracks, AnimationTree, state machines |
| `plugin` | Enable/disable/list editor plugins |
| `input` | Input action mapping |
| `tilemap` | TileSet / TileMap operations |
| `audio` | Audio buses, effects, volume |
| `navigation` | Nav regions and agents |
| `theme_ui` | Theme colors, font sizes, shaders |
| `asset_store` | Search/download CC0 assets |
| `testing` | Screenshots, viewport capture, input injection |
| `dx_tools` | Error log, project health, find usages, scaffold |
| `intent_tracking` | Intent capture, decision logs, handoff briefs |
| `class_advanced` | Class inheritance inspection |
| `lsp` | GDScript completions, hover, symbols |
| `dap` | Breakpoints, stepping, stack traces |
| `version_gate` | Version validation, patch verification |

## Activation flow

1. **If the tool you need isn't in the visible (core) list**, activate its group:
   - **Discovery by query**: call `tool.catalog` with a keyword (e.g. `"animation"`, `"breakpoint"`, `"tilemap"`). Matching groups auto-activate.
   - **Explicit**: call `tool.groups` with the group name(s).
2. **Use the tool.** Most MCP clients refresh the tool list automatically on the server's `notifications/tools/list_changed`. If a freshly activated tool seems missing, just call it — the round-trip forces a refresh.
3. **Reset groups when done** with `tool.groups` to keep context lean for the next task.

## Task → tool routing

| Task | Group | Notes |
|---|---|---|
| Add/remove/rename a node | core (`scene.node.*`) | Don't hand-edit `.tscn` |
| Set node properties (position, exports) | core (`set_node_properties`) | |
| Duplicate or reparent | `scene_advanced` | |
| Create/modify a script | core (`script.create`, `script.modify`) | Bridge maintains `.uid` |
| Move/rename a script | `uid` + core | Hand-renaming breaks references |
| Inspect a running game's tree | `runtime` | Requires `godot_mcp_runtime` autoload enabled and game running |
| Call a method on a running node | `runtime` (`call_runtime_method`) | |
| Set a breakpoint, step, stack trace | `dap` | |
| LSP completions / hover / symbols | `lsp` | `lsp.diagnostics` is in core |
| AnimationPlayer, AnimationTree, state machines | `animation` | |
| TileSet, TileMap layers | `tilemap` | |
| Materials, shaders, custom Resource | `resource` | |
| Autoload singletons | `autoload` | |
| Input action map | `input` | |
| Audio buses & effects | `audio` | |
| Theme / fonts / UI shader | `theme_ui` | |
| Screenshot, inject key/click for UI test | `testing` (or core for basics) | |
| Search/download CC0 assets | `asset_store` | |
| Find usages, error log, project health | `dx_tools` | |
| Headless test runs, builds, exports | **Bash, not MCP** | Use Godot CLI |
| Pure data files (`.json`, plain `.gd`) | **Read/Edit/Write, not MCP** | MCP adds no value |

## Conventions

- **MCP for editor-touching artifacts; standard tools for data and prose.** A `.tscn` change goes through MCP; a `data/*.json` edit uses `Edit`.
- **Don't fight auto-reload.** `godot_mcp_auto_reload` re-imports anything edited externally while the editor is open. If you must edit a `.tscn`/`.tres` by hand, expect it to be reloaded by the editor — prefer the MCP path.
- **Don't hand-write `.uid` files.** The bridge owns them.
- **Runtime plugin is opt-in.** `godot_mcp_runtime` autoload is off by default; enable it (Project Settings → Plugins) only when you need to inspect a running game. Off = no runtime overhead.
- **Bridge port** is 6505 (override with `GODOT_BRIDGE_PORT` env var in `.mcp.json` if it conflicts).
- **Verify environment before MCP calls fail.** If a scene/resource MCP tool errors with "bridge unavailable":
  1. Is the Godot editor open?
  2. Is `godot_mcp_editor` enabled in **Project Settings → Plugins**?
  3. Is `GODOT_PATH` correct in `.mcp.json` (auto-detected if omitted)?

## Setup reference (one-time per machine / project)

**Global install:**
```bash
npm install -g gopeak
```

**Project `.mcp.json`** (using the local fork — see profile model section):
```json
{
  "mcpServers": {
    "godot": {
      "command": "node",
      "args": ["/Users/nicholas/Developer/MCP/Gopeak-godot-mcp/build/cli.js"],
      "env": {
        "GODOT_PATH": "/Applications/Godot.app/Contents/MacOS/Godot"
      }
    }
  }
}
```

**Editor addons** (run once at the project root):
```bash
curl -sL https://raw.githubusercontent.com/HaD0Yun/Gopeak-godot-mcp/main/install-addon.sh | bash
```
Then open the project in Godot and enable the relevant plugins under **Project Settings → Plugins** (`godot_mcp_editor` is required; `godot_mcp_auto_reload` recommended; `godot_mcp_runtime` only when needed).

After changing `.mcp.json`, restart Claude Code so it picks up the new MCP server.

## Anti-patterns

- Hand-editing `.tscn` while the editor is open and `godot_mcp_auto_reload` is on → silent reverts.
- Activating `full` profile when only one or two groups are needed → unnecessary token use.
- Using MCP for plain JSON data edits → slower than `Edit`, no benefit.
- Renaming a script via `mv` → breaks `.uid` cross-references; use the script-move MCP tool.
- Leaving every dynamic group activated across long sessions → bloated tool list. Reset when done.
