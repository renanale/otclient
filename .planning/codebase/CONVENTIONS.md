# Coding Conventions

**Analysis Date:** 2026-07-21

This is a hybrid C++20 / Lua codebase (OTClient). C++ engine lives in `src/`, Lua game
modules in `modules/`, `mods/`, `data/`, with entry scripts `init.lua` and `otclientrc.lua`.

## Naming Patterns

**Files:**
- C++: all-lowercase, no separators — `src/client/attachedeffectmanager.cpp`, `src/client/localplayer.h`
- Headers paired 1:1 with sources in the same directory (`animator.h` / `animator.cpp`)
- Lua modules: directory named `<category>_<name>` with matching entry file —
  `modules/game_interface/gameinterface.lua`, `modules/client_options/`
- Tests: `<subject>_test.cpp` under `tests/<area>/` — `tests/map/map_spectators_test.cpp`

**Types (C++):**
- Classes: PascalCase — `Animator`, `LocalPlayer`, `OTMLDocument`
- Enums: PascalCase type with prefixed PascalCase values —
  `enum AnimationPhase : int16_t { AnimPhaseAutomatic = -1, ... }` in `src/client/animator.h`
- Always give enums an explicit underlying type (`: int16_t`, `: uint8_t`)
- Shared-pointer aliases end in `Ptr` — `FileStreamPtr`, `OTMLNodePtr`, `OTMLDocumentPtr`;
  declared in `declarations.h` files (`src/client/declarations.h`, `src/framework/core/declarations.h`)

**Functions/Variables (C++):**
- Methods: camelCase — `unserializeAppearance()`, `getPhaseAt()`, `resetAnimation()`
- Member fields: `m_` prefix, camelCase — `m_animationPhases`, `m_phaseDurations`, `m_loopType`
- Accessors: `getX()` / `setX()`; booleans use `isX()` — `isAsync()`, `isComplete()`, `isInfiniteLoop()`
- Free functions in `stdext` use snake_case — `stdext::is_valid_utf8()`, `stdext::utf8_to_latin1()`
- Global singletons use `g_` prefix — `g_logger`, `g_game`, `g_platform`, `g_resources`

**Lua:**
- Functions and variables: camelCase — `updateSidePanelButtons()`, `lastManualWalk`
- Module-scoped helpers use `local function`; module API functions are globals in the module chunk
- Cross-module access via `modules.<module_name>.<fn>` — `modules.client_options.getOption('showLeftPanel')`

## Code Style

**Formatting (C++):** driven by `.editorconfig` (MSVC `cpp_*` keys), not clang-format.
- 4-space indent; namespace contents indented, access specifiers NOT indented
- Braces: same line for blocks/lambdas/functions; **new line before open brace for types**
  (`cpp_new_line_before_open_brace_type = new_line`) — see class declarations in `src/client/animator.h`
- `else`, `catch`, `while` of do-while stay on the closing-brace line
- Pointer/reference binds left: `const std::string& s`, `ThingType* type`
- No space before function call parens; space after control-flow keywords (`if (x)`)
- Spaces around binary, assignment, and ternary operators
- Single-line scopes are preserved (`cpp_wrap_preserve_blocks = all_one_line_scopes`) —
  short guard bodies on one line are idiomatic
- `CMakeLists.txt`: tab indent, size 2, trailing whitespace trimmed, final newline

**Linting:** enforced in CI via reviewdog (`.reviewdog.yml`, `.github/workflows/reusable-checks.yml`)
- `cppcheck --enable=warning,style,performance,portability` over `src/` (advisory; `src/protobuf` excluded)
- `luacheck` over `data/` and `modules/` — ignores `4 211 212 213 311 542`, no line-length rules (advisory)
- `luac` (`luajit -b`) — **fail-level error**: every Lua file must compile
- `xmllint` over `data/**/*.xml` — **fail-level error**
- `yamllint -c .yamllint.yaml .github` — blocking
- All reviewdog runners use `-filter-mode=added` (only new lines are reported)

## Includes & Imports

**C++ order** (see `src/client/animator.cpp`, `tests/map/map_spectators_test.cpp`):
1. Own header first, quoted relative: `#include "animator.h"`
2. Project headers, quoted with module-rooted path: `#include "framework/core/clock.h"`, `#include "client/tile.h"`
3. Framework/angle-bracket project headers: `#include <framework/core/logger.h>`
4. Standard library / third-party: `<string>`, `<sstream>`, `<gtest/gtest.h>`

- Headers use `#pragma once` (never include guards) — `src/client/animator.h`
- Optional subsystems are guarded by feature macros: `#ifdef FRAMEWORK_PROTOBUF`,
  plus `CLIENT`, `FRAMEWORK_GRAPHICS`, `FRAMEWORK_NET`, `FRAMEWORK_SOUND`, `FRAMEWORK_XML`
- Forward declarations and `Ptr` aliases belong in `declarations.h`, not in consumer headers

**Lua:** no import statement; modules are discovered via `.otmod` files and reached through the
global `modules` table.

## Error Handling

**C++ — two-tier strategy:**
- **Programmer/API misuse → throw:** `throw Exception("Unable to change protocol version while online");`
  (`src/client/game.cpp:1703`). `Exception` accepts fmt-style formatting:
  `throw Exception("Protocol version {} not supported", version);` (`game.cpp:1706`)
- **Recoverable / protocol-level anomalies → log and continue:**
  `g_logger.error("Got an invalid ping from server");` (`game.cpp:266`),
  `g_logger.warning("Unknown bounty task action type {}", static_cast<int>(actionType));` (`game.cpp:1894`)
- Invariants inside trusted paths use `assert()` — `assert(m_animationPhases == static_cast<int>(m_phaseDurations.size()));`
  in `src/client/animator.cpp`
- Never crash on malformed server data; degrade with a warning that names the offending value

**Rule of thumb:** throw only for states the caller could have prevented; log for anything driven
by remote input or asset content.

## Logging

**Framework:** `g_logger` from `src/framework/core/logger.h`.

- Levels used: `g_logger.error(...)`, `g_logger.warning(...)`, plus fatal/info/debug
- Messages are fmt-formatted; pass values as arguments, do not concatenate
- Cast enums/small integer types explicitly for logging: `static_cast<int>(actionType)`
- Prefix the message with the subsystem when ambiguous — `"BountyTalismanUpgrade: invalid pathIndex {}"`

## Comments

- Every C++ source and header opens with the MIT copyright block
  (`Copyright (c) 2010-2026 OTClient <https://github.com/edubart/otclient>`) — copy it verbatim into new files
- Inline comments explain **why**, not what, and are used for non-obvious workarounds:
  the zero-duration phase fallback in `src/client/animator.cpp` is commented over two lines
- Lua globals that exist for cross-module contracts are documented at declaration —
  see `lastManualWalk` in `modules/game_interface/gameinterface.lua`
- No JSDoc/Doxygen convention is applied; there are no `@param`/`@return` blocks

## Function & Class Design

- Trivial accessors are defined inline in the header body
  (`int getAnimationPhases() const { return m_animationPhases; }`)
- Non-trivial logic goes in the `.cpp`
- Mark leaf classes `final`; mark overrides `override` (see `DummyCreature` in `tests/map/map_spectators_test.cpp`)
- Ownership uses `std::shared_ptr` with `enable_shared_from_this` where self-reference is needed
  (`class Animator : public std::enable_shared_from_this<Animator>`)
- Prefer `const auto&` for loop bindings and structured bindings for pairs:
  `for (const auto& [mn, mx] : m_phaseDurations)`
- Use `emplace_back` over `push_back` for in-place construction
- Keep helper functions that are file-local inside an anonymous `namespace { ... }`

## Module Design

**C++:** two top-level layers — `src/framework/` (engine: core, graphics, net, sound, otml, stdext,
luaengine) and `src/client/` (game logic). `src/client/` depends on `src/framework/`; never the reverse.
Lua bindings are registered in `src/client/luafunctions.cpp` and `src/client/luavaluecasts_client.cpp`.

**Lua:** one directory per module with an `.otmod` descriptor, `.lua` logic and `.otui` layout.
Module-private state uses `local`; only what other modules need is left global.

---

*Convention analysis: 2026-07-21*
