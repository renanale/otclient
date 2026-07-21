# External Integrations

**Analysis Date:** 2026-07-21

## APIs & External Services

**Game servers (primary integration):**
- Open Tibia / Tibia-protocol game servers - Binary TCP protocol
  - Client: asio-based `src/framework/net/connection.cpp`, protocol framing in `src/framework/net/protocol.cpp`, `inputmessage.cpp`, `outputmessage.cpp`
  - Opcode mapping: `src/client/protocolcodes.cpp`
  - Endpoint (host/port) supplied at runtime by the user via the Lua login UI in `modules/`, not hardcoded

**Login / session web API:**
- HTTP(S) login endpoint exposed by the server operator
  - Client: `cpp-httplib` (`httplib::SSLClient` / `httplib::Client`) in `src/framework/net/httplogin.cpp`
  - Scheme selected at runtime (`http://` vs `https://`), see `src/framework/net/httplogin.cpp:321`
  - Sends `User-Agent: Mozilla/5.0`
  - TLS trust anchors from bundled `cacert.pem`

**Generic HTTP/WebSocket layer:**
- `src/framework/net/protocolhttp.cpp` - Lua-exposed HTTP requests/downloads
- `src/framework/net/webconnection.cpp` - WebSocket transport (`ixwebsocket`), used for browser/WASM and proxied play

**Discord:**
- Discord Rich Presence via `discord-rpc` - `src/framework/discord/discord.cpp`, `discord.h` (`g_discord`)
  - Application ID configured by the distributing project, not an org secret in-repo

**Proxy:**
- Custom proxy relay client - `src/framework/proxy/proxy.cpp`, `proxy_client.cpp`

## Data Storage

**Databases:**
- None. No database client or ORM is present.

**File Storage:**
- Local filesystem via PhysicsFS virtual filesystem (`physfs`), data assets under `data/`, `modules/`, `mods/`
- Archive formats handled by `libarchive`, `zlib`, `liblzma`
- Packet capture/replay files written by `src/framework/net/packet_recorder.cpp` and read by `packet_player.cpp` (see `records/`)

**Caching:**
- In-process only (texture atlases per `config.ini` `[graphics]`, sprite/appearance caches). No external cache service.

## Authentication & Identity

**Auth Provider:**
- Server-operator controlled. Account name/email plus password (and optional token/2FA fields) posted to the server's login endpoint from `src/framework/net/httplogin.cpp`; the returned session key is passed into the game protocol handshake.
- Protocol-level RSA/XTEA handshake handled in `src/framework/net/protocol.cpp` with `openssl`.

## Monitoring & Observability

**Error Tracking:**
- None (no Sentry/Crashlytics integration detected).

**Logs:**
- `spdlog` plus the framework logger in `src/framework/core/logger.cpp`; output to console and local log files.

## CI/CD & Deployment

**Hosting:**
- Distributed as downloadable binaries/APK and a static WASM web build (`browser/`, served locally by `tools/emscripten-web-serve.py`).

**CI Pipeline:**
- GitHub Actions - `.github/workflows/ci.yml` orchestrating reusable jobs:
  - `reusable-build-windows.yml`, `reusable-build-linux.yml`, `reusable-build-macos.yml`
  - `reusable-build-android-apk.yml`, `reusable-build-browser.yml`, `reusable-build-docker.yml`
  - `reusable-checks.yml`, `reusable-tests-lua.yml`
  - Maintenance: `update_vcpkg_baseline.yml`, `clean-cache.yaml`, `cron-stale.yml`, `issue.yml`, `pr-labeler.yml`
- Lint reporting via reviewdog (`.reviewdog.yml`)

## Environment Configuration

**Build-time env vars:**
- `VCPKG_ROOT` - vcpkg toolchain location (`CMakeLists.txt`)
- `VCPKG_DEFAULT_TRIPLET` - target triplet
- Android SDK/NDK paths consumed by `setup_android_deps.sh`, `build_luajit_android.sh`

**Secrets location:**
- No `.env` file and no committed secret files detected. CI secrets live in GitHub Actions repository secrets; end-user credentials are entered at runtime and never stored in the repo.

## Webhooks & Callbacks

**Incoming:**
- None. `src/framework/net/server.cpp` provides a local listening socket used for tooling/proxying, not public webhooks.

**Outgoing:**
- None beyond the login HTTP call and Discord RPC IPC.

---

*Integration audit: 2026-07-21*
