# Codebase Concerns

**Analysis Date:** 2026-07-21

## Tech Debt

**Protocol parsing monolith:**
- Issue: `src/client/protocolgameparse.cpp` is 7,554 lines — a single flat switch over every server opcode, with parsing, validation and game-state mutation interleaved.
- Files: `src/client/protocolgameparse.cpp`, `src/client/protocolgamesend.cpp` (1,849 lines)
- Impact: Merge conflicts on nearly every protocol PR; opcode handlers cannot be unit tested in isolation; version-specific branches (`g_game.getClientVersion() >= X`) are scattered instead of centralized.
- Fix approach: Extract opcode handlers into per-domain translation units (market, bestiary, imbuement, cyclopedia) registered into a dispatch table; keep the switch as a thin router.

**Unfinished protocol features left as stubs:**
- Issue: Multiple opcodes are parsed and then discarded — data is read off the wire but never surfaced to Lua/UI.
- Files: `src/client/protocolgameparse.cpp:5394` (game news), `:5485` (items price), `:5509` (bestiary entry changed), `:4982` (second item id for double bundle), `:2214` (trapper creature state), `:2495` (`processEditText` with `ItemPtr`)
- Impact: Silent feature gaps; servers sending these packets appear to work but the client shows nothing.
- Fix approach: Either wire each to a Lua signal or explicitly document as unsupported in `docs/`.

**Tile rendering marked for refactor:**
- Issue: Two explicit `// TODO: Need refactoring` markers on core drawing/stacking logic.
- Files: `src/client/tile.cpp:240`, `src/client/tile.cpp:335`
- Impact: Tile stack ordering is one of the most bug-prone areas of the client and it is flagged unmaintainable by its own authors.
- Fix approach: Extract stack-position resolution into a pure function that can be covered by gtest alongside `tests/map/map_spectators_test.cpp`.

**Map/minimap serialization without compression:**
- Issue: `//TODO: compression flag with zlib` duplicated in two serializers.
- Files: `src/client/mapio.cpp:489`, `src/client/minimap.cpp:383`
- Impact: Larger on-disk OTBM/minimap artifacts than necessary.

**Map naming heuristics via substr:**
- Issue: `/// TODO: Ask the user for a map name instead of those ugly uses of substr`
- Files: `src/client/mapio.cpp:257`
- Impact: Map identity derived from path string slicing; breaks on unusual paths.

**Lua-eligible logic kept in C++:**
- Issue: `//TODO: this could be moved to lua` on static text formatting.
- Files: `src/client/statictext.cpp:57`, `src/client/statictext.cpp:119`
- Impact: Server owners cannot customize message rendering without recompiling.

**Shader manager mixes framework and client concerns:**
- Issue: `// TODO: Move these setup methods to a ClientShaderManager`
- Files: `src/framework/graphics/shadermanager.h:49`
- Impact: The `framework/` layer depends on client-specific shader assumptions, weakening the framework/client separation the tree otherwise enforces.

## Known Bugs

**X11 window handling contains an acknowledged incorrect fix:**
- Symptoms: `// TODO: fix this in a correct way` at `src/framework/platform/x11window.cpp:1417`, plus three bare `//TODO` gaps at `:725`, `:735`, `:1342`.
- Files: `src/framework/platform/x11window.cpp`
- Trigger: Linux window state / input edge cases.
- Workaround: The existing hack; no upstream fix.

**Win32 window unimplemented paths:**
- Symptoms: bare `//TODO` at `src/framework/platform/win32window.cpp:442` and `:460`.
- Files: `src/framework/platform/win32window.cpp`
- Impact: Feature parity gaps between Windows and other platforms are undocumented.

**Sprite overwrite detection missing:**
- Symptoms: `//TODO: Check for overwritten sprites.`
- Files: `src/client/spritemanager.cpp:198`
- Trigger: Loading spr sets where sprite ids collide — later sprite silently wins.

**Incomplete mipmap generation:**
- Symptoms: `//FIXME: calculate mipmaps for 8x1, 4x1, 2x1 ...`
- Files: `src/framework/graphics/image.cpp:208`, related `//TODO` at `src/framework/graphics/texture.cpp:275`
- Impact: Non-power-of-two / extreme aspect textures get degraded minification.

## Security Considerations

**HTTPS login disables all certificate and hostname verification (highest priority):**
- Risk: Full MITM of the login flow. Credentials (`email`, `password`), the 2FA `token`, and the returned session key are sent over a TLS channel whose peer identity is never checked.
- Files: `src/framework/net/httplogin.cpp:381-383` — `set_ca_cert_path("./cacert.pem")` is immediately negated by `enable_server_certificate_verification(false)` and `enable_server_hostname_verification(false)`.
- Current mitigation: None. `cacert.pem` ships at the repo root but is effectively unused.
- Recommendations: Enable both verifications by default; expose an opt-out only as an explicit, loudly-logged developer/self-signed-server flag. Resolve the CA bundle relative to the resource dir rather than the process CWD (`"./cacert.pem"` breaks whenever the client is launched from another directory, silently forcing the fallback path).

**No TLS layer on the HTTP module:**
- Risk: `src/framework/net/protocolhttp.cpp` (1,181 lines) contains no `ssl`/`verify` usage at all, so module-driven HTTP traffic (updater/asset downloads) is plaintext-only.
- Files: `src/framework/net/protocolhttp.cpp`
- Recommendations: Audit which callers download executable or asset content over this path; asset installation (`feat(client): add automatic modern asset installation`) fetching over unauthenticated HTTP is a code-execution-adjacent supply-chain risk. Require HTTPS plus a content hash for anything written to disk.

**Untrusted network input parsed with raw buffer primitives:**
- Risk: `src/framework/net/inputmessage.cpp` guards reads via `canRead()`/`checkRead()` which throw `stdext::exception`, but adjacent parsers use raw `memcpy`/C string functions.
- Files: `src/framework/net/inputmessage.cpp:137-152`, `src/framework/net/inputmessage.h`, `src/framework/net/outputmessage.cpp`, `src/client/protocolgameparse.cpp`
- Current mitigation: Bounds checks in `InputMessage`; XTEA key generated from `std::random_device` (`src/framework/net/protocol.cpp:319-321`) which is correct.
- Recommendations: Ensure every `protocolgameparse` handler that sizes a loop from a wire-supplied count also bounds that count; a malicious server can otherwise drive multi-million-iteration allocations. Consider fuzzing `InputMessage` + the opcode dispatcher.

**Archive/decompression code is vendored C:**
- Risk: `src/framework/core/minizip/unzip.c` and `ioapi_mem.c` are third-party C using `memcpy` throughout, driven by downloaded/user-supplied archives (`src/framework/core/unzipper.cpp`, `src/client/spriteappearances.cpp`).
- Recommendations: Pin and document the vendored minizip version; check it against known CVEs; validate uncompressed sizes to prevent zip-bomb expansion.

**Lua is not sandboxed:**
- Risk: `dofile`/`dofiles` are replaced with resource-manager-backed versions (`src/framework/luaengine/luainterface.cpp:583,599,823-829`), but modules and mods run with full engine bindings. `mods/` content is user-installable.
- Recommendations: Document explicitly that installing a third-party mod is equivalent to running arbitrary code, and avoid any feature that auto-downloads Lua.

## Performance Bottlenecks

**Per-call Lua object metatable lookup:**
- Problem: `// TODO: this could be cached for more performance`
- Files: `src/framework/luaengine/luaobject.cpp:120`
- Cause: Repeated lookup on every Lua<->C++ object crossing, which is on the hot path for UI and game events.
- Improvement path: Cache the resolved metatable/class ref per type.

**Redundant widget property writes:**
- Problem: `//TODO: prevent setting already set proprieties`
- Files: `src/framework/ui/uiwidget.cpp:1906`
- Cause: Setters do not early-out on unchanged values, so layout/repaint invalidation fires unnecessarily.
- Improvement path: Compare-and-skip in the setters; measurable win in list-heavy UIs.

**Uncompressed minimap/map I/O:** see Tech Debt — more disk I/O and larger memory buffers than needed.

## Fragile Areas

**UI widget core:**
- Files: `src/framework/ui/uiwidget.cpp` (2,613 lines), `src/framework/ui/uiwidgethtml.cpp` (2,091), `src/framework/ui/uitextedit.cpp` (1,762), `src/framework/ui/uilayoutflexbox.cpp` (1,282)
- Why fragile: Very large, deeply coupled to Lua bindings and OTML styles; behavior changes here affect every module under `modules/`.
- Safe modification: Change via the narrowest setter; verify against `modules/` visually — there is no automated UI regression coverage.
- Test coverage: None. Only OTML parsing is tested (`tests/otml/otml_alias_test.cpp`).

**Platform windowing backends:**
- Files: `src/framework/platform/x11window.cpp` (1,452), `src/framework/platform/win32window.cpp` (1,259), `src/framework/platform/cocoawindow.mm`
- Why fragile: Three independent implementations of one contract, each with its own TODO gaps; a fix on one platform routinely does not exist on the others.
- Safe modification: Change all three together or the divergence widens.
- Test coverage: None; only CI compilation per platform.

**Client/framework layering leak:**
- Files: `src/framework/graphics/shadermanager.h`, `src/framework/luafunctions.cpp` vs `src/client/luafunctions.cpp`
- Why fragile: `framework/` is meant to be client-agnostic (it is a separately reusable layer) but carries client assumptions; refactors that assume the boundary holds will break.

## Scaling Limits

**Single-file opcode dispatch:**
- Current capacity: ~7.5k lines, one switch.
- Limit: Every new Tibia protocol version adds branches to the same file; compile time and conflict rate grow linearly with protocol support.
- Scaling path: Table-driven per-version handler registration.

**Version conditionals scattered across the client:**
- Current capacity: Supports a broad range of client versions via inline `getClientVersion()` comparisons.
- Limit: Adding a new version requires auditing every conditional by hand.
- Scaling path: A single capability/feature-flag table keyed by version (`src/client/const.h` is already 1,578 lines and is the natural home).

## Dependencies at Risk

**Vendored minizip:**
- Risk: Copied into `src/framework/core/minizip/` rather than consumed via vcpkg, so it does not receive `vcpkg.json` baseline updates from `.github/workflows/update_vcpkg_baseline.yml`.
- Impact: Silent exposure to upstream archive-parsing CVEs.
- Migration plan: Move to the vcpkg-managed `minizip`/`libzip` port, or record the vendored upstream commit and add it to the update workflow.

**LuaJIT:**
- Risk: LuaJIT upstream releases are infrequent; the project maintains its own Android build path (`build_luajit_android.sh`, `overlay-ports/`).
- Impact: Custom overlay ports must be re-validated on every toolchain bump.
- Migration plan: None needed short-term; keep `overlay-ports/` documented as the source of truth.

**Bundled `cacert.pem`:**
- Risk: A static CA bundle at the repo root that will expire/rotate and has no refresh mechanism — and is currently not even honored (see Security).
- Migration plan: Use the OS trust store where available; otherwise automate bundle refresh.

## Missing Critical Features

**Certificate pinning / verified asset delivery:**
- Problem: No integrity verification for downloaded assets or login responses.
- Blocks: Safe use of the automatic asset installation feature on untrusted networks.

**Structured protocol version capability table:**
- Problem: Version support is expressed as ad-hoc inline comparisons.
- Blocks: Confidently declaring which client versions are actually supported.

## Test Coverage Gaps

**Protocol parsing — zero coverage:**
- What's not tested: All of `src/client/protocolgameparse.cpp` and `src/client/protocolgamesend.cpp`, plus `src/framework/net/inputmessage.cpp` bounds behavior.
- Files: `src/client/protocolgameparse.cpp`, `src/client/protocolgamesend.cpp`, `src/framework/net/inputmessage.cpp`
- Risk: This is the untrusted-input boundary of the whole application and it is entirely unverified. Regressions ship silently.
- Priority: High

**UI framework — zero coverage:**
- What's not tested: `src/framework/ui/` (~8k lines across widget, html, textedit, flexbox).
- Files: `src/framework/ui/uiwidget.cpp`, `uiwidgethtml.cpp`, `uitextedit.cpp`, `uilayoutflexbox.cpp`
- Risk: Layout and text-editing regressions reach users; only caught by manual play.
- Priority: High

**Overall test suite is minimal:**
- What's not tested: The gtest suite consists of exactly three targets — `tests/map/map_spectators_test.cpp`, `tests/otml/otml_alias_test.cpp`, `tests/stdext/string_encoding_test.cpp` — against a ~102k-line C++ codebase.
- Files: `tests/CMakeLists.txt`, `tests/*/CMakeLists.txt`
- Risk: Effectively no safety net for refactoring.
- Priority: High

**Tests are off by default in the build:**
- What's not tested: `OTCLIENT_BUILD_TESTS` defaults to `OFF` in `CMakeLists.txt`, so local contributors do not run the suite unless they opt in.
- Files: `CMakeLists.txt`
- Risk: Test breakage discovered only in CI.
- Priority: Medium

**Static analysis is advisory only:**
- What's not tested: `.github/workflows/reusable-checks.yml` runs `cppcheck` and `luacheck` through reviewdog with `|| true`, so their findings never fail a PR (only `luac` and `xmllint` are `-fail-level=error`). Filtering is `-filter-mode=added`, so pre-existing issues are permanently invisible.
- Files: `.github/workflows/reusable-checks.yml`
- Risk: C++ defects flagged by cppcheck are routinely merged.
- Priority: Medium

**Platform windowing — zero coverage:**
- What's not tested: `src/framework/platform/*` beyond "it compiles".
- Risk: The most divergent code in the tree has the least verification.
- Priority: Medium

---

*Concerns audit: 2026-07-21*
