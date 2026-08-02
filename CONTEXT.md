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

**Year**:
Twenty-four hours of wall-clock time from the hatch instant, whether the device was on,
asleep or flat. The unit the pet's age is counted in.
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
The pet's rest state, entered when a sleepy pet has its lamp dimmed. Meters still drain,
but slowly.
_Avoid_: Night, dormant, idle

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
sudden.
_Avoid_: Grace period, last chance

### The device

**Ambient**:
The mode where the pet is visible and idle, refreshes are rare, and the hearts are hidden.
_Avoid_: Idle mode, standby, home screen

**Session**:
The mode entered by attending the device, where the hearts are visible and the full
refresh budget is available.
_Avoid_: Active mode, awake, interactive mode

**Lamp**:
The depicted room light the player raises and dims. It renders a dark room; it is not
device hardware.
_Avoid_: Light, backlight, brightness

**Paper chrome**:
The greyscale surround — dithered greys, hairline rules, serif type — that frames the
pet's region and carries the care log.
_Avoid_: UI, frame, border, HUD

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

**Replay log**:
A recorded sequence of events which, with the pet's seed, reproduces a life exactly. The
basis of fast-forward, bug reproduction and balance testing.
_Avoid_: Trace, event stream, recording, history
