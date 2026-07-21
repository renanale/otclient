# Technology Stack

**Analysis Date:** 2026-07-21

## Languages

**Primary:**
- C++ (C++20, C++23 on some toolchains) - Engine and client, `src/framework/`, `src/client/`
- Lua (LuaJIT dialect) - Game modules and UI logic, `modules/`, `mods/`, `init.lua`, `otclientrc.lua`

**Secondary:**
- Protocol Buffers - Appearance/sound/static data schemas, `src/protobuf/appearances.proto`, `src/protobuf/sounds.proto`, `src/protobuf/staticdata.proto`
- Shell / Batch / Python - Build and tooling scripts, `recompile.sh`, `build_luajit_android.sh`, `tools/emscripten-web-serve.py`
- Java/Gradle - Android host app, `android/`

## Runtime

**Environment:**
- Native desktop (Windows, Linux, macOS)
- Android (NDK, `android/`, `Dockerfile.android`)
- WebAssembly via Emscripten (`browser/`, `Dockerfile.browser`, `WASM` CMake option)

**Package Manager:**
- vcpkg (manifest mode) - `vcpkg.json`, `vcpkg-configuration.json`
- Lockfile equivalent: `builtin-baseline` pinned to `cd61e1e26a038e82d6550a3ebbe0fbbfe7da78e3`
- Custom port overlays: `overlay-ports/`

## Frameworks

**Core:**
- Custom in-house engine (`src/framework/`) with subsystems: `core`, `graphics`, `sound`, `net`, `ui`, `luaengine`, `otml`, `html`, `input`, `platform`, `proxy`, `stdext`
- LuaJIT - Scripting VM (desktop/native only; excluded on `android` and `wasm32` per `vcpkg.json`)

**Testing:**
- GoogleTest - `tests/CMakeLists.txt`, enabled by `OTCLIENT_BUILD_TESTS` + vcpkg feature `tests`
- Lua test workflow - `.github/workflows/reusable-tests-lua.yml`

**Build/Dev:**
- CMake >= 3.16 - `CMakeLists.txt`, presets in `CMakePresets.json`, modules in `cmake/`
- ccache / sccache (opt-in: `OPTIONS_ENABLE_CCACHE`, `OPTIONS_ENABLE_SCCACHE`)
- IPO/LTO enabled by default (`OPTIONS_ENABLE_IPO`)
- Docker - `Dockerfile`, `Dockerfile.android`, `Dockerfile.browser`
- Visual Studio solution - `vc18/`

## Key Dependencies

**Critical:**
- `asio` - Async networking backbone for `src/framework/net/connection.cpp`
- `luajit` - Scripting engine bound in `src/framework/luaengine/`
- `protobuf` + `abseil` - Appearance/sound data decoding (`src/protobuf/`)
- `openssl` - TLS for HTTPS login (`src/framework/net/httplogin.cpp`)
- `cpp-httplib` - HTTP/HTTPS client (`httplib::SSLClient` in `src/framework/net/httplogin.cpp`)
- `ixwebsocket` - WebSocket transport (non-wasm), `src/framework/net/webconnection.cpp`

**Graphics/Audio:**
- `opengl`, `glew` (Windows/Linux/macOS), `angle` (Windows)
- `freetype` - Font rendering
- `libpng` (with `apng` feature), `libogg`, `libvorbis`, `openal-soft`

**Infrastructure/Utility:**
- `physfs` - Virtual filesystem for data archives
- `zlib`, `liblzma`, `libarchive` - Compression/archives
- `nlohmann-json`, `pugixml`, `inih[cpp]` - JSON/XML/INI parsing
- `fmt`, `spdlog` - Formatting and logging
- `parallel-hashmap`, `bshoshany-thread-pool`, `stduuid`, `utfcpp`, `cppcodec`, `libobfuscate`
- `discord-rpc` - Rich Presence (`src/framework/discord/`)
- `gtest` - Test feature only

## Configuration

**Runtime config:**
- `config.ini` - Public graphics/atlas and font settings
- `init.lua`, `otclientrc.lua` - Lua bootstrap and per-user runtime config
- `cacert.pem` - Bundled CA bundle for TLS verification

**Build config:**
- `CMakeLists.txt`, `src/CMakeLists.txt`, `CMakePresets.json`, `cmake/`
- `vcpkg.json` (dependencies + `tests` feature), `vcpkg-configuration.json`
- Env vars consumed by build: `VCPKG_ROOT`, `VCPKG_DEFAULT_TRIPLET`
- Key CMake options: `OTCLIENT_BUILD_CLIENT`, `OTCLIENT_BUILD_TESTS`, `WASM`, `VCPKG_TARGET_ANDROID`, `TOGGLE_PRE_COMPILED_HEADER`

**Code style/lint:**
- `.editorconfig`, `.yamllint.yaml`, `.reviewdog.yml`

## Platform Requirements

**Development:**
- CMake 3.16+, a C++20-capable compiler (MSVC 2022 / GCC / Clang), vcpkg checkout with `VCPKG_ROOT` set
- Android: NDK toolchain via `cmake/vcpkg_android.cmake`, helper scripts `setup_android_deps.sh`, `setup_android_wsl.sh`
- Browser: Emscripten toolchain (`em++`), pthreads/atomics/bulk-memory flags applied automatically

**Production:**
- Distributed as a native game client binary per platform, plus an Android APK and a WASM browser build.

---

*Stack analysis: 2026-07-21*
