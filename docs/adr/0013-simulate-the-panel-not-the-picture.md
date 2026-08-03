# Simulate the panel, not the picture

Core owns every pixel, so the simulator never draws the pet. `render(State, Region, span<byte>)`
hands it 16-grey bytes and its entire job is to be the **panel**: a per-pixel waveform state machine
walking M5GFX's own LUTs, an optical model turning accumulated drive into a reflectance, and a shell
that skips **rest** without ever compressing **presence**. One binary serves the window, the CI soak
and the replay runner; `SDL_VIDEODRIVER=dummy` is the only thing that separates them.

The ticket asked where the fidelity line sits, from "draw the framebuffer instantly at full contrast"
to simulating quantisation, latency, the flash and ghosting. The line is at the far end, and it is
cheaper there than it looks: note [`0009`](../research/0009-refresh-strategy.md) §3.1 decoded the
encoding (`0` end, `1` darken, `2` lighten, `3` no-op; 16 two-bit fields indexed by target grey),
§3.3–§3.4 documented the per-pixel state machine, and the whole waveform set is **85 steps across
four modes**. Simulating it collapses three separate fakes — A2's reduced contrast, refresh latency,
accumulated ghosting — into **one model with one tunable constant**, and quantisation stops being a
decision at all because core already emits panel-format bytes.

## Considered options

- **Instant present at full contrast.** The naive simulator, and the one the ticket warns about: it
  makes every game feel good and ship broken. Rejected outright — panel latency is what shapes game
  feel, so a simulator that hides it validates a product that does not exist.
- **Hold-and-swap: block for the intent's modelled duration, then swap the image.** Genuinely
  viable, ~15 lines, and it captures the constraint that actually bites — never re-drive a region
  before its waveform completes (note 0009 §7.2 rule 5). Rejected because reduced contrast,
  ghosting and the `Clear` flash would then each need their own invented approximation, and three
  hand-tuned lies cost more than one physical model and agree with each other less.
- **A second, SDL-free app for the headless soak.** Faster to run and simpler to reason about.
  Rejected: it makes CI green while the artefact a human actually looks at goes untested, and it
  doubles the shell that has to stay honest.
- **A uniform speed multiplier over everything.** One knob, trivially honest about time. Rejected
  because at 100× a 407 ms `Clear` lasts 4 ms and an `Animate` run is invisible — you can
  fast-forward or you can look at something, never both. Rest and presence are compressed
  separately instead.
- **Mouse position straight to a tilt vector.** Ten minutes' work, and actively harmful: absolute
  position, instant response, no noise, no drift, no arm fatigue, and no screen being tilted away
  from your own eyes. A maze tuned against it is a different game.
- **Real sockets for the `Network` adapter.** The host has a working stack and the cloud target is
  real. Rejected on determinism: latency, ordering and failure would become properties of the room,
  the **replay log** would stop reproducing, and the soak's flakiness would be blamed on core. It is
  also the wrong tool — the happy path is the one case that needs no test.
- **Process-per-wake, `fork`/`exec` per ambient boot.** The literal reading of "no in-RAM state
  between wakes", and the only thing that catches a `static` inside core. Rejected: it destroys the
  interactive window, and the thing it uniquely buys is checkable at build time by inspecting core's
  object files for mutable static storage — faster, more precise, and it fails at the commit that
  introduced it.
- **Simulator-generated golden images.** One tool instead of two. Rejected: it couples the most
  stable thing in the system to the least stable, so every golden would churn the day the optical
  ramp gets measured values.

## Consequences

- **The simulator knows something the device cannot afford to know, and must not act on it.**
  [ADR-0008](0008-refresh-policy.md) considered per-pixel impulse tracking (Nook's US9613599B2) and
  rejected it *as an implementation* — "it needs a per-pixel side buffer and a waveform LUT indexed
  by charge history, and M5GFX exposes neither." The simulator now has exactly that. So **area
  actuates and impulse observes**: the sim decides when to `Clear` by the identical area budget the
  device uses, and tracks per-pixel residual error alongside it as a read-only instrument. Clearing
  on impulse would playtest a device that deghosts on a signal the real one cannot compute.
- **That observer is the only check ADR-0008's borrowed constant has ever had.** That ADR says
  outright the ~400-fast-updates figure is "Rockchip's `20 × screen_area` constant divided by our
  window fraction — a different panel, a waveform that may not be balanced… the number is a
  placeholder until measured." The soak can now bound peak residual at the instant each `Clear`
  lands, across N seeded fortnights. It is a model checking a heuristic, not hardware checking
  either — but it gives the direction of the error, which is more than existed before.
- **The waveform tables become a second copy of the device's behaviour, so they are generated and
  diffed.** The LUTs are file-static in `Panel_EPD.cpp` and unreachable from the host build. A tool
  under `tools/` extracts them from the pinned M5GFX source into a committed header; CI regenerates
  and diffs. This turns ADR-0008 rule 10 — *"pin the M5GFX version exactly, and re-decode the LUTs
  on every upgrade"* — from a discipline into a build failure, at the M5GFX bump, while a human is
  already looking. **A parse failure must fail the build and never fall back to the committed
  copy.**
- **`ports/` gains a shared policy type, and `CONTEXT.md`'s "adapters are deliberately dumb" is
  amended to admit it.** The **ghosting budget**, per-region serialisation, white-frame bracketing
  and 4-pixel region rounding are identical physics on both builds. They cannot live in core
  (waveform knowledge, forbidden by ADR-0008) and duplicating them in two adapters recreates exactly
  the drift this ticket exists to prevent. `ports/` already depends on nothing on either build
  ([ADR-0006](0006-two-builds-two-entry-points.md)), so a pure, allocation-free, host-tested type
  sits there beside the concepts. The gate is narrow: **identical on device and simulator, and
  derived from the panel rather than the platform.**
- **Compression applies to rest, never to presence, and the panel clock is not connected to the
  knob.** The shell does not sleep through rest — it sets the clock to the **armed instant** and
  boots, which is what the device does. Everything visible runs true: `Animate` at ~88 ms, `Clear`
  at ~407 ms, the **attention window**, the 90 s session timeout. A fortnight is almost entirely
  rest, so months still compress into seconds. The soak zeroes the panel timing model separately,
  which means **panel-timing bugs are only ever caught by a human** — accepted.
- **The panel model advances on simulated time and is presented at host vsync.** Waveform frames
  tick at ~85–90 Hz and monitors do not divide by that, so the panel is treated as a physical
  surface being sampled rather than a frame source. A 60 Hz monitor sees an aliased A2 flicker,
  which is worth knowing when judging whether a flash reads as punctuation or as a glitch.
- **Every ambient wake is a logical cold boot with poisoning.** `load → step → render → save`, then
  the shell destroys its state and overwrites the buffer before reloading through the simulated
  storage adapter. Paired with a build-time check that `core/` carries no mutable static storage,
  this covers what process-per-wake would have. The free side effect is real coverage: N fortnights
  of soak is N fortnights of continuous exercise of the save envelope and the crash-safe journal
  that [ADR-0004](0004-effects-core-no-ports-in-core.md) pushed into core (note
  [`0005`](../research/0005-persistence-and-crash-safe-saves.md)).
- **There is no pickup input, because there is none in the device.** The BMI270's `INT1` pulls the
  `E_TRG` net and the PMS150G re-energises the board; the ESP32-S3 is not party to the decision, and
  [ADR-0011](0011-two-clocks-and-the-life-clock-cannot-step.md) has the shell *infer* a **pickup**
  from `now < armed_instant`. The simulator therefore offers a **latch input** — a control that
  powers the board on now — and every press exercises that inference, including the hop-boot path
  that re-arms and powers off without ever entering core. Held down, it is the jostle storm
  [ADR-0009](0009-pickup-listens-touch-attends.md) needs to turn "a reserve that outlasts an unknown
  number of jostles" into a number.
- **`Button` is retired; there are eight ports, not nine.** Note
  [`0006`](../research/0006-input-inventory.md) is unambiguous — "exactly one, and the ESP32 cannot
  read it": `SW_PWR` reaches pin `PA4` of the PMS150G and no ESP32 GPIO, and M5Unified has no GPIO
  button read for this board at all. A port with no possible device adapter is the ticket's own
  failure mode sitting in the port list. The side button is real and player-visible, so it joins
  pickup as a second **latch input** — click on, double-click off — which also gives the simulator
  the nasty case worth testing: a player who powers off mid-session, or mid-save. This amends
  [ADR-0004](0004-effects-core-no-ports-in-core.md)'s inventory; it reopens none of its decisions.
- **The bezel rule is absolute: nothing that is not on the panel is ever drawn on the panel.** No
  debug overlay, no click ripple, no "window closed" hint. Outside a window the device is deaf and
  gives no feedback, and reproducing that is most of the point — the single biggest felt difference
  between this design and the obvious one. Diagnostics live in surrounding chrome, where a dropped
  click reads `touch dropped — no attention window (closed 4m12s ago)`. One exception erodes the
  panel's trustworthiness entirely.
- **Clicks are dropped in the touch adapter, not by core.** An unpowered GT911 reports nothing, so
  neither does the simulated one. Delivering the event and letting core ignore it would put phantom
  input in the **replay log** and let a replay diverge from the life it claims to reproduce.
- **Player view is the default and resets on every launch.** Instrument view adds the full state
  dump — real meter values, weight, discipline, the hidden **care mistake** count, time to next
  call. That is the best debugging surface in the tool and a direct attack on what the tool is for:
  the design's central bet is that a pet can be read from sparse indirect signals, and a numeric
  readout in peripheral vision disqualifies every judgement about legibility. A sticky preference
  would mean enabling it once during a bug hunt and playtesting through it for a month. The
  **structured log carries everything in both modes**, because a dump read afterwards cannot corrupt
  a judgement made at the time.
- **Tilt is handicapped on purpose, and tilt-maze's difficulty constants are hardware-only.** The
  mouse drives a *target* orientation which the simulated device follows through lag, a noise floor
  and a dead zone, deliberately pessimistic — better to find the game easier on hardware than the
  reverse. The simulator can answer tilt-maze's structural questions (does the maze fit the region,
  is the goal reachable, does a round read at ~7 fps, does the round-end `Clear` land well) and has
  no standing to answer its difficulty. #16 owns the game; this ADR owns only the vector.
- **Steps are generated by diurnal profiles sampled on the life clock.** The BMI270 counter
  free-runs across board-off and is read lazily, so the simulated one must accrue through compressed
  rest — otherwise a fortnight in seconds yields no steps and ADR-0007's soak assertion that "all
  four evolution branches reachable" fails on BODY forever, or passes for the wrong reason. Profiles
  rather than a flat rate because step-walk's real question is about distributions across a week.
  A counter-reset injector covers the field bug nothing else can reach: `IMU_VDD` survives
  main-power-off but not a total battery death, so a recharged device reads a delta that has gone
  backwards.
- **Determinism is an invariant, not a default.** The simulator is reproducible given
  `(seed, event log, injection script)`, and any adapter that cannot meet that does not ship.
  `Network` returns scripted outcomes on the simulated clock, `Storage` is file-backed with
  injectable torn writes and late failure events, `Power` is a coulomb model — which is also the
  only way **Torpor** is reachable at all. The payoff is that the project's documented hazards stop
  being prose: ADR-0011's lost clock, ADR-0009's Torpor jostle, note 0005's torn write, ADR-0012's
  enrolment drop-out and #26's dead timezone endpoint become a regression suite that runs every
  push. **The accepted cost is that the real network is never exercised**, so a cloud target
  misbehaving in an unimagined way is found by a device on a desk — which is true of any simulator.
- **Recording is always on; rewind is replay.** A replay log is written continuously beside the
  structured log, with a key to bookmark "the bug is here", because the interesting failure at hour
  200 of a soak is precisely the one nobody thought to record. Determinism means "back up three
  days" is a replay from the seed to an earlier instant, so no snapshot system is needed.
- **Goldens call `render()` directly and the simulator's CI contribution is the soak.** A pixel test
  should not depend on SDL, the LUTs, the optical ramp and host sampling — four provisional models
  with nothing to say about whether the pixels are right. The panel model is asserted numerically
  instead: budget never exceeded, no region re-driven mid-waveform, every animation run ended on its
  **rest frame**, peak residual bounded at each `Clear`. The line generalises — **pixels are core's
  problem, physics is the port's problem, behaviour over time is the simulator's problem.**
- **#10 becomes downstream rather than parallel.** Once the panel model is real and INSTINCT is
  playable, "is catch-the-falling-food actually fun at ~5 fps?" is a session in the simulator rather
  than a separate spike — but the simulator has to exist first.
- **SDL3, pinned to a release tag.** SDL2 is in maintenance and there is no legacy here to carry;
  the dependency is built from source under ADR-0006's FetchContent rule, so distro packaging is
  irrelevant to the choice. The cost is that much of the available example material is SDL2-shaped.

The numbers here — the reflectance ramp's sixteen entries, reflectance-per-drive-frame, the tilt
lag/noise/dead-zone constants, the step profiles, the per-wake coulomb envelopes — are starting
values for hardware to correct. Following the precedent of ADR-0008, ADR-0009 and ADR-0011: **the
model is what this ADR commits to; the numbers are placeholders until measured.** The linear optical
model in particular is known to be wrong in a specific way — real black→white and white→black are
not symmetric — which makes every ghosting estimate optimistic in one direction until the
exponential replaces it.
