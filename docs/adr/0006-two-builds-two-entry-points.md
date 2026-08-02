# Two builds from one tree, with two honest entry points

`idf.py` wraps CMake rather than being driven by it, and it does not read
`CMakePresets.json`. Rather than paper over that, the repo admits it: the **root is a plain
CMake project** governed by presets, `apps/device` is the **ESP-IDF project** driven by
`idf.py -C`, and a root `Makefile` is the single discoverable entry point wrapping both.
Shared modules carry one `CMakeLists.txt` that branches on `ESP_PLATFORM` —
`idf_component_register` under IDF, `add_library` otherwise — so there is exactly one
source list per module and no shims to drift.

```
CMakeLists.txt  CMakePresets.json  Makefile  .clang-format  .clang-tidy
core/  ports/  adapters/esp/  adapters/sim/
apps/device/ (IDF project, sdkconfig.defaults)   apps/sim/ (SDL)
tests/golden/  tests/replays/  tests/integration/
```

## Considered options

- **A `device` preset that bypasses `idf.py`.** Plain CMake can be pointed straight at the
  IDF toolchain file, giving one uniform interface. Rejected: it steps around the supported
  entry point, so IDF version bumps that touch `project.cmake` can break it, and
  `sdkconfig` generation becomes hand-managed.
- **Repo root as the IDF project.** Most familiar to an IDF developer arriving cold, but
  presets could not govern the root, `sdkconfig` would clutter it, and the host build and
  simulator would become second-class citizens in a subdirectory.
- **`libs/` plus thin component shims under `apps/device/components/`.** Keeps every IDF
  concept inside the device app, at the cost of a second source list per module that drifts
  the moment a file is added.
- **Catch2 for tests.** Superseded by **doctest**: it is designed to live inside production
  sources (matching co-located tests), compiles far faster, and `DOCTEST_CONFIG_DISABLE`
  strips it to nothing for the device build. Verified compiling under `gnu++26`.
- **Tests under the `linux` IDF target with CMock.** Rejected — a second build system whose
  mocks verify that the adapter matches *our idea* of the IDF API rather than the real one.

## Consequences

- **Presets are host-only and say so.** `host-debug` (sanitizers, `-Werror`),
  `host-release`, `host-tidy` (Clang + clang-tidy over `core/` and `ports/`), and matching
  ctest presets. All export `compile_commands.json`.
- **`host-tidy` largely pre-empts #7.** clang-tidy over core works today with an ordinary
  desktop Clang and no Xtensa Clang risk, so the toolchain fork now only affects adapter
  code.
- **Tests are host-only, in three tiers** — `static_assert`, doctest, and replay of
  recorded event logs. **Adapters are deliberately not unit-tested**; the rule is that
  logic worth testing gets pushed into core (ADR-0004).
- **Unit tests are co-located** as `*_test.cpp`, globbed only when `NOT ESP_PLATFORM` so the
  device build never sees them. `tests/` holds golden images, replay logs and integration
  tests.
- **Dependencies are declared and pinned, never installed.** FetchContent pinned to release
  tags (SDL, doctest) on the host; `idf_component.yml` on the device. `core/` and `ports/`
  depend on nothing on either build.
- **Formatting is enforced by make targets plus an opt-in pre-commit hook**, with the
  `clang-format` version pinned — unpinned formatters produce phantom diffs. `gersemi`
  formats the CMake files, which `idf.py` gives no help with either.
