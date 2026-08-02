# Sleep is demand-driven, not scheduled

Gen1 put the pet to sleep on a fixed clock and required the player to dim and raise the
room light at those times. We keep the lamp verb and both of its care mistakes, but drive
sleep from the pet instead of the clock: the pet becomes **sleepy**, calls to be put to
bed, and sleeps once its lamp is dimmed. Morning wake is automatic.

A fixed schedule imposes two appointments a day at times the player did not choose. On a
keychain carried everywhere that is fine; on a desk you walk away from, with a budget of
2–4 care events per day, it produces a device that punishes you for being in a meeting at
8pm. Demand-driven sleep turns the same verb into one of the day's ordinary care events.

## Considered options

- **Verbatim fixed schedule.** Rejected for the reason above. It would have handed the
  time model a trivially simple definition of night, which is the real cost of this
  decision.
- **Cut the verb; dim the room automatically.** Rejected under ADR-0001 — it deletes one
  of Gen1's eight verbs and turns the pet's night into a screensaver.

## Consequences

- **The light mechanic was never a hardware problem.** The P1 was a passive LCD with no
  backlight; its Light button toggled a *depicted* room light. On e-ink the dark room is
  cheaper than the original — held at zero power, two region inversions a day, and sleep
  becomes the lowest-refresh state on the device.
- **Night is defined by the pet**, bounded by user-set quiet hours. The time model
  (#18) inherits this rather than choosing its own definition.
- **Player-forced sleep would have been a pause button.** If the lamp could start sleep
  and sleep suppressed decay, a player could idle out the fortnight. Closed from both
  sides: sleep is pet-initiated, and meters keep draining overnight at a reduced rate.
- Dimming the lamp on a wide-awake pet remains a care mistake, as in Gen1.
