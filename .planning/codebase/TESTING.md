# Testing Patterns

**Analysis Date:** 2026-07-21

## Test Framework

**Runner:**
- GoogleTest (`GTest::gtest` + `GTest::gtest_main`), pulled via vcpkg
- Registered with CTest through `gtest_discover_tests()` in `tests/CMakeLists.txt`
- Gated behind the CMake option `OTCLIENT_BUILD_TESTS` (default `OFF`, `CMakeLists.txt:29`)
  and the vcpkg manifest feature `tests` (`vcpkg.json`). Configuring with tests ON but without the
  feature is a hard `FATAL_ERROR` (`CMakeLists.txt:151`).
- Tests are **not built on Android** (`CMakeLists.txt:159`)

**Assertion Library:** GoogleTest macros only. No mocking library (`gmock` is not a dependency).

**Run Commands:**
```bash
# Configure + build with tests (use a *-tests preset from CMakePresets.json)
cmake --preset linux-tests          # or windows-tests / windows-asan-tests
cmake --build build/linux-tests

# Run all tests
ctest --test-dir build/linux-tests --output-on-failure --parallel 2
```
CI runs exactly this on the `linux-debug` matrix leg (`.github/workflows/reusable-build-linux.yml:115`).

## Test File Organization

**Location:** separate top-level `tests/` tree mirroring `src/` subsystem names — never co-located.

**Naming:** `<subject>_test.cpp`.

**Structure:**
```text
tests/
├── CMakeLists.txt          # defines otclient_add_gtest(), adds subdirs
├── map/
│   ├── CMakeLists.txt
│   └── map_spectators_test.cpp
├── otml/
│   ├── CMakeLists.txt
│   ├── otml_alias_test.cpp
│   └── ampersand_dual_purpose.otui   # fixture asset
└── stdext/
    ├── CMakeLists.txt
    └── string_encoding_test.cpp
```

## Adding a New Test

1. Create `tests/<area>/<subject>_test.cpp`
2. Create `tests/<area>/CMakeLists.txt` calling the shared helper:
   `otclient_add_gtest(<target_name> <subject>_test.cpp)`
3. Add `add_subdirectory(<area>)` at the bottom of `tests/CMakeLists.txt`

`otclient_add_gtest()` (`tests/CMakeLists.txt`) centralizes everything: C++20, the `src/framework/pch.h`
precompiled header, linking against `otclient_core`, the feature-macro set
(`CLIENT`, `FRAMEWORK_GRAPHICS`, `FRAMEWORK_NET`, `FRAMEWORK_SOUND`, `FRAMEWORK_XML`), ASAN flags,
MSVC `/utf-8` + runtime library selection, Android `log`/`c` linkage, and `gtest_discover_tests`.
Never hand-roll `add_executable` for a test.

## Test Structure

Bare `TEST(Suite, Case)` macros — no fixture classes (`TEST_F`) are used anywhere.
Suite names are PascalCase subject names; case names are PascalCase behaviour descriptions.

```cpp
// tests/otml/otml_alias_test.cpp
TEST(OTMLAlias, ResolvesRootAliases)
{
    const std::string document = R"(
&primaryColor: #112233

TestStyle < UIWidget
  color: $primaryColor
)";

    std::istringstream stream(document);
    const auto doc = OTMLDocument::parse(stream, "otml_alias_test");

    const auto style = findStyleByTag(doc, "TestStyle < UIWidget");
    ASSERT_NE(nullptr, style);
    EXPECT_EQ("#112233", style->valueAt("color"));
}
```

**Patterns:**
- Arrange / act / assert separated by blank lines; no setup or teardown hooks
- `ASSERT_*` for preconditions whose failure makes the rest meaningless (null checks); `EXPECT_*` for
  every subsequent assertion so one test reports all failures
- Expected value first in `EXPECT_EQ` in the OTML/map tests; `stdext` tests put the call first —
  match the surrounding file
- Group many related assertions into one `TEST`, separated by blank lines by category
  (valid inputs / invalid inputs / edge cases) — see `TEST(StringEncoding, Utf8Validation)`
- Test-local helpers live in an anonymous `namespace { ... }` at the top of the file
  (`findStyleByTag` in `tests/otml/otml_alias_test.cpp`)

## Mocking

No mocking framework. Two substitution techniques are used instead:

**1. Hand-written derived stubs** — subclass the real type and override the hooks:
```cpp
// tests/map/map_spectators_test.cpp
class DummyCreature final : public Creature
{
public:
    DummyCreature() { setRemovedSilently(false); }
    void onPositionChange(const Position&, const Position& oldPos) override { setOldPositionSilently(oldPos); }
    void onDisappear() override { setRemovedSilently(true); setOldPositionSilently({}); }
    ThingType* getThingType() const override
    {
        static ThingType type;
        static const bool initialized = [] { type.m_null = false; /* ... */ }();
        return &type;
    }
};
```
Note the `static` + immediately-invoked-lambda idiom for one-time initialization of shared fixtures.

**2. Access-specifier override** for engine internals that expose no seam:
```cpp
#define private public
#define protected public
#include "client/map.h"
#include "client/creature.h"
#undef protected
#undef private
```
This is confined to `tests/map/map_spectators_test.cpp` and must wrap only the project includes —
never leave the macros defined across framework or standard headers.

**What to mock:** graphics/network/asset-loading dependencies that the unit under test only touches
incidentally. **What NOT to mock:** pure logic (`stdext`, OTML parsing) — feed it real strings.

## Fixtures and Test Data

- Inline raw string literals (`R"( ... )"`) are the default for parser input, kept in the test body
- On-disk fixtures sit next to the test that uses them: `tests/otml/ampersand_dual_purpose.otui`
- Documents parsed from memory pass a synthetic source name to `OTMLDocument::parse(stream, "otml_alias_test")`
  so failure messages point back at the test

## Coverage

No coverage tooling, threshold, or reporting is configured. C++ coverage is limited to three
subsystems (map spectators, OTML aliases, stdext string encoding); the client, framework graphics,
net, sound, and the entire Lua module layer have no unit tests.

## Test Types

**Unit tests:** the only C++ test type. Pure-function and parser-level, plus one integration-flavoured
map/spectator test that constructs real `Map`, `Tile`, and `Creature` objects.

**Lua tests:** no behavioural test suite. Lua is validated by **syntax compilation only** —
`.github/workflows/reusable-tests-lua.yml` walks `data`, `modules`, and `mods`, runs
`luajit -b "$file" /dev/null` on every `.lua`, and fails the job if any file fails to compile.
The workflow also hard-fails if any of those three roots is missing.

**Static analysis as a gate:** `xmllint` on `data/**/*.xml` and `luac` on all Lua are error-level
in `.reviewdog.yml`; `cppcheck` and `luacheck` are advisory.

## Common Patterns

**Sanitizers:** `ASAN_ENABLED` wires `/fsanitize=address` (MSVC) or `-fsanitize=address` (GCC/Clang)
into every gtest target. The `windows-asan-tests` preset (`CMakePresets.json`) builds release + ASAN + tests.

**Round-trip testing** for encoders/decoders:
```cpp
const std::string ascii = "Hello World 123!";
EXPECT_EQ(stdext::latin1_to_utf8(stdext::utf8_to_latin1(ascii)), ascii);
```

**Exhaustive range sweeps** rather than sampled cases:
```cpp
std::string latin1All;
latin1All.reserve(256);
for (int i = 0; i < 256; ++i)
    latin1All += static_cast<char>(i);
EXPECT_TRUE(stdext::is_valid_utf8(stdext::latin1_to_utf8(latin1All)));
```

**Binary/edge-case inputs** written as explicit escapes with adjacent-literal separation to avoid
hex-escape greediness: `"\xDC""ber"`, `std::string("\x00", 1)`.

---

*Testing analysis: 2026-07-21*
