# Pickup listens, touch attends

The mode machine is four states — **Ambient**, **Session**, **Torpor**, **Memorial** — and the
player enters a session by *touching* the device, which is only possible inside a bounded
**attention window** that a **pickup**, a buzz, or a recent session has opened. Core owns the mode
and emits a **wake deadline**; the shell alone decides whether to honour it by light-sleeping or by
powering the board off.

The shape is forced by one pin. `TP_INT` is on G48, outside the ESP32-S3's RTC GPIO range of 0–21,
so touch cannot end deep sleep — and once the board is off there is no chip left to interrupt
(note [`0006`](../research/0006-input-inventory.md)). The BMI270's `INT1`, by contrast, reaches the
gate of `Q4` and pulls the `E_TRG` net that latches the PMS150G, so motion re-energises a dead board
as a cold boot (note [`0012`](../research/0012-power-budget.md) §3.6). **The gesture the design
wanted for attending is the one the silicon cannot hear, and the gesture it can hear is the one a bag
jostle also produces.**

So the two are given different jobs. A **pickup** makes the pet notice you: one `Animate` run on the
**rest frame**, a ~30 s attention window, and nothing else — no hearts, no `Clear`, no session
logged, no care state touched. A touch inside that window makes it attend to you. Windows also open
for ~20 min after a session ("it's awake for a bit after you use it") and for ~60 s after a buzz,
which is the highest-probability touch in the day.

## Considered options

- **Pickup opens a session.** The obvious reading of the map's "entered by touch or pickup".
  Rejected because session entry spends a full-panel greyscale pass and brings up the hearts, and a
  device on a desk gets knocked, moved and handed around. Every jostle would spend the most
  expensive repaint we have, and the **care log** would fill with sessions nobody had.
- **Permanently touch-armed ambient** — light sleep with the GT911 in Green and
  `gpio_wakeup_enable(48, LOW)` armed, at 3.3 mA. Genuinely viable, and the honest rival: note 0012
  §9.1.4 prices it at 1020 mAh/week, still clearing one week at 9.9 days. Rejected because it spends
  ~70 % of the budget to remove a limitation a bounded window removes for 2.7 %, and because it
  forecloses everything else the headroom later buys.
- **Touch never wakes the device; the RTC is the only way in.** The fallback note 0006 named.
  Rejected: a desk companion you cannot poke is a clock.
- **Core names its own rest state.** Note 0012 §9.1.3 says "the rest-state decision and the cadence
  decision are one decision", which taken literally puts board-off-versus-light-sleep inside core.
  Rejected against [ADR-0004](0004-effects-core-no-ports-in-core.md). The two must be chosen
  *consistently*, which is a property of a pair of numbers, not of the module that holds them. The
  ~3-minute crossover is an unmeasured hardware constant (note 0012 §10, measurement 1) and it must
  be free to move without touching core or invalidating a **replay log**.

## Consequences

- **There is no in-RAM state between ambient wakes.** Board-off ambient is a cold boot: the wake
  path is `load → step → render → save → power off`, not an event loop. That suits the pure `step()`
  of ADR-0004 — core is already a function from persisted state — but core must never be written as
  though a `static` survived.
- **There is no reason to tick, and that is the cadence model.** `step()` is pure and pet state is a
  function of the **hatch instant** and the event log, so nothing needs computing early; the BMI270's
  step counter free-runs across board-off and is read lazily. **The only reasons to wake in Ambient
  are that something visible changes, or that it is time to buzz.** Core therefore emits a *deadline*
  — the earliest instant either becomes true — not a period.
- **Because the hearts are hidden in Ambient, most state changes are invisible and cost nothing.**
  A Hunger heart dropping changes no pixel. The short list that *is* visible unattended is the pose,
  the **silhouette**, poop, the skull, the dark room, and the call indicator — and that list, not the
  simulation, is what sets the wake schedule.
- **The resting animation cadence is one ~1 s run every 3 minutes**, and quiet hours are still
  ([ADR-0010](0010-alerts-escalate-animation-de-escalates.md)). Shorten the run before slowing the
  cadence: the boot-and-shutdown envelope is a fixed ~41 µAh per wake, so halving a run buys ~1.5×
  the frequency for the same money, and for liveliness *how often* beats *how long*. The whole device
  lands near 22 days at nominal, ~3× the one-week target.
- **The chrome is frozen in Ambient; the care log is written up at session entry.** A **care
  mistake** logged at 2pm would otherwise force a 37-scan greyscale pass — the most expensive repaint
  we have, and the electrically imbalanced one (ADR-0008) — to update text nobody is reading. The
  side effect is a better artefact than live updating would be: a written record that is brought up
  to date when you sit down with it.
- **Core emits `Clear` at four narrative moments only** — hatching, the pet's morning wake, entering
  Torpor, entering Memorial — and `Settle` everywhere else, which the port may promote when the
  ghosting budget demands. ADR-0008 puts the budget in the port and says a clearing refresh fires on
  a narrative moment *or* on the budget running out, so the port owns that second trigger. Session
  entry therefore flashes when it needs to, not on a schedule. Torpor and Memorial clear
  unconditionally because both hold an image at zero power for days: the last thing the panel does
  before a long silence should be to leave itself clean.
- **The device must infer why it woke**, since a pickup and a timed wake arrive as the same power-on
  reset. Comparing now against the persisted wake deadline always works and needs no hardware
  knowledge. Reading the BM8563's timer flag would be more direct but is unverified across the
  PMS150G power cycle, so the deadline comparison is the primary and the flag is an optimisation.
- **Pickup cannot be disarmed.** The latch is hardware. Torpor is therefore cold-booted every time a
  flat device in a bag is jostled, spending charge that is by definition already gone — so the Torpor
  boot path must read the battery and drop power again *before* initialising PSRAM or the panel, and
  the final cloud snapshot must be taken on the way in, behind a reserve that outlasts an unknown
  number of jostles.
- **The session ends after 90 s of no touch, with one exception:** an open game round holds it, under
  its own 3-minute abandonment timeout. MIND is a nonogram with no timing pressure and a player will
  stare at a hard board for minutes; the ordinary rule would end the session and lose the puzzle.
  Note 0012 §9.1.5 calls this timeout "the cheapest real lever", but that assumes a session burns
  full power throughout — with light sleep between touches an idle session is ~4.1 mA, not ~120 mA,
  and 60 s versus 90 s rounds to nothing. **Spend the lever on implementing light-sleep-between-
  touches and choose the timeout on feel.**
- **Session end is not a save point.** Every care event saves (ADR-0005), so durability is already
  finer-grained than the session — which is what makes a brownout mid-session cost only the session.
- **Sleep, calling, illness and charging are conditions, not modes.** They change cadence, content
  and alerting inside Ambient and add no edges. `CONTEXT.md` already defines **Sleep** as the *pet's*
  rest state; promoting it to a mode would duplicate every Ambient edge for nothing. A session is
  permitted while the pet is asleep — checking on it is a Gen1 verb, and raising the lamp must stay
  reachable or ADR-0002's care mistake becomes unreachable. A pickup during Sleep or quiet hours
  listens silently and renders nothing: sleep is pet-initiated and a jostled desk must not end it.
- **Charging removes the limitation entirely.** On a charger the attention window never closes and
  the animation runs at 1/min — energy has stopped being the constraint, and the device is
  touch-responsive exactly when a player is most likely to be fiddling with it. It does not suppress
  the buzzer; quiet hours already covers the overnight case.
- **Deadlines are inert during a session.** The device is already awake, so a call threshold or the
  morning wake crossing mid-game is an ordinary `step()`, not a mode change.
- **The simulator (#17, #20) must model cold boots and the attention window**, or it will reproduce a
  device markedly more responsive than the real one, and playtesting will validate a product that
  does not exist.

The numbers here — the 30 s / 20 min / 60 s windows, the 90 s timeout, the 3-minute abandonment, the
1 s run every 3 minutes — are starting values for the simulator to tune. Following ADR-0008's
precedent: **the model is what this ADR commits to; the numbers are placeholders until measured.**
