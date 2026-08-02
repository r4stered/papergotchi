# Toolchain pinned natively via a shared uv venv; CI builds flashable images

Development happens in WSL, so containers buy isolation that is already there and cost a
layer of indirection. Instead the toolchain is pinned natively: **one `.venv` with two
owners**. `uv.lock` pins the build and lint tools by hash — `cmake`, `ninja`,
`clang-format`, `clang-tidy`, `gersemi` — while ESP-IDF's own constraints file owns the IDF
tools — `esptool`, `idf-component-manager`, `esp-idf-kconfig`, `esp-idf-monitor`. The two
sets deliberately do not overlap, so there is nothing to conflict over.

```
IDF_PYTHON_ENV_PATH=$PWD/.venv     IDF_TOOLS_PATH=$PWD/.espressif
make setup → uv sync --inexact && tools/idf.sh install   # IDF at a pinned tag
```

Two mechanisms make the shared venv safe: `IDF_PYTHON_ENV_PATH` points `install.sh` at an
existing venv instead of `~/.espressif/python_env`, and `uv sync --inexact` does not remove
extraneous packages — without it, every sync would prune IDF's packages out.

## Considered options

- **Docker / devcontainers.** Rejected: WSL already provides the isolation, and images add
  a layer between the developer and the build.
- **A Nix flake for everything.** The strongest reproducibility story on paper, and it is
  native. Rejected on a verified fact: `mirrexagon/nixpkgs-esp-dev`, the primary ESP-IDF
  flake, last committed 2026-01-08 and pins ESP-IDF **v5.5.2**. IDF v6.0 shipped
  2026-03-19, so adopting Nix would mean either living on 5.5.2 — losing the `gnu++26`
  default and GCC 15.1 — or owning an IDF derivation the community has not solved.
- **apt-pinned bootstrap with no venv.** Reproducibility would rest on the runner image
  rather than on a lockfile.
- **Catch2 as an Espressif component.** Superseded by doctest (ADR-0006).

## Consequences

- **ESP-IDF's `tools.json` is already a lockfile.** Pinning the IDF git tag pins the Xtensa
  compiler and every other tool binary by exact version and sha256, so the device toolchain
  needs no help from us.
- **Two things cannot be pinned this way: host GCC 15 and host Clang.** No compiler wheels
  exist — the `clang-format` and `clang-tidy` wheels contain only those two binaries. Both
  come from apt. The mitigation is a hard assert in the root `CMakeLists.txt`
  (`CMAKE_CXX_COMPILER_VERSION VERSION_LESS 15` → `FATAL_ERROR`), which turns invisible
  drift into a loud configure-time failure. Pinning the formatter separately from the
  compiler is deliberate: formatting must not shift when the compiler moves.
- **CI must name `ubuntu-26.04` explicitly.** `ubuntu-latest` still resolves to 24.04, which
  tops out at GCC 14; 26.04 carries GCC 15 and is currently a public-preview image.
- **Five jobs, with a host compiler matrix** over GCC 15 and Clang: lint (`clang-format
  --dry-run --Werror`, `gersemi --check`, clang-tidy), sanitized host tests, host-release,
  device build, and a simulator soak. The matrix also produces real evidence for the #7
  toolchain fork instead of speculation.
- **The soak job is the payoff from determinism.** Under `SDL_VIDEODRIVER=dummy`, CI
  fast-forwards N seeded two-week pet lives per push and asserts invariants — no crash, no
  stuck state, all four evolution branches reachable, care-mistake accounting balances.
  This is balance-regression coverage no unit test can provide, and it exists only because
  `step` is deterministic (ADR-0004).
- **Every push produces a flashable image.** `idf.py merge-bin` yields a single file
  flashed at `0x0`; `app.bin` ships alongside it for a future OTA path, plus an
  `idf-size` report diffed against `main` so flash and PSRAM growth are visible per commit
  rather than discovered at 90% full. Tags cut a GitHub Release with the image, a flash
  script and `SHA256SUMS`, versioned from `git describe` — which IDF already embeds in the
  app descriptor.
