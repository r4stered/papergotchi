# Core is a pure function; no file in core names a port

The simulation core is a pure, total function of its arguments rather than an object graph
holding hardware interfaces. Everything the pet does is expressed as
`step(State, Event, Instant) → (State, Effects)`, where `Effects` is a fixed-capacity list
of intents — `Repaint`, `Buzz`, `Persist`, `WakeAt`, `DrawEntropy`. All input arrives
through a single `Event` channel; effect outcomes and hardware failures re-enter as later
events. Frame composition is a second pure function, `render(State, Region, span<byte>)`,
so the expensive pixel work stays out of the constexpr path while core still owns every
pixel and picks the waveform per region.

The consequence that drove the decision: `step` is deterministic. The PRNG lives in
`State`, seeded once at hatch from a `DrawEntropy` effect and persisted in the save, so a
`(seed + event log)` pair reproduces an entire pet life bit-for-bit on the device, in the
simulator, and in a test. On a project whose subject takes two weeks to run once, that is
the difference between balance being testable and being guesswork.

## Considered options

- **Abstract base classes injected into core.** The conventional answer, and virtual
  dispatch costs nothing real here — the display port is called once per frame with a
  rect, not per pixel. Rejected in favour of static dispatch. ESP-IDF's own C++ guide
  notes that vtables live in flash and are unreachable when the flash cache is disabled,
  which makes virtual calls a hazard in interrupt context.
- **Concepts and templates through core.** Chosen first, then narrowed. Templating core on
  a ports bundle spreads instantiation across the whole tree, forces core into headers,
  and makes every test instantiate the world. Once ports were removed from core entirely,
  the question stopped being architectural: dispatch now appears only in each app's shell.
- **`constexpr` instead of templates.** Considered and rejected as a category error —
  `constexpr` is compile-time *evaluation*, templates are compile-time *polymorphism*.
  Removing ports from core is what actually dissolved the problem; `constexpr` then earns
  its keep elsewhere (see Consequences).
- **Fire-and-forget effects with failures handled in the shell.** Rejected: core could
  never surface a persistence failure to the player, and each shell would grow hidden
  state that no test covers.

## Consequences

- **Ports are concepts, checked locally.** Nine narrow ports — Display, Touch, Button, Imu,
  Clock, Storage, Network, Power, Buzzer — each a concept, each mapping onto exactly one
  open research ticket. Every adapter carries `static_assert(DisplayPort<M5PaperDisplay>)`
  in its own TU, so a non-conforming adapter fails there with a readable message.
- **The shell is the only impure code.** Each app writes its own concrete, non-template
  shell (~200 lines) holding adapters by value. There are no templates and no vtables
  anywhere in `core/`.
- **Nothing can be swapped at runtime.** The device and simulator are separate binaries
  with separate concrete types. The fast-forward harness (#20) must therefore be a
  **runtime speed knob on one simulated clock type**, not a second clock type swapped in.
- **`constexpr` does the work templates were doing.** `step` is `constexpr`, so a large
  slice of the simulation is tested by `static_assert` at compile time. Balance constants
  are `consteval`-validated; dither matrices, greyscale ramps and layout rects are computed
  at build time into flash rather than at boot.
- **Time is an argument, never a read.** `Instant` is a parameter of `step`; NTP sync
  arrives as an event (#18). Core never asks anything for the time.
- **Logic migrates inward.** Because adapters are not unit-tested, anything worth testing
  belongs in core — the crash-safe save journal (#5) is core logic and the storage adapter
  is reduced to "write these bytes".
