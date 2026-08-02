# Refresh is budgeted by driven area, and clearing refreshes are events

The display port exposes four refresh intents — **`Animate`**, **`Flip`**, **`Settle`**,
**`Clear`** — and owns a ghosting budget denominated in *accumulated driven area*, not in update
count. A **clearing refresh** is triggered by a narrative moment or by the budget running out,
never by a timer, and never more often than once a day at minimum.

Three source findings force this shape (research note
[`0009`](../research/0009-refresh-strategy.md)):

- **Update time is independent of region size.** Every waveform frame clocks all 540 panel rows in
  both candidate libraries, with no config knob. A small window buys panel wear, CPU and ghosting
  headroom — not milliseconds.
- **The fast modes are exactly DC-balanced over a black↔white cycle; the greyscale clearing mode is
  not** (−3 per cycle, and +2/−5 when refreshing static content in place). Animation is
  electrically cheap; periodic full refreshes are electrically expensive. The naive policy —
  animate sparingly, clean up on a timer — is backwards on this panel.
- **Ghosting is a per-pixel property.** Pixels never driven accumulate nothing, so a budget that
  counts *updates* over-charges a small region by the ratio of screen area to window area.

## Considered options

- **Count-based: a clearing refresh every N fast updates.** The convention almost everywhere —
  KOReader and Allwinner's BSP independently landed on 6, Good Display advises 5. Rejected because
  every one of those is shaped for full-screen e-reader page turns. Applied to a pet window at ~5 %
  of the screen it would over-clear by roughly 20×, spending a 400–740 ms flash to fix ghosting
  that has not happened.
- **Time-based: clear on a fixed interval.** Rejected outright once the LUTs were decoded — each
  clearing pass is itself DC-imbalanced, so a timer converts the cure into the disease. Survives
  only as the 24-hour floor, which is the one interval any vendor frames as damage prevention.
- **Track per-pixel accumulated impulse**, as Nook's US9613599B2 does with an 8-bit signed charge
  history per pixel. Physically the most correct option and the model this ADR borrows its
  reasoning from. Rejected as implementation: it needs a per-pixel side buffer and a waveform LUT
  indexed by charge history, and M5GFX exposes neither. Region-level area accounting captures most
  of the benefit for none of the cost, because our regions are fixed by layout and our fast modes
  are already balanced.
- **Shrink the refresh region to buy framerate.** The original premise, and it is simply false on
  this hardware. Recorded here because it shaped the map for months.

## Consequences

- **The animation vocabulary inherits a hard constraint: every run returns to its rest frame.**
  Balance holds over a *complete* black↔white cycle, so a run that ends where it began is net-zero
  by construction and needs no cleanup at all. A pose that drifts and never closes a loop does. This
  is #13's constraint to design within, and it is free.
- **Every animation run is bracketed by a white frame,** entering and leaving. This is E Ink's own
  published ritual for their A2 mode, and it is shipped as first-class `A2_IN`/`A2_OUT` waveform
  modes in Allwinner's production driver. The port owns it so games cannot forget it.
- **The clearing refresh has somewhere to hide, and it was already in the design.** Entering a
  session, the pet waking, the pet going to sleep, and a minigame round ending are all moments that
  already justify a visual transition. Demand-driven sleep (ADR-0002) guarantees a daily wake, which
  is exactly where the 24-hour obligation goes. We did not have to invent a moment.
- **The 1-bit pet inside greyscale chrome is now justified on physics, not taste.** Saturated black
  and white self-correct on every drive because the pigment is packed against the electrode and the
  optical response saturates; mid-greys have no such stop and integrate their error. The art
  direction picked the right split for the wrong reason.
- **Layout (#17) must not size the pet window hoping to buy framerate.** Size it for composition,
  for cumulative panel stress over a two-week life, and for the area budget — three real reasons, no
  longer the imaginary one.
- **The port grows responsibilities beyond drawing:** the area budget, per-region serialisation
  (a waveform interrupted mid-flight is DC-imbalanced by construction), white-frame bracketing, and
  two workarounds for current M5GFX defects. Core still names none of this — it emits a repaint
  effect carrying an intent, and the port decides what is affordable.
- **`CONFIG_FREERTOS_HZ=1000` becomes a correctness-adjacent setting, not a tuning knob.** At
  ESP-IDF's default of 100 Hz the driver's per-frame `vTaskDelay(1)` quantises every waveform frame
  to 20 ms and halves every refresh rate on the device.
- **The budget number is borrowed and provisional.** ~400 fast updates for a 5 %-of-screen window is
  Rockchip's `20 × screen_area` constant divided by our window fraction — a different panel, a
  waveform that may not be balanced. The *model* is what this ADR commits to; the *number* is a
  placeholder until measured.
