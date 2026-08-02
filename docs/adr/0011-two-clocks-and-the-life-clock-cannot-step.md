# Two clocks, and the life clock cannot step

The device derives two clocks from the one BM8563 register. The **life clock** — `rtc + skew_offset`
— is what the simulation runs on, and every accepted correction writes the same delta into the
offset, so it **cannot step**. The **civil clock** — `rtc + utc_offset` — steps freely and is read
only by **quiet hours**, the **morning wake**, and **care log** rendering. They are distinct types
over `int64` milliseconds, so mixing them is a compile error rather than a pet that ages wrong for
days.

The ticket asked for a guard against the clock running backwards and double-aging or un-aging the
pet. This is the answer, and it is a shape rather than a rule: **a correction is invisible to the
simulation by construction**, so there is nothing to guard. That is the same move
[ADR-0010](0010-alerts-escalate-animation-de-escalates.md) made when it inverted the animation
cadence — closing a failure case by construction rather than by a policy someone has to remember.

Drift turns out not to be the problem the ticket expected. A 32.768 kHz crystal at ±20 ppm is
~1.7 s/day, so a **full 14-day life accumulates ~24 s** **[D]/[U]** — less than the rounding this
ADR deliberately adds to a *single* wake (below). **Steps are the entire threat; rate error is
noise.**

## Considered options

- **One clock, with jump detection and a policy per class.** Classify each observed change —
  sub-second boot regression, small NTP nudge, large step, backwards — and act per bucket. Rejected:
  four behaviours, four sets of tests, and a rule every future mechanic must be told about. It is
  the shape ADR-0010 rejected twice and [ADR-0009](0009-pickup-listens-touch-attends.md) once.
- **A single closed form `f(elapsed)` from the anchor**, as note
  [`0005`](../research/0005-persistence-and-crash-safe-saves.md) *"re-derives meters from
  `now − last_anchored_instant`"* can be read to imply. Rejected: it cannot express reduced overnight
  drain, which [ADR-0002](0002-demand-driven-sleep.md) named as the thing keeping the
  idle-out-the-fortnight exploit shut.
- **Fixed-step catch-up.** Chop a gap into slices and iterate. Rejected: it reintroduces the tick
  ADR-0009 deleted, quantises every threshold, and makes a week in **Torpor** ~10,000 iterations of
  an allocation-free loop for a result that is arithmetic.
- **An IANA timezone picker with tzdata on device.** Correct, including DST, at the price of a
  scrolling list of ~350 entries on a 7 fps panel — a worse input problem than #19's passphrase —
  and a blob that goes stale on a device with no update path.
- **Killing a pet on a guessed gap** when the clock is lost. Rejected against note 0005's
  *"Never silently resurrect and never silently kill."*
- **Seconds in core, with session-local time for the games.** Genuinely viable, and it makes the
  cross-boot sub-second regression (below) unrepresentable rather than clamped. Rejected because it
  buys that with a conversion seam between the simulation's clock and the games', and seams are
  where mismatches hide.

## Consequences

- **A gap is crossed by walking regime boundaries, inside core.** Core computes the next boundary
  after the anchor — a quiet-hours edge, the morning wake, sleep onset, a **stage** transition, a
  call threshold, a buzz instant, death — integrates in closed form to it, applies what is discrete
  about it, re-anchors, repeats. Work is proportional to boundaries crossed, not to time elapsed: a
  week in Torpor is ~30 segments. A gap exceeding the **lifespan band** short-circuits to the death
  evaluation rather than walking it.
- **`next_boundary()` is also the wake deadline.** One function answers "when do I integrate to
  next?" and "when should the shell wake me?", so the two hardest-to-test behaviours in the system
  share an implementation that every single wake exercises.
- **The walk lives in core, not in the shell.** A shell looping *"step until the deadline is in the
  future"* is tempting, since it already honours deadlines — but each segment would emit effects
  without knowing whether it is history or now, and a device back from a week in Torpor would
  **buzz about a call from last Tuesday**. Core walking its own boundaries knows which segments are
  past, coalesces internally, and emits one repaint and no stale buzzes.
- **Exact boundary instants are what ADR-0003 needs.** Care quality is evaluated per stage; an
  approximate stage boundary puts a **care mistake** near midnight in the wrong stage and the
  evaluation is quietly wrong.
- **The life clock is monotonic in seconds and not monotonic across boots.** The RTC has no
  sub-second field, so `boot_wall_ms` is always a floor. A session running 800 ms past its last RTC
  tick leaves the next cold boot **up to 999 ms behind** what was already simulated — un-aging,
  arriving from the boot path rather than from NTP. The guard is `now = max(now, last_seen)`,
  persisted with the save envelope. Sub-second, so it distorts nothing.
- **That same clamp is the whole lost-clock story.** Handed a year-2000 RTC after a total battery
  death it clamps to `last_seen`, which *is* zero elapsed — the only honest number when the gap is
  unknowable. Detection is by plausibility (`rtc_now < last_seen`) first; the PCF8563-family **VL**
  flag would be more direct but is unverified here, so it is an optimisation, following ADR-0009's
  precedent for wake-reason inference. Recovery attempts NTP on the charger boot, and since
  `utc_offset` survives in the save, local time returns with no player interaction.
- **A player can freeze a pet by leaving it flat.** Accepted. They get no play from a device that is
  off, so the exploit is "don't play with your pet"; and it extends a protection the map already
  grants — *"unpowered time accrues neglect but cannot kill"* — rather than opening a new one.
- **Civil time comes from the network at first setup**, with a timezone endpoint proposing and the
  player confirming. The offset refreshes opportunistically whenever the radio is already up for a
  backup, so **DST corrects itself within a day with no tzdata and no transition rules on device** —
  and it arrives through the *same* path as an NTP correction, already proven safe against the
  simulation. **A manual correction pins the offset and disarms the refresh** for the life of that
  pet: the player's judgement outranks a service that cannot see their VPN. Manual entry is the
  fallback wherever the endpoint is unreachable, which is also what keeps a dead third-party service
  from becoming a dead device.
- **Quiet hours are a predicate, not a pair of scheduled events**, evaluated at `civil_now` on every
  ambient wake — which is what ADR-0010 already requires. A predicate is idempotent under any jump:
  fall back cannot fire it twice, spring forward cannot skip it.
- **The morning wake and sleep onset are latched by pet state.** The morning wake is not "an event at
  07:00" but "the transition out of **Sleep**", so a repeated hour fires nothing and a skipped hour
  still leaves a sleeping pet whose predicate now says *not night*. At most once, at least once, free.
- **A wrong civil clock has a tiny blast radius.** It cannot age the pet, kill it, or move a meter.
  Its worst outcome is a badly-timed night — a comfort bug, not a data-loss bug, which is why no
  classifier is warranted. The one thing lost is the ability to *notice*, so clock corrections are
  written to the **care log** as dated lines: the player reads "clock corrected +5h58m" and
  understands their pet's odd bedtime without the device having to reason about it.
- **Quiet hours are mandatory and clamped to 4–10 h, defaulting to 22:00–07:00**, as a hard clamp in
  the UI rather than advice — ADR-0010's reasoning about exceptions applies equally to soft limits.
  Four hours is below anyone's real night, so nobody feels the floor.
- **The pet wakes when quiet hours end.** ADR-0002 left "morning wake is automatic" undefined;
  anchoring it to the player's own declared night applies that ADR's own principle — it refused to
  *"punish you for being in a meeting at 8pm"* — to the morning, and puts the overnight call backlog
  on screen when the player is most likely standing in front of the device. **A late bedtime is
  therefore a short night**, which gives ADR-0002's "lamp left up on a sleepy pet" care mistake teeth
  without inventing a second penalty. Onset stays entirely the pet's.
- **This is why quiet hours cannot be switched off**: the pet would have no night to wake from.
- **The morning wake hands ADR-0008 a free daily deghost.** Its `Clear` now lands at a predictable
  civil time every day, so the ghosting budget is discharged on a **narrative moment** rather than
  the port having to spend a flash on a timer — which [ADR-0008](0008-refresh-policy.md) said it
  never wanted to do.
- **Age-driven discrete events land at the morning wake.** Hatching requires the player present, so
  hatch instants cluster in the evening and every **year** boundary would otherwise fall at that
  hour — an evolution at 22:30, inside quiet hours, where only `Flip` is permitted and the
  **silhouette** change would happen silently in a dark room. Landing them at the morning wake means
  the pet goes to bed one thing and wakes up another, reuses the `Clear` already being spent, and
  **demotes the year to a derived `floor((now − hatch)/24h)` that leaves the boundary set entirely.**
- **Nothing can die overnight.** Offered to #14 as a paused **rescue window**: time inside quiet
  hours does not count against it. ADR-0010 defends absolute quiet hours on the promise that *"a
  dying pet found in the morning is still savable"* — without pausing, a pet entering its window at
  18:00 with a 12 h window dies at 06:00 and the promise is only *probably* true. Pausing makes it
  structurally true, and it is the same principle already accepted twice: unpowered time cannot kill,
  and a lost clock elapses zero. Meters keep draining throughout; only the death timer pauses.
- **Arming rounds up, never down.** ADR-0009 infers wake reason by comparing now against the
  persisted deadline, so a countdown that expires *early* is misread as a **pickup** and the pet
  animates and opens an attention window for nothing. Late is read correctly and costs seconds no
  mechanic can perceive. **The PCF8563 countdown decrements on a free-running source, so its first
  tick is fractional** — up to a full minute short against the 1/60 Hz source **[D]/[U]**. Arming
  `ceil(delta/unit) + 1` absorbs it. M5Unified rounds to *nearest* (`(afterSeconds + 30) / 60`),
  which is exactly the wrong direction; the registers are written directly, which the map's "pure
  ESP-IDF, no Arduino" already implied.
- **The shell persists `armed_instant` beside the true wake deadline.** A boot is timed if
  `now ≥ armed_instant` and a pickup otherwise — still ADR-0009's mechanism, applied to the number
  the RTC was actually given. This is what makes the **255-minute ceiling** clean: an 8 h night is
  decomposed into hops by the shell, and a hop boot sees `now ≥ armed_instant` but
  `now < true_deadline`, re-arms and powers off **without ever entering core** — no `step()`, no
  panel, no save, just the ~41 µAh boot envelope, twice a night. Core never learns the ceiling
  exists, which keeps the hardware constant out of core per
  [ADR-0004](0004-effects-core-no-ports-in-core.md).
- **A deadline already in the past needs no special case.** Overslept through Torpor, or a lost
  clock, is exactly what the boundary walk is for.
- **There is no scheduled NTP.** Its three jobs are setup, the DST offset refresh, and silent
  lost-clock recovery — correcting drift is not one of them, since ~24 s over a full life is smaller
  than the rounding one wake deliberately adds. Time therefore rides on connections that were
  happening anyway, and **a week with no network costs ~12 s**. The only real casualty is a DST
  transition inside an offline week, uncorrected until the next connection. Every NTP result passes
  a plausibility floor — nothing earlier than the hatch's own civil time — before it is accepted.
  Note [`0012`](../research/0012-power-budget.md) §6.2's two defaults matter here now that time rides
  on backup bursts: `CONFIG_LWIP_SNTP_STARTUP_DELAY=n` (the stock default spends **up to 5 s doing
  nothing**) and `CONFIG_LWIP_DHCP_DOES_NOT_CHECK_OFFERED_IP=y`.
- **The boundary set is a list every future mechanic must contribute to, and a forgotten
  contribution fails silently** — wrong integration *and* a missed wake, with no crash. This is the
  real price of the model. It is a single total function, so it is exhaustively testable, and the
  **replay log** and the simulator are what will catch an omission.
- **~~#19's founding premise is reversed.~~** *Amended by
  [ADR-0012](0012-the-keyboard-belongs-to-the-phone.md).* This ADR concluded that requiring the
  network at first setup makes provisioning **hatch-blocking**, against #19's premise that *"the game
  is fully playable with no network, so provisioning must never be a gate on play"*. #19 settled it
  the other way: **what blocks hatching is a confirmed civil clock, not a network**, and manual entry
  is a *peer door* rather than a fallback — so the premise survives almost intact. Everything else
  here stands, including that civil time comes from the network when the player takes that door.
  ADR-0012 also closes a hole this one left: the plausibility floor is defined as *"nothing earlier
  than the hatch's own civil time"*, which does not exist at **first** enrolment, where the floor is
  the firmware build timestamp instead.

The numbers here — the 4–10 h quiet-hours clamp, the 22:00–07:00 default, the 15-minute offset
rounding — are starting values for the simulator to tune. Following the precedent of ADR-0008 and
ADR-0009: **the model is what this ADR commits to; the numbers are placeholders until measured.**
