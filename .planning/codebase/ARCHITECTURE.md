<!-- refreshed: 2026-07-21 -->
# Architecture

**Analysis Date:** 2026-07-21

## System Overview

```text
┌─────────────────────────────────────────────────────────────┐
│                  Lua Module Layer (UI + gameplay)            │
├──────────────────┬──────────────────┬───────────────────────┤
│  client_* modules│  game_* modules  │  corelib / gamelib    │
│  `modules/client`│ `modules/game_*` │  `modules/corelib`    │
└────────┬─────────┴────────┬─────────┴──────────┬────────────┘
         │                  │                     │
         ▼                  ▼                     ▼
┌─────────────────────────────────────────────────────────────┐
│              Lua Binding Layer (LuaInterface)                │
│  `src/framework/luaengine/`, `src/client/luafunctions.cpp`   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Client domain layer  `src/client/`                          │
│  Game, Map, LocalPlayer, ProtocolGame, ThingType, MapView    │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Framework layer  `src/framework/`                           │
│  core · graphics · ui · net · sound · input · otml · stdext  │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Platform / OS: OpenGL(ES), SDL/Win32, sockets, filesystem   │
│  `src/framework/platform/`                                   │
└─────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| `main()` | Parse args, boot application, run `init.lua` | `src/main.cpp` |
| `Client` (`g_client`) | Registers client Lua bindings, inits/terminates all client singletons | `src/client/client.cpp` |
| `GraphicalApplication` | Main loop, frame pacing, poll/render split | `src/framework/core/graphicalapplication.cpp` |
| `ModuleManager` | Discovers, loads and unloads `.otmod` modules | `src/framework/core/modulemanager.cpp` |
| `LuaInterface` (`g_lua`) | Lua VM ownership, C++↔Lua binding, sandboxing | `src/framework/luaengine/luainterface.cpp` |
| `UIManager` (`g_ui`) | Widget tree, OTUI loading, event routing | `src/framework/ui/uimanager.cpp` |
| `Game` (`g_game`) | Game session state machine, gameplay commands | `src/client/game.cpp` |
| `ProtocolGame` | Tibia protocol encode/decode | `src/client/protocolgame.cpp`, `protocolgameparse.cpp`, `protocolgamesend.cpp` |
| `Map` / `MapView` | Tile/thing storage; viewport culling and rendering | `src/client/map.cpp`, `src/client/mapview.cpp` |
| `ThingTypeManager` (`g_things`) | Appearance/dat/spr metadata | `src/client/thingtypemanager.cpp` |
| `DrawPoolManager` | Batched draw command pools per layer | `src/framework/graphics/drawpoolmanager.cpp` |
| `EventDispatcher` (`g_dispatcher`) | Deferred/scheduled events, thread hand-off | `src/framework/core/eventdispatcher.cpp` |

## Pattern Overview

**Overall:** Layered C++ engine with a scripted, hot-loadable module plugin system on top (engine/scripting split, "everything above the protocol is Lua").

**Key Characteristics:**
- Global singleton managers (`g_game`, `g_map`, `g_ui`, `g_lua`, `g_things`, `g_dispatcher`) — the canonical way subsystems reach each other.
- All UI and gameplay presentation lives in Lua modules under `modules/`, declared by `.otmod` manifests with explicit dependency and `load-later` ordering.
- Declarative UI via OTML (`.otui`) styles/widgets, CSS-like, parsed by `src/framework/otml/`.
- Event-driven: C++ raises signals consumed by Lua callbacks (`onGameStart`, `onLogin`, widget `@onClick`).
- Separate poll thread and render thread coordinated by `GraphicalApplication`.

## Layers

**Framework (`src/framework/`):**
- Purpose: game-agnostic engine — windowing, rendering, networking, Lua, UI toolkit, sound, OTML parsing, HTML layout.
- Depends on: platform/OS, third-party deps via vcpkg (`vcpkg.json`).
- Used by: `src/client/`, Lua modules through bindings.

**Client domain (`src/client/`):**
- Purpose: Tibia-specific model and protocol — creatures, items, map, spritesheets, protocol game/login.
- Depends on: framework.
- Used by: Lua modules via `src/client/luafunctions.cpp`.

**Lua libraries (`modules/corelib/`, `modules/gamelib/`, `modules/modulelib/`):**
- Purpose: shared Lua helpers (table/string/math extensions, net message wrappers, keybinds, UI helpers).
- Used by: every feature module.

**Feature modules (`modules/client_*`, `modules/game_*`, 75 total):**
- Purpose: one self-contained feature each (login, hotkeys, market, cyclopedia, containers…).
- Location: `modules/<name>/<name>.lua` + `.otui` + `.otmod`.

**Mods (`mods/`):**
- Purpose: optional/user-supplied modules loaded after core modules (`mods/game_bot`, `mods/client_profiles`).

## Data Flow

### Startup

1. `main()` initializes app + resources (`src/main.cpp`)
2. `g_client.init()` registers Lua bindings and inits `g_gameConfig`, `g_map`, `g_minimap`, `g_game`, `g_shaders`, `g_sprites`, `g_spriteAppearances`, `g_things` (`src/client/client.cpp:44`)
3. `init.lua` runs — defines `Services`, `Servers_init`, loads first modules
4. `ModuleManager` discovers `.otmod` files under `modules/` and `mods/`, resolves dependencies, fires `@onLoad` (`src/framework/core/modulemanager.cpp`)
5. `modules/client` composes the shell; `modules/game_interface` loads all in-game modules listed in its `load-later` block

### Frame loop

1. `GraphicalApplication::run()` polls input and network (`src/framework/core/graphicalapplication.cpp`)
2. `g_dispatcher` flushes queued and scheduled events
3. `g_ui` updates/lays out the widget tree; `MapView` culls and enqueues tiles
4. `DrawPoolManager` flushes batched pools to `Painter`, then swap buffers

### Server packet

1. Socket read → `ProtocolGame::onRecv` (`src/client/protocolgame.cpp`)
2. Opcode dispatch in `src/client/protocolgameparse.cpp`
3. Mutates `g_game` / `g_map` / `LocalPlayer` state
4. C++ raises a Lua signal; module callbacks update widgets (e.g. `modules/game_healthinfo`)

**State Management:**
- Authoritative runtime state lives in C++ singletons; Lua modules hold view state only and re-read from `g_game` on events.
- Persisted settings via `g_configs` / `modules/corelib/settings.lua` and `config.ini`.

## Key Abstractions

**Module (`.otmod`):**
- Purpose: unit of load/unload with `sandboxed`, `scripts`, `dependencies`, `load-later`, `@onLoad`/`@onUnload`.
- Example: `modules/game_interface/interface.otmod`.

**UIWidget / OTUI style:**
- Purpose: composable widget with anchors and named styles; layouts in `src/framework/ui/uianchorlayout.cpp`, `uiboxlayout.cpp`, `uigridlayout.cpp`, `uilayoutflexbox.cpp`.
- Examples: `modules/game_interface/gameinterface.otui`, `data/styles/`.

**Thing / ThingType:**
- Purpose: anything on a tile (item, creature, effect, missile) and its appearance metadata.
- Files: `src/client/thing.cpp`, `src/client/thingtype.cpp`, `src/client/item.cpp`, `src/client/creature.cpp`.

**DrawPool:**
- Purpose: deferred, batched, hashable draw commands enabling the render/poll thread split.
- File: `src/framework/graphics/drawpool.cpp`.

## Entry Points

**`src/main.cpp`** — desktop entry; parses CLI args, boots `GraphicalApplication`, executes `init.lua`.
**`src/androidmain.cpp`** — Android/NDK entry.
**`init.lua`** — first Lua executed; service URLs, server list, initial module load.
**`otclientrc.lua`** — post-boot user hooks/overrides.
**`modules/startup/`, `modules/client/`** — application shell modules.

## Architectural Constraints

- **Threading:** poll/update thread plus a render thread; cross-thread work must go through `g_dispatcher` (`eventdispatcher.h`) or `AsyncDispatcher`. Lua is single-threaded — never touch `g_lua` off the main thread.
- **Global state:** pervasive singletons (`g_client`, `g_game`, `g_map`, `g_ui`, `g_lua`, `g_things`, `g_sprites`, `g_dispatcher`, `g_configs`) declared in each subsystem header and defined in the matching `.cpp`.
- **Init/terminate symmetry:** every singleton inited in `Client::init` must be torn down in `Client::terminate` in reverse order.
- **Compile-time feature flags:** `FRAMEWORK_EDITOR`, `FRAMEWORK_NET`, `ENABLE_DISCORD_RPC`, `ANDROID` gate whole subsystems — guard new includes the same way.
- **Module load order:** `.otmod` `dependencies` / `load-later` is the only ordering mechanism; a module must not assume another is loaded without declaring it.
- **Sandboxing:** `sandboxed: true` modules get an isolated Lua env; globals leak only when explicitly exported.

## Anti-Patterns

### Direct cross-module Lua globals

**What happens:** a module reads another module's globals directly instead of declaring a dependency.
**Why it's wrong:** breaks unload/reload and sandboxing; nil-crashes when load order shifts.
**Do this instead:** declare it in `dependencies`/`load-later` in the `.otmod` and use exported accessor functions (see `modules/game_interface/interface.otmod`).

### Gameplay logic in C++

**What happens:** UI or gameplay rules added to `src/client/` instead of a Lua module.
**Why it's wrong:** the project's core value is that all interface behavior is scriptable and hot-reloadable without recompiling.
**Do this instead:** expose a minimal binding in `src/client/luafunctions.cpp` and implement behavior in `modules/game_*`.

### Rendering outside a draw pool

**What happens:** immediate `Painter` calls from arbitrary code.
**Why it's wrong:** bypasses batching and breaks the render-thread split.
**Do this instead:** enqueue via `g_drawPool` (`src/framework/graphics/drawpoolmanager.h`).

### Blocking the main loop

**What happens:** synchronous file/network I/O during update.
**Why it's wrong:** stalls frames for every player.
**Do this instead:** use `AsyncDispatcher` (`src/framework/core/asyncdispatcher.h`) and post the result back via `g_dispatcher`.

## Error Handling

**Strategy:** C++ exceptions (`stdext::exception`) for recoverable failures at boundaries; `g_logger` for reporting.

**Patterns:**
- `g_logger.error/warning/fatal` (`src/framework/core/logger.h`) rather than raw `std::cerr`.
- Lua errors are caught by `LuaInterface` and surfaced as traceback logs; module `@onLoad` failure unloads the module rather than killing the client.
- Protocol parse errors abort the packet and log the opcode (`src/client/protocolgameparse.cpp`).

## Cross-Cutting Concerns

**Logging:** `g_logger` in C++, `pinfo`/`perror`/`pwarning` in Lua (`modules/corelib/globals.lua`).
**Validation:** protocol-level bounds checks in `InputMessage`; OTML schema errors raised during style parsing.
**Authentication:** login handled by `modules/client_entergame` over `ProtocolLogin`; session token held by `g_game`.
**Localization:** `modules/client_locales` + `data/locales/`.
**Configuration:** `config.ini`, `data/setup.otml`, `init.lua` `Services` table.

---

*Architecture analysis: 2026-07-21*
