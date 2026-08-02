# Care mistakes are logged permanently but evaluated per stage

Gen1 kept care mistakes in a single hidden counter that only ever went up, and read it at
every evolution. We keep the mistakes themselves permanent — each one is written to the
care log as a timestamped line, and the raw count stays hidden from the player exactly as
it was in Gen1 — but **each evolution reads only the stage that just ended**, not the
whole life.

The original's accounting was harsher on paper than in practice, because a player who
ruined a run could reset in ten seconds. Here permadeath is structural, the run is a
fortnight long, and a restored snapshot of a dead pet is still dead. Lifetime-monotonic
accounting would let a bad Tuesday silently foreclose every good adult form, with no way
for the player to know it had happened or that recovery was pointless.

## Considered options

- **Lifetime monotonic counter.** Rejected for the reason above, despite being the more
  faithful reading of Gen1 and the basis of its perfect-care-run culture.
- **Continuously decaying care score.** Rejected as illegible: nothing in the care log
  would correspond to the current score, evolution outcomes would be hard to explain and
  hard to assert on in the fast-forward harness, and it would reward cramming care just
  before a stage boundary.

## Consequences

- The save schema must retain mistakes as **dated events**, not as an integer, so that a
  stage window can be evaluated after the fact.
- Care quality is a **per-stage derivation**, and the evolution tree (#14) composes it
  with play style at each evolution moment. How they compose is #14's decision; that they
  are both stage-scoped is this one's.
- The care log is the player-facing surface for the mistake record. It is the one thing
  the original could not have had, and it is what makes a hidden counter fair.
- A player who has a bad first day can still reach a good adult form. This is deliberate.
