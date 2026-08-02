# papergotchi

A Tamagotchi-style virtual pet for the M5Paper S3 — a desk companion whose care loop
runs on e-ink, in real time, over a roughly two-week lifespan ending in permadeath.

## Language

### The pet and its state

**Pet**:
The single living creature the device simulates. Exactly one exists at a time.
_Avoid_: Character, creature, avatar, tamagotchi

**Hunger**:
How fed the pet is, shown as four hearts. A meal restores one heart.
_Avoid_: Food level, satiety, fullness

**Happy**:
How contented the pet is, shown as four hearts. A snack or a game restores one heart.
_Avoid_: Mood, happiness, joy

**Heart**:
One quarter of a Hunger or Happy meter, and the unit the player reasons in. The value
beneath is continuous; the heart is what is drawn.
_Avoid_: Pip, bar segment, unit

**Weight**:
A number that food raises and games lower. Past a threshold the pet is **overweight**.
_Avoid_: Mass, size, fatness

**Overweight**:
The state past the weight threshold: a rounder silhouette, a higher illness chance, and
a mark against care quality.
_Avoid_: Fat, obese

**Discipline**:
How well-behaved the pet is, raised by scolding a needless call and lowered by indulging
one. Shown as a bar.
_Avoid_: Obedience, training, manners

**Illness**:
A state entered from poop left standing, being overweight, ignored calls, or a low random
baseline. Marked by a skull and cured only by a full course.
_Avoid_: Sickness, disease, being unwell

**Silhouette**:
The pet's outline, which changes with weight. One of the few honest reads on pet state
available at a glance, since the hearts are hidden in ambient.
_Avoid_: Shape, body, outline

### Time and life

**Hatch instant**:
The exact moment the pet emerged. All ages are measured from it, which keeps age immune
to timezone and DST.
_Avoid_: Birthday, birth date, start time

**Life clock**:
The monotonic instant the simulation runs on. It cannot step: every accepted clock
correction is absorbed by an offset, so an NTP fix, a DST shift or a user edit is invisible
to the pet. The only clock that can age it. Defined in ADR-0011.
_Avoid_: Game clock, elapsed time, uptime, monotonic time

**Civil clock**:
Local wall-clock time — what a clock in the room says. Correctable, steppable, and read only
by quiet hours, the morning wake and the care log. A wrong one costs a badly-timed night and
nothing else. Defined in ADR-0011.
_Avoid_: Wall clock, local time, real time, UTC

**Year**:
Twenty-four hours of wall-clock time from the hatch instant, whether the device was on,
asleep or flat. The unit the pet's age is counted in. Derived on demand rather than marked:
nothing happens at the boundary, because age-driven events land at the morning wake.
_Avoid_: Day, cycle, tick

**Lifespan band**:
The range of ages within which old age arrives. Poor care shortens it; excellent care
extends it. There is no fixed death day.
_Avoid_: Max age, life expectancy

**Stage**:
One span of the pet's life between evolutions. Care quality is evaluated per stage, not
across the whole life.
_Avoid_: Phase, form, level, age bracket

**Sleepy**:
The pet's own signal that it wants the lamp dimmed. Sleep is pet-initiated; the player
cannot start it.
_Avoid_: Tired, bedtime, night mode

**Sleep**:
The pet's rest state, entered when a sleepy pet has its lamp dimmed and left at the morning
wake. Meters still drain, but slowly.
_Avoid_: Night, dormant, idle

**Quiet hours**:
The player's declared night. Mandatory, 4–10 hours, and absolute: they suppress the buzzer
and the idle animation, they anchor the morning wake, and they pause the rescue window.
The one setting the player gives that three mechanics depend on.
_Avoid_: Do not disturb, night mode, silent hours, bedtime

**Morning wake**:
The pet leaving sleep at the end of quiet hours. Surfaces the calls that quiet hours
suppressed overnight, carries any evolution, and spends one of the four clearing refreshes.
A late bedtime therefore makes a short night, not a late morning.
_Avoid_: Wake-up, sunrise, dawn, alarm

### Care and its record

**Care event**:
One act by the player that satisfies a need: a meal, a snack, a cleaning, a dose, a
scold, or dimming the lamp.
_Avoid_: Interaction, action, input

**Meal**:
Food that restores one Hunger heart and adds one weight. Refused when Hunger is full.
_Avoid_: Food, dinner, rice

**Snack**:
Food that restores one Happy heart and adds two weight. Never refused.
_Avoid_: Treat, candy, sweet

**Course**:
The full two-dose treatment that cures an illness, the doses separated by hours. One dose
alone does nothing lasting.
_Avoid_: Medicine, cure, treatment

**Call**:
The pet signalling that it wants something. Cleared only by satisfying the underlying
need — attending the device merely silences the buzzer.
_Avoid_: Alert, beep, notification, cry

**Needless call**:
A call with no underlying need. The correct response is to scold, not to indulge.
_Avoid_: False alarm, spoiled call, fake call

**Scold**:
Telling off a pet that made a needless call. Raises Discipline. Scolding a genuine call
costs a Happy heart.
_Avoid_: Discipline (the verb), punish, tell off

**Care mistake**:
A recorded failure of care — a call left unsatisfied past its threshold, poop left
standing, illness left untreated, or the lamp left up on a sleepy pet.
_Avoid_: Error, fault, neglect event

**Care quality**:
How well the pet was looked after during one stage, derived from the care mistakes logged
within it. An input to evolution alongside play style.
_Avoid_: Care score, happiness rating, performance

**Care log**:
The written, timestamped record of care events and care mistakes, shown in the paper
chrome. The player-facing surface for the mistake count, which itself stays hidden.
_Avoid_: History, journal, feed, activity log

**Rescue window**:
The signposted period during which a visibly dying pet can still be saved. Death is never
sudden, and it never happens overnight: the window pauses through quiet hours, so a dying
pet found in the morning is always still savable.
_Avoid_: Grace period, last chance

### The device

**Ambient**:
The mode where the pet is visible and idle, refreshes are rare, and the hearts are hidden.
_Avoid_: Idle mode, standby, home screen

**Session**:
The mode entered by touching the device within an attention window, where the hearts are
visible and the full refresh budget is available.
_Avoid_: Active mode, awake, interactive mode

**Attention window**:
The bounded span during which the device is touch-responsive. Opened by a pickup, a buzz,
or the end of a session. Outside it the panel is unpowered and touch does nothing.
_Avoid_: Awake window, touch window, warm period, listening mode

**Pickup**:
Motion that re-energises a powered-off board through the power latch. A cold boot rather
than an interrupt, and it cannot be disarmed. It opens an attention window, not a session.
_Avoid_: Lift, wake-on-motion, shake, nudge

**Torpor**:
The state the device enters when charge runs out — a final snapshot, then power off,
holding a fixed tableau. The pet is not dead: unpowered time accrues neglect but cannot
kill. Only a charger ends it.
_Avoid_: Flat, dead battery, shutdown, hibernation

**Memorial**:
The state after the pet dies. No decay, no calls, no care events, held at zero power. The
only thing a session offers is a new egg.
_Avoid_: Death screen, graveyard, game over, epitaph

**Lamp**:
The depicted room light the player raises and dims. It renders a dark room; it is not
device hardware.
_Avoid_: Light, backlight, brightness

**Paper chrome**:
The greyscale surround — dithered greys, hairline rules, serif type — that frames the
pet's region and carries the care log.
_Avoid_: UI, frame, border, HUD

**Refresh intent**:
What a repaint is *for* — one of four: animate the pet, flip a discrete element, settle
the chrome, or clear the panel. Core names an intent; only the port knows which waveform
it becomes. Defined in ADR-0008.
_Avoid_: Refresh mode, waveform, update mode, epd mode

**Clearing refresh**:
The full greyscale pass that discharges accumulated ghosting. It flashes, so it is spent
at a moment that already justifies a visual break — never on a timer.
_Avoid_: Full refresh, flush, deghost, reset

**Rest frame**:
The pose an animation returns to. Returning to it closes the loop that keeps the pet's
region electrically balanced, so a run that ends on its rest frame owes nothing.
_Avoid_: Idle frame, base pose, neutral, default sprite

**Play style**:
Which of the four games the player favours, accumulated over a stage. An input to
evolution alongside care quality.
_Avoid_: Preference, playstyle, game history

### The architecture

Engineering vocabulary rather than game vocabulary, kept here so the project has one
ubiquitous language rather than two. Defined in ADR-0004 and ADR-0006.

**Core**:
The pure simulation library. Owns pet state, evolution, game rules, the save format, the
ambient/session machine and every pixel. Names no port, allocates nothing, and depends on
nothing.
_Avoid_: Engine, domain layer, business logic, model

**Step**:
One pure transition — `(State, Event, Instant)` in, `(State, Effects)` out. Total,
deterministic, and evaluable at compile time.
_Avoid_: Tick, update, advance, frame

**Event**:
Anything entering core: a touch, a timer firing, a completed save, an NTP sync, a battery
warning, a failed write. The single input channel.
_Avoid_: Message, signal, command, input

**Effect**:
An intent core returns for the shell to carry out — repaint a region, buzz, persist,
wake at an instant. Fire-and-forget; its outcome arrives later as an event.
_Avoid_: Action, side effect, command, output

**Regime boundary**:
An instant at which the rate or the rules change — a quiet-hours edge, the morning wake,
sleep onset, a stage transition, a call threshold, death. Core crosses a gap by integrating
in closed form between them rather than by ticking. Defined in ADR-0011.
_Avoid_: Tick, step boundary, event time, checkpoint

**Wake deadline**:
The instant core asks to be woken next — the earliest moment at which something visible
changes or an alert is due, and always a life clock instant. The only thing core says about
time passing.
_Avoid_: Tick interval, wake cadence, timer, poll rate

**Armed instant**:
What the shell actually asked the RTC for, which is never earlier than the wake deadline and
may be a hop short of it. The shell's number, not core's: it is what distinguishes a timed
wake from a pickup.
_Avoid_: Alarm, countdown, timer, next wake

**Port**:
A concept describing one piece of hardware. Nine exist, and core never calls one.
_Avoid_: Interface, driver, HAL, service

**Adapter**:
One concrete implementation of a port — real hardware for the device, simulated for the
simulator. Deliberately dumb: logic worth testing belongs in core.
_Avoid_: Implementation, backend, provider

**Shell**:
The only impure code in the tree. Holds adapters, pumps events into core, executes the
effects that come out. One per app, concrete and small.
_Avoid_: Runtime, main loop, host, driver

**Rest state**:
How the shell idles between wakes — the board de-energised, or light sleep. Chosen from
the wake deadline, and never named by core.
_Avoid_: Deep sleep, standby, idle, sleep (which is the pet's)

**Replay log**:
A recorded sequence of events which, with the pet's seed, reproduces a life exactly. The
basis of fast-forward, bug reproduction and balance testing.
_Avoid_: Trace, event stream, recording, history
