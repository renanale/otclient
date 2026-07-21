# Codebase Structure

**Analysis Date:** 2026-07-21

## Directory Layout

```
otclient/
├── src/                    # C++ sources
│   ├── main.cpp            # Desktop entry point
│   ├── androidmain.cpp     # Android entry point
│   ├── client/             # Tibia domain layer (game, map, protocol, things)
│   ├── framework/          # Engine (core, graphics, ui, net, sound, otml, lua)
│   ├── protobuf/           # Generated appearance protocol bindings
│   └── tools/              # Editor-only helpers (datdump)
├── modules/                # Lua feature modules (75) — core client + gameplay
├── mods/                   # Optional/user modules loaded after modules/
├── data/                   # Runtime assets: styles, fonts, images, locales, things
├── tests/                  # C++ unit tests (map, otml, stdext)
├── docs/                   # Building guides and HTML/flex examples
├── cmake/ , CMakeLists.txt , CMakePresets.json   # Build system
├── vcpkg.json , overlay-ports/                   # Dependency manifest
├── android/ , browser/                           # Platform build targets
├── init.lua                # First Lua executed at boot
├── otclientrc.lua          # Post-boot user hooks
├── config.ini              # Runtime configuration
└── AGENTS.md , README.md , CHANGELOG.md , BUGS
```

## Directory Purposes

**`src/framework/`:**
- Purpose: game-agnostic engine.
- Contains: `core/` (application, module manager, dispatcher, resources, logger), `graphics/` (drawpool, painter, shaders, particles, fonts), `ui/` (widgets, layouts, uimanager), `net/`, `sound/`, `input/`, `otml/`, `html/`, `luaengine/`, `platform/`, `stdext/`, `util/`, `proxy/`, `discord/`.
- Key files: `src/framework/core/graphicalapplication.cpp`, `src/framework/core/modulemanager.cpp`, `src/framework/ui/uimanager.cpp`, `src/framework/luaengine/luainterface.cpp`.

**`src/client/`:**
- Purpose: Tibia-specific domain model and protocol.
- Key files: `client.cpp`, `game.cpp`, `map.cpp`, `mapview.cpp`, `localplayer.cpp`, `protocolgame.cpp`, `protocolgameparse.cpp`, `protocolgamesend.cpp`, `thingtype.cpp`, `luafunctions.cpp`.

**`modules/`:**
- Purpose: all UI and gameplay behavior, one directory per feature.
- Groups: `corelib/`, `gamelib/`, `modulelib/` (shared libs); `client_*` (shell: entergame, options, topmenu, serverlist, terminal, locales, styles, assets); `game_*` (in-game features).

**`mods/`:**
- Purpose: optional add-ons — `game_bot`, `game_buttons`, `game_tasks`, `client_profiles`, `client_textedit`, `client_mods`.

**`data/`:**
- Purpose: shipped assets — `styles/` (base OTUI styles), `things/` (dat/spr/appearances), `fonts/`, `images/`, `sounds/`, `particles/`, `locales/`, `cursors/`, `json/`, `setup.otml`.

**`tests/`:**
- Purpose: CMake/CTest C++ unit tests. Key files: `tests/map/map_spectators_test.cpp`, `tests/otml/otml_alias_test.cpp`, `tests/stdext/string_encoding_test.cpp`.

## Key File Locations

**Entry Points:**
- `src/main.cpp`: desktop boot
- `src/androidmain.cpp`: Android boot
- `init.lua`: first Lua script, service + server config
- `otclientrc.lua`: user override hooks
- `modules/client/`: application shell module

**Configuration:**
- `config.ini`: runtime settings
- `data/setup.otml`: engine setup
- `vcpkg.json`: dependency manifest
- `CMakePresets.json`, `CMakeLists.txt`, `cmake/`: build config

**Core Logic:**
- `src/client/game.cpp`: session state and gameplay commands
- `src/client/protocolgameparse.cpp` / `protocolgamesend.cpp`: protocol
- `src/framework/core/modulemanager.cpp`: module lifecycle
- `src/framework/ui/uimanager.cpp`: widget tree

**Testing:**
- `tests/` with per-subdirectory `CMakeLists.txt`

## Naming Conventions

**Files:**
- C++: lowercase, no separators — `thingtypemanager.cpp`, `graphicalapplication.h`; header/source pairs share a stem; each subsystem has `declarations.h`.
- Lua modules: `modules/<module_name>/<modulename>.lua` (underscores dropped in the script name, e.g. `game_interface/gameinterface.lua`).
- UI: `.otui` for layouts/styles, `.otmod` for module manifests, `.otml` for generic config.
- Tests: `<subject>_test.cpp`.

**Directories:**
- Modules prefixed by scope: `client_*` (outside game), `game_*` (in game), `*lib` (shared library).
- Framework subsystems: single lowercase word (`core`, `graphics`, `luaengine`).

## Where to Add New Code

**New in-game feature (the default):**
- Directory: `modules/game_<feature>/`
- Manifest: `modules/game_<feature>/<feature>.otmod` with `sandboxed: true`, `scripts`, `dependencies`, `@onLoad: init()`, `@onUnload: terminate()`
- Logic: `modules/game_<feature>/<feature>.lua`
- Layout: `modules/game_<feature>/<feature>.otui`, custom widgets under `widgets/`, styles under `styles/`
- Register it in the `load-later` list of `modules/game_interface/interface.otmod`

**New engine capability:**
- Implementation: `src/framework/<subsystem>/`
- Declare types in that subsystem's `declarations.h`, add sources to `src/CMakeLists.txt`

**New Tibia domain object or protocol opcode:**
- Model: `src/client/`
- Parse/send: `src/client/protocolgameparse.cpp`, `src/client/protocolgamesend.cpp`, opcode in `src/client/protocolcodes.h`

**Exposing C++ to Lua:**
- Bindings: `src/client/luafunctions.cpp` (client) or `src/framework/luafunctions.cpp` (framework)
- Value conversion: `src/client/luavaluecasts_client.cpp`

**Shared Lua helpers:**
- Generic: `modules/corelib/`
- Game-aware: `modules/gamelib/`

**Tests:**
- `tests/<subsystem>/<subject>_test.cpp` plus a `CMakeLists.txt` entry

**Optional/user add-on:**
- `mods/<name>/`

## Special Directories

**`src/protobuf/`:** generated appearance protocol code — generated, committed.
**`vc18/`:** Visual Studio project files — committed, not generated by CMake.
**`records/`:** captured session/recording data — generated.
**`browser/`, `android/`:** platform-specific build scaffolding (triplets, overlay-ports, gradle) — committed.
**`overlay-ports/`:** vcpkg port overrides — committed.
**`.planning/`:** GSD planning artifacts — committed, not part of the build.

---

*Structure analysis: 2026-07-21*
