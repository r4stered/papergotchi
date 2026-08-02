# Core never allocates, and owns a hand-written fixed-capacity vector

Core allocates nothing after construction. Every buffer it needs is injected by the app —
the 960×540 4bpp framebuffer (~259 KB) arrives as a `std::span` that the device places in
PSRAM and the simulator places on the heap. Internally, core uses fixed-capacity containers
whose capacities are compile-time constants: care log depth, effect list length, dirty-rect
count. Running out of space is a domain event, not an allocation failure, which matters
because ESP-IDF disables exceptions by default and there is no `bad_alloc` to catch.

That container is **written in-repo**, roughly forty lines, as a strict subset of the
`std::inplace_vector` API. This looks like reinvention and is not: neither the standard
library nor the obvious third-party libraries can supply a constexpr fixed-capacity vector
today, and `step` being `constexpr` is load-bearing (ADR-0004).

## Considered options

- **`std::inplace_vector`.** The right answer, unavailable. ESP-IDF v6.0 ships GCC 15.1.0
  and `__cpp_lib_inplace_vector` is undefined there; it lands in libstdc++ 16. Verified
  against the local GCC 15.2 host toolchain, which is effectively the same library.
- **ETL (Embedded Template Library).** The mature embedded answer — heap-free, MIT, widely
  used. Rejected on a verified fact: ETL targets C++03 and its containers use uninitialised
  storage with placement new, so they are **not constexpr-constructible**. Checked against
  ETL 20.48.1: `etl::vector<int,8> v;` inside a `constexpr` function gives
  `error: call to non-'constexpr' function 'etl::vector<...>::vector()'`. Adopting it would
  cost the entire compile-time test tier.
- **`boost::container::static_vector`.** Same defect, plus Boost on ESP-IDF.
- **Bare `std::array` plus an explicit count.** Trivially constexpr and needs no type at
  all, but scatters manual index bookkeeping through core and loses bounds discipline.
- **Allocate freely, or through an injected `pmr::memory_resource`.** Rejected: with
  exceptions off a failed allocation aborts, heap fragmentation over a two-week uptime is
  unbounded, and the simulator would never feel the ceiling the device has.

## Consequences

- **Do not "fix" this by adding ETL.** The constexpr requirement is the reason, and it is
  verifiable in one compile.
- **There is a scheduled exit.** When the toolchain reaches GCC 16, delete the header and
  replace it with `using fixed_vector = std::inplace_vector<T, N>;`. Call sites don't
  change, because the API is deliberately a subset — except `try_push_back`, which returns
  `false` when full rather than throwing, and suits exceptions-off anyway.
- **Capacities are part of the design.** Care log depth and effect-list length are
  compile-time constants that the simulator exercises identically to the device, so the
  ceiling is hit in testing rather than in the field.
- **Core has zero dependencies** on either build. `std::variant`, `std::expected` and
  `std::span` are all verified working under `-std=gnu++26 -fno-exceptions -fno-rtti`.
