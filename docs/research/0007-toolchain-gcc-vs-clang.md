# Toolchain fork: GCC (gnu++26 default) or Clang (required for clang-tidy)?

Research for issue #7. Investigated 2026-08-02 against ESP-IDF v6.0 (released 2026-03-20),
ESP-IDF `master` as it stood on that date, LLVM 20.1.1 / 21.1.0, and pyclang 0.7.0.

**This answer is version-sensitive and will rot.** Every claim below names the version it was
checked against. Claims are marked either _documented_ (Espressif or LLVM states it), _read from
source_ (checked in the named file at the named tag), or _predicted_ (follows from what was read
but was not run).

## The question

ESP-IDF v6.0 builds C++ with `-std=gnu++26` under GCC. `idf.py clang-check` — Espressif's
supported clang-tidy path — is documented as requiring `IDF_TOOLCHAIN=clang`. Linting is a named
requirement for this project, so the two cannot obviously be had at once. Four sub-questions:
does Clang match GCC's language version on Xtensa; is Xtensa Clang production-ready for the
ESP32-S3; can clang-tidy be run off a GCC-produced `compile_commands.json`; and does the pure
core's host build make the whole target-side question small?

## Version pins

From `tools/tools.json` at tag `v6.0`
([source](https://github.com/espressif/esp-idf/blob/v6.0/tools/tools.json)):

| Tool | Version pinned by ESP-IDF v6.0 | Install policy |
| --- | --- | --- |
| `xtensa-esp-elf` (GCC) | `esp-15.2.0_20251204` → GCC 15.2.0 | `always` |
| `esp-clang` (LLVM) | `esp-20.1.1_20250829` → LLVM 20.1.1 | `on_request` |

On `master` as of 2026-08-02 these are `esp-16.1.0_20260609` (GCC 16.1.0) and
`esp-21.1.3_20260408` (LLVM 21.1.3) respectively; the install policies are unchanged. The GCC
toolchain is a required install on every ESP-IDF setup. Clang is not — you must ask for it.

## 1. Does Clang support `gnu++26` on Xtensa?

**The flag: yes, and it is already applied to the Clang build.** _Read from source._
`tools/cmake/build.cmake` at `v6.0` sets the standard for all chip targets without any
`IDF_TOOLCHAIN` condition:

```cmake
if(NOT IDF_TARGET STREQUAL "linux")
    list(APPEND c_compile_options   "-std=gnu23")
    list(APPEND cxx_compile_options "-std=gnu++26")
endif()
```

([build.cmake, v6.0](https://github.com/espressif/esp-idf/blob/v6.0/tools/cmake/build.cmake))
This matches the v6.0 release notes — "Upgraded default C++ standard to gnu++26"
([release notes](https://github.com/espressif/esp-idf/releases/tag/v6.0)) — and the API guide:
"By default, ESP-IDF compiles C++ code using C++26 language standard with GNU extensions
(`-std=gnu++26`) for chip targets"
([C++ Support, v6.0](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-guides/cplusplus.html)).

Clang accepts the spelling. `clang/include/clang/Basic/LangStandards.def` at `llvmorg-20.1.1`
declares `LANGSTANDARD(gnucxx26, "gnu++2c", ...)` with `LANGSTANDARD_ALIAS(gnucxx26, "gnu++26")`;
the same lines are present at `llvmorg-18.1.0`, so any Clang ≥ 18 takes the flag
([LangStandards.def](https://github.com/llvm/llvm-project/blob/llvmorg-20.1.1/clang/include/clang/Basic/LangStandards.def)).
esp-clang 20.1.1 is well past that.

**The standard library: no, it lags by a major version.** _Documented._ esp-clang does not use
LLVM's libc++ for the ESP targets; it bundles libstdc++ built from GCC. The last bump before the
version ESP-IDF v6.0 pins was in `esp-19.1.2_20250211`: "Upgraded `libgcc` and `libstdc++` from
GCC 14.2.0"
([release](https://github.com/espressif/llvm-project/releases/tag/esp-19.1.2_20250211)). The
`esp-20.1.1_20250829` notes list no libstdc++ change
([release](https://github.com/espressif/llvm-project/releases/tag/esp-20.1.1_20250829)). So on
ESP-IDF v6.0 the clang toolchain compiles against **libstdc++ 14.2.0** while the GCC toolchain
compiles against **libstdc++ 15.2.0**. Espressif caught up later — `esp-21.1.3_20260304`
"Upgraded libstdc++ and libgcc to 15.2.0"
([release](https://github.com/espressif/llvm-project/releases/tag/esp-21.1.3_20260304)) — but by
then `master`'s GCC had moved to 16.1.0, so the gap persists.

**Front-end C++26 coverage differs between the two compilers and neither is complete.** Both
vendors describe C++26 as a working draft: Clang documents the mode as
"Working draft for C++2c" and tracks per-paper status
([clang cxx_status](https://clang.llvm.org/cxx_status.html)); GCC does the same
([GCC C++ status](https://gcc.gnu.org/projects/cxx-status.html)). I did not enumerate the delta
between GCC 15.2 and Clang 20.1. If any specific C++26 paper is load-bearing for this project,
check it against both pages before relying on it — the flag being accepted is not the same as the
feature being implemented.

**Trade summary for sub-question 1:** switching to Clang does not cost you the `-std=gnu++26`
flag. It costs you a libstdc++ that is one GCC major version behind, plus an unquantified
front-end feature delta.

## 2. Is Xtensa Clang production-ready for ESP32-S3?

Espressif's own current characterisation is "still under development", and the surrounding
evidence agrees.

- _Documented._ The v6.0 clang-tidy guide carries a warning: "This functionality **and the
  toolchain it relies on** are still under development. There may be breaking changes before a
  final release." The prerequisites section repeats it: "This toolchain is still under
  development. After the final release, you do not have to install them manually."
  ([IDF Clang-Tidy, v6.0](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-guides/idf-clang-tidy.html);
  source at
  [docs/en/api-guides/tools/idf-clang-tidy.rst](https://github.com/espressif/esp-idf/blob/v6.0/docs/en/api-guides/tools/idf-clang-tidy.rst)).
  I checked the same file on `master` on 2026-08-02: the warning is byte-for-byte identical. This
  is a current statement, not a stale one.
- _Read from source._ `esp-clang` is `install: on_request` in `tools.json`, against `always` for
  the GCC toolchain. Espressif does not put it on developers' machines by default.
- _Read from source._ Xtensa is not a core LLVM target. `llvm/CMakeLists.txt` lists it under
  `LLVM_ALL_EXPERIMENTAL_TARGETS` (alongside ARC, CSKY, DirectX, M68k) at `llvmorg-20.1.1`,
  `llvmorg-21.1.0`, and `main` as of 2026-08-02
  ([llvmorg-21.1.0](https://github.com/llvm/llvm-project/blob/llvmorg-21.1.0/llvm/CMakeLists.txt)).
  Building it requires `LLVM_EXPERIMENTAL_TARGETS_TO_BUILD=Xtensa`. Espressif's fork still
  describes itself as "Fork of LLVM with Xtensa specific patches. To be upstreamed."
  ([espressif/llvm-project](https://github.com/espressif/llvm-project)).
- _Documented, live._ esp-idf issue
  [#18460](https://github.com/espressif/esp-idf/issues/18460) — "`vTaskDelete` results in
  IllegalInstruction Exception with clang `-O2`" — was opened 2026-04-14 against IDF v6.0 and is
  still open, last activity 2026-07-30, where a second reporter reproduced it on **ESP32-S3,
  ESP-IDF v6.0, Espressif Clang 20.1.1, `CONFIG_COMPILER_OPTIMIZATION_PERF=y`**. That is our exact
  chip and our exact toolchain version, and it is a wrong-code bug, not a warning.
- _Documented._ esp-idf issue
  [#14358](https://github.com/espressif/esp-idf/issues/14358) — clang link failure
  (`undefined reference to __register_frame_info`) when `CONFIG_COMPILER_CXX_EXCEPTIONS=y` — has
  been open since 2024-08-12. Relevant to a C++ project even if we disable exceptions, because it
  shows the C++ runtime integration is where the seams are.

### The finding that outweighs the rest: Clang loses Picolibc on v6.0

ESP-IDF v6.0 "Changed default libc from Newlib to Picolibc"
([release notes](https://github.com/espressif/esp-idf/releases/tag/v6.0)). _Read from source_,
`components/esp_libc/Kconfig` at `v6.0`:

```kconfig
choice LIBC
    prompt "LibC to build application with"
    default LIBC_NEWLIB if IDF_TOOLCHAIN_CLANG
    default LIBC_PICOLIBC

    config LIBC_NEWLIB
        bool "NewLib"
    config LIBC_PICOLIBC
        bool "Picolibc"
        depends on !IDF_TOOLCHAIN_CLANG
endchoice
```

([esp_libc/Kconfig, v6.0](https://github.com/espressif/esp-idf/blob/v6.0/components/esp_libc/Kconfig))

Setting `IDF_TOOLCHAIN=clang` on ESP-IDF v6.0 makes Picolibc **unselectable** and silently drops
you back to Newlib. That is not a lint-versus-language-version trade; it is a different C library
under the whole firmware, with different sizes, different `stdio`, and different behaviour.

The same pattern shows up across v6.0's build files: `components/soc/project_include.cmake`,
`components/esp_libc/project_include.cmake` and the root `CMakeLists.txt` all wrap their flag
plumbing in `if(CONFIG_IDF_TOOLCHAIN_GCC)` with an `else() # TODO IDF-14338` fallback. v6.0
reworked compiler-flag handling into response files, and the Clang path has not fully caught up.

**Conclusion for sub-question 2:** on ESP-IDF v6.0, for ESP32-S3, Clang is a preview path.
Espressif says so, ships it opt-in, has an open `-O2` miscompilation reproduced on this exact
chip and version, and has not wired v6.0's own default libc up to it.

## 3. Can we keep GCC and run clang-tidy off `compile_commands.json`?

This is the interesting one, because parts of the machinery for it already exist — and one part
that used to work now looks broken.

**What Espressif ships.** `idf.py clang-check` is provided by `pyclang`, which is listed in
`tools/requirements/requirements.core.txt` at `v6.0`, so it is installed by default; only the
`esp-clang` binaries need `idf_tools.py install esp-clang`. In pyclang 0.7.0 the IDF extension
runs this chain
([pyclang/idf_extension.py](https://github.com/espressif/clang-tidy-runner/blob/main/pyclang/idf_extension.py)):

```python
runner.idf_reconfigure().filter_cmd().remove_command_flags().run_clang_tidy().remove_color_output()
```

**`remove_command_flags` is exactly the flag-filtering trick, already written.** _Read from
source_, `pyclang/runner.py` 0.7.0:

```python
GCC_FLAGS_MAPPING = {
    '-fstrict-volatile-bitfields': '',
    '-fno-tree-switch-conversion': '',
    '-fno-test-coverage': '',
    '-mlongcalls': '-mlong-calls',
}
```

plus regex removal of `-fmacro-prefix-map=`, `-fdebug-prefix-map=`, `-ffile-prefix-map=`. It is a
plain textual find-and-replace over `build/compile_commands.json`. The `-mlongcalls` →
`-mlong-calls` entry is itself the evidence that Clang rejects the GCC spelling — Espressif would
not have written the mapping otherwise.

**The tooling does not hard-require `IDF_TOOLCHAIN=clang` — it only warns.** _Read from source_,
same file: `check_clang_toolchain()` reads `IDF_TOOLCHAIN` out of `build/CMakeCache.txt` and, if
it is anything other than `clang`, emits `log.warn(...)`. It does not abort. So the documented
"only clang based toolchain is currently supported" is a support statement, not an enforced
precondition.

**What _is_ hard-enforced is esp-clang.** `check_esp_clang()` resolves `clang-tidy` on `PATH` and
calls `log.die(...)` unless the path contains `esp-clang` (or the pre-5.0 `xtensa-esp32-elf-clang`).
A desktop Clang is refused outright. This is correct, not arbitrary: LLVM's JSON compilation
database infers the target triple from `argv[0]`'s prefix and only applies it if that target is
registered in the binary
(`ToolChain::getTargetAndModeFromProgramName` →
`llvm::TargetRegistry::lookupTarget`,
[ToolChain.cpp, llvmorg-20.1.1](https://github.com/llvm/llvm-project/blob/llvmorg-20.1.1/clang/lib/Driver/ToolChain.cpp)).
With `xtensa-esp32s3-elf-gcc` as `argv[0]` and a stock desktop Clang, `xtensa-esp32s3-elf` is not
registered, no `--target` is applied, and ESP-IDF headers get parsed as if for the host triple —
wrong pointer model, wrong predefined macros, no Xtensa intrinsics. So "run clang-tidy with the
clang I already have" is not on the table for target code, only "run esp-clang's clang-tidy".

**The concrete failure mode, and it is new in v6.0.** _Read from source, outcome predicted._ In
ESP-IDF v6.0 the Xtensa flags are no longer written into `compile_commands.json` as literal text.
`components/soc/project_include.cmake` adds them through the response-file mechanism, gated on GCC:

```cmake
if(CONFIG_IDF_TOOLCHAIN_GCC)
    ...
    if(CONFIG_IDF_TARGET_ARCH_XTENSA)
        idf_toolchain_add_flags(COMPILE_OPTIONS "-mlongcalls"
                                                "-fno-builtin-memcpy"
                                                "-fno-builtin-memset"
                                                "-fno-builtin-bzero")
```

([soc/project_include.cmake, v6.0](https://github.com/espressif/esp-idf/blob/v6.0/components/soc/project_include.cmake))

and `tools/cmake/toolchain.cmake` points the compiler flags at those files:

```cmake
set(CMAKE_C_FLAGS   "@\"${IDF_TOOLCHAIN_BUILD_DIR}/cflags\"" ...)
set(CMAKE_CXX_FLAGS "@\"${IDF_TOOLCHAIN_BUILD_DIR}/cxxflags\"" ...)
```

([toolchain.cmake, v6.0](https://github.com/espressif/esp-idf/blob/v6.0/tools/cmake/toolchain.cmake))

Meanwhile LLVM's JSON compilation database expands response files when it loads the file —
`inferTargetAndDriverMode(inferMissingCompileCommands(expandResponseFiles(...)))` in
`JSONCompilationDatabasePlugin::loadFromDirectory`
([JSONCompilationDatabase.cpp, llvmorg-20.1.1](https://github.com/llvm/llvm-project/blob/llvmorg-20.1.1/clang/lib/Tooling/JSONCompilationDatabase.cpp)).

Put together: under a GCC-configured v6.0 build, `-mlongcalls` lives in `build/toolchain/cflags`,
pyclang's textual substitution scans `compile_commands.json` and finds nothing to rewrite, and
clang-tidy then pulls `-mlongcalls` back in from the response file when it loads the database. The
filtering step is bypassed. **I did not run this** — no ESP-IDF installation here — so treat it as
a strong prediction from the sources, not an observation. But it is the specific thing to test
first if anyone tries this route, and it explains why the flag-filtering idea sounds sounder than
it is on v6.0 specifically.

Note also that on the *supported* Clang path this problem does not arise: the entire
`-mlongcalls` block is inside `if(CONFIG_IDF_TOOLCHAIN_GCC)`, so those flags are never added.
Espressif fixed the "GCC flags leak into the clang-tidy run" problem by not generating them, not
by filtering them better.

**A second sign of decay.** `idf_clang_tidy` still exposes `--xtensa-include-dir` (defaulting to
`/opt/espressif/xtensa-esp32-elf-clang/xtensa-esp32-elf/include/`), and ESP-IDF's own CI passes it
([.gitlab/ci/static-code-analysis.yml, v6.0](https://github.com/espressif/esp-idf/blob/v6.0/.gitlab/ci/static-code-analysis.yml)).
In pyclang 0.7.0 the value is assigned to `self.xtensa_include_dir` in `Runner.__init__` and
**never read anywhere else** — verified by grepping the whole package. The escape hatch for
"clang-tidy cannot find the Xtensa headers GCC assumes" is now dead code. That the option existed
at all says the header problem was real; that it is dead says nobody is exercising the GCC-side
path.

**And Espressif's CI does not exercise it.** The `clang_tidy_check` job sets
`IDF_TOOLCHAIN: clang` explicitly before running `idf_clang_tidy` (same CI file). There is no
job that runs clang-tidy over a GCC-configured build. What Espressif does test on the GCC side is
GCC's own analyser: a `gcc_static_analyzer` job that builds `hello_world` with
`CONFIG_COMPILER_STATIC_ANALYZER=y`.

**Conclusion for sub-question 3:** the have-both option is not blocked by policy — pyclang only
warns — but on ESP-IDF v6.0 the mechanism it depends on has been undercut by the response-file
refactor, its Xtensa-header escape hatch is dead, and Espressif tests neither. It is unverified at
best and probably broken. It is not a foundation to build a lint requirement on.

## 4. Does the host build make the target-side question much less important?

Yes — for the code that matters most, almost entirely.

The pure core is the part of this project worth linting: it holds the care loop, the meter and
weight arithmetic, the stage and lifespan-band logic, the care-mistake accounting that ADR-0003
requires to be reconstructible per stage. It is also, by design (issue #8), free of ESP-IDF types.
That means the host build is an ordinary CMake C++ project, and clang-tidy over it is an ordinary
clang-tidy run: plain `CMAKE_EXPORT_COMPILE_COMMANDS=ON`, a system Clang, a host triple, no
`-mlongcalls`, no response files, no Xtensa headers, no `--target` inference, no esp-clang. None
of sections 1–3 apply.

Two caveats, both real:

- Host clang-tidy sees the core, and nothing else. The ports and the app layer — the code that
  touches M5GFX, NVS, FreeRTOS, the GT911 — go unlinted by it. The mitigation is architectural
  rather than tooling: keep the ports thin enough that there is little logic in them to lint, which
  is the same discipline the fast-forward harness already demands.
- The host build will not necessarily be `gnu++26`. If the core is built through ESP-IDF's Linux
  target, `__linux_build_set_lang_version` in `build.cmake` walks
  `gnu++26 gnu++2b gnu++20 gnu++2a gnu++17 gnu++14` and takes the first the host compiler accepts.
  If the core is built with plain CMake instead, we set the standard ourselves and should pin it to
  `gnu++26` explicitly so the host and target builds agree.

## Recommendation

**Build with GCC. Do not set `IDF_TOOLCHAIN=clang`. Get clang-tidy from the host build of the
pure core, using a normal desktop Clang against a plain-CMake `compile_commands.json`.**

The reasoning is not primarily about lint. It is that on ESP-IDF v6.0, choosing Clang means
choosing Newlib over Picolibc (`components/esp_libc/Kconfig` forbids the combination), a
libstdc++ one GCC major behind, an LLVM backend upstream still classes as experimental, and an
open `-O2` miscompilation reproduced on ESP32-S3 at exactly our IDF and toolchain versions. That
is a large bill for a linter, on a project whose firmware has to survive a fortnight of
uninterrupted uptime per pet.

Meanwhile the lint we actually want is over the core, and the core is host-buildable. Set the
host build's standard to `gnu++26` explicitly so the two builds agree, add a `.clang-tidy` at the
repo root, and run clang-tidy in CI over the host compilation database. This is the boring path
and it works today with no Espressif-specific machinery.

**Honest trade-offs.**

- We give up static analysis of the ports and app layers. Partial mitigations: enable
  `CONFIG_COMPILER_STATIC_ANALYZER=y` for a CI-only build (this is precisely what ESP-IDF's own
  `gcc_static_analyzer` job does), and keep the ports thin. Neither is as good as clang-tidy.
- We give up clangd-quality editor tooling for the target build. esp-clang does ship clangd
  (bundled since `esp-17.0.1_20240419`, separately packaged since `esp-21.1.3_20260304`), and
  installing esp-clang for *editor* use does not require switching `IDF_TOOLCHAIN`. That is a
  cheap, low-risk thing to do independently of this decision.
- We are betting that the core stays genuinely pure. If IDF types leak into it, the host lint
  degrades silently. Issue #8 should make that boundary explicit and CI should enforce it by
  building the core standalone.
- **This decision should be revisited**, not treated as permanent. The trigger conditions are
  concrete: Picolibc becoming selectable under Clang (watch
  `components/esp_libc/Kconfig` and the `TODO IDF-14338` markers), issue #18460 closing, and the
  clang-tidy guide's "still under development" warning being removed.

**If target-side lint later becomes non-negotiable**, the route to try is a *second, throwaway*
build directory — `idf.py -B build-clang -DIDF_TOOLCHAIN=clang clang-check` — that never produces
shipped firmware, accepting that its Newlib-vs-Picolibc divergence makes the lint slightly
unfaithful to the real build. Do **not** reach for the `compile_commands.json` flag-filtering
trick; see section 3. Either way, prove it works on a hello-world before depending on it.

## What remains uncertain

- **The section 3 failure mode is predicted, not observed.** I could not run an ESP-IDF build. The
  chain — `-mlongcalls` in the response file, pyclang substituting only over the JSON, LLVM
  expanding response files at database load — is read from source at pinned versions, but the
  end-to-end outcome is inference. Someone with a working ESP-IDF v6.0 install can settle it in
  ten minutes: configure `hello_world` with GCC, run `idf.py clang-check`, and grep `warnings.txt`
  for `unknown argument`.
- **Whether the two-build-directory approach works at all** is untested. ESP-IDF's toolchain
  mismatch check (`targets.cmake`) fatals on a cache/environment mismatch within one build
  directory; whether two separate directories coexist cleanly across `sdkconfig` I did not verify.
- **The exact C++26 front-end delta between GCC 15.2 and Clang 20.1** was not enumerated. If a
  specific C++26 paper turns out to be load-bearing, check both vendors' status pages directly.
- **Whether esp-clang's Newlib-only constraint has consequences beyond libc choice** — reentrancy,
  TLS, `getreent()` compatibility — was not investigated. The `LIBC_PICOLIBC_NEWLIB_COMPATIBILITY`
  option in `esp_libc/Kconfig` hints that the two libcs are not interchangeable at the ABI level.
- **IDF-14338**, the internal ticket referenced by the `TODO` markers gating Clang out of v6.0's
  flag plumbing, is a Jira ID with no public tracker entry. There is no way from outside to tell
  whether Clang parity is scheduled for v6.1 or is open-ended.
- **Whether `master`'s newer esp-clang 21.1.3 fixes any of this** was not checked beyond release
  notes. The Picolibc `depends on !IDF_TOOLCHAIN_CLANG` line and the clang-tidy "under
  development" warning were both still present on `master` on 2026-08-02.

## Sources

Primary, all read directly:

- ESP-IDF v6.0 release notes — https://github.com/espressif/esp-idf/releases/tag/v6.0
- `tools/cmake/build.cmake` @ v6.0 — https://github.com/espressif/esp-idf/blob/v6.0/tools/cmake/build.cmake
- `tools/cmake/toolchain.cmake`, `toolchain-clang-esp32s3.cmake`, `targets.cmake` @ v6.0
- `tools/tools.json` @ v6.0 and @ master — https://github.com/espressif/esp-idf/blob/v6.0/tools/tools.json
- `components/esp_libc/Kconfig` @ v6.0 — https://github.com/espressif/esp-idf/blob/v6.0/components/esp_libc/Kconfig
- `components/soc/project_include.cmake`, `components/esp_libc/project_include.cmake` @ v6.0
- `.gitlab/ci/static-code-analysis.yml` @ v6.0 — https://github.com/espressif/esp-idf/blob/v6.0/.gitlab/ci/static-code-analysis.yml
- IDF Clang-Tidy guide @ v6.0 and @ master — https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-guides/idf-clang-tidy.html
- C++ Support guide @ v6.0 — https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-guides/cplusplus.html
- IDF Tools guide @ v6.0 — https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-guides/tools/idf-tools.html
- espressif/clang-tidy-runner (pyclang 0.7.0, commit `3a0fbd5`, 2026-07-13) — `pyclang/runner.py`,
  `pyclang/idf_extension.py`, `pyclang/scripts/idf_clang_tidy.py` —
  https://github.com/espressif/clang-tidy-runner
- espressif/llvm-project releases — https://github.com/espressif/llvm-project/releases
- espressif/esp-toolchain-docs, `clang/esp-idf-app-clang-build.md` —
  https://github.com/espressif/esp-toolchain-docs/blob/main/clang/esp-idf-app-clang-build.md
- LLVM `llvm/CMakeLists.txt` @ llvmorg-20.1.1, llvmorg-21.1.0, main
- LLVM `clang/include/clang/Basic/LangStandards.def` @ llvmorg-18.1.0, llvmorg-20.1.1
- LLVM `clang/lib/Tooling/JSONCompilationDatabase.cpp`, `clang/lib/Driver/ToolChain.cpp` @ llvmorg-20.1.1
- Clang C++ status — https://clang.llvm.org/cxx_status.html
- GCC C++ status — https://gcc.gnu.org/projects/cxx-status.html
- esp-idf issues #18460, #14358 — https://github.com/espressif/esp-idf/issues/18460
