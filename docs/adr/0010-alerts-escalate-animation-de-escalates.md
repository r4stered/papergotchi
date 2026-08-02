# Alerts escalate, animation de-escalates, and quiet hours are absolute

A **call** raises its visual signal immediately, buzzes twice — at +10 and +30 minutes — and is then
permanently silent for that call. Once the buzz phase is spent the pet's *animation cadence falls*.
Quiet hours suppress the buzzer **and** the idle animation, with no exception for any state,
including a flat battery and a dying pet.

Note [`0012`](../research/0012-power-budget.md) §9.1.2 identifies the one shape that breaks the power
budget: a pet left calling. Its own suggestion is to escalate the wake cadence during a call and rely
on the buzz phase being bounded to bring it back down. We keep the bound and invert the cadence —
**an unanswered pet does less, not more** — which closes the failure case by construction rather than
by a policy someone has to remember.

The visual signal carries almost all of the load, and it is free: the call indicator is e-ink, so
once flipped it persists at zero power indefinitely.

## Considered options

- **Escalate the animation along with the alert**, as note 0012 §9.1.2 sketches. Rejected on both
  currencies. In power it makes the worst case the most expensive state, so the budget comes to
  depend on the buzz phase being correctly bounded forever. In character it reads as a tantrum, which
  is the wrong note for a desk object across an unattended afternoon.
- **One buzz, as the map's attention model says.** Rejected as too easy to miss — one buzz while you
  are in another room is no buzz at all. Two is escalation; three is an alarm clock.
- **Let something pierce quiet hours** — the low-battery warning, or the **rescue window** on a dying
  pet. Rejected in favour of turning both into constraints on other tickets (below). An exception,
  once granted, gets widened.
- **Animate through quiet hours.** Rejected: e-ink emits no light, so a pet animating in a dark room
  is unobserved by construction, and this board appears to have no ambient light sensor with which to
  know better. Stillness costs nothing perceptible and deletes roughly a third of the dominant line
  item in the power budget.

## Consequences

- **An ignored pet is cheaper than a content one.** The budget's worst case is now its cheapest
  state.
- **A sleeping pet raises no calls.** Meters still drain overnight — [ADR-0002](0002-demand-driven-sleep.md)
  keeps that exploit shut — but a need crossing its threshold at 3am produces a call at the **morning
  wake**. ADR-0002 rejected a design that punishes you for being in a meeting at 8pm; punishing you
  for being asleep is the same error.
- **A call raised inside quiet hours never arms its buzz schedule** — suppressed, not deferred.
  Buzzing at 07:00 about a need acquired at 02:00 is pedantry, and the morning wake is about to
  surface it anyway.
- **A needless call is alert-identical to a genuine one.** If the two buzzed differently the player
  could tell them apart without looking, and **scold** would stop being a judgement.
- **Attending silences the phase; the indicator stays.** Per `CONTEXT.md`, a call is cleared only by
  satisfying the underlying need — attending merely silences the buzzer.
- **Every buzz opens a ~60 s attention window** ([ADR-0009](0009-pickup-listens-touch-attends.md)).
  The moment after a buzz is the likeliest touch of the day, and it is the difference between a
  device that nags and one that answers.
- **Two constraints leave this ADR for other tickets.** #14: the **rescue window** must outlast the
  longest permitted quiet-hours span, or a dying pet found in the morning is unsavable and quiet
  hours needs the exception this ADR refuses. #18: quiet hours therefore needs a *maximum* span, and
  it is bounded by #14's answer rather than chosen freely. Separately, the low-battery warning must
  fire with days of margin rather than minutes — **early beats loud**, and it is what lets the
  device never choose between waking you and going flat.
- **Quiet hours are a rendering rule as well as an alerting one**, which #13 and #17 inherit: only
  `Flip` is permitted, so a discrete change such as poop appearing is still drawn and nothing else
  moves.

+10/+30, the two-buzz bound and the de-escalated cadence are starting values for the simulator to
tune. The model is what this ADR commits to.
