# The keyboard belongs to the phone, and core never learns the word WiFi

**Enrolment** is never a gate on the pet. What blocks hatching is a confirmed **civil clock**, and
there are two first-class doors to one — connect, or set it yourself. The connecting door renders the
device's own WPA2 **enrolment portal** as a `WIFI:` QR code: the phone's camera joins it in one tap
and the WPA2 passphrase gets typed on a real keyboard. **Core owns the flow and every pixel of it
and never names the radio** — it emits `BeginEnrolment`/`EndEnrolment` and receives
`EnrolmentReady{join_hint}`, `EnrolmentPeerPresent`, `EnrolmentSucceeded` and `EnrolmentFailed`.

The ticket asked whether an on-screen keyboard could be avoided entirely. It can, and the answer is
not to eliminate the keyboard but to **relocate** it. Every mechanism except a file on the SD card
ends with a passphrase typed somewhere; the only question is whose screen. A phone has a good
keyboard. This device has a 7 fps two-point panel.

Core's half is [ADR-0008](0008-refresh-policy.md)'s move applied to the radio: **core names an
intent, and only the port knows what hardware it becomes.** `join_hint` is an opaque string core
draws as a QR without understanding a byte of it, so the pure simulation library holds a whole
enrolment flow while remaining innocent of SoftAP, SSID and 2.4 GHz.

## Considered options

- **An on-screen keyboard.** The thing the ticket asked us to avoid, and it deserves its rejection
  written down: a WPA2 passphrase is up to 63 characters across four character classes, entered on a
  panel with **~160 ms per mono frame** ([ADR-0008](0008-refresh-policy.md)) and two-point touch with
  no hardware gestures (note [`0006`](../research/0006-input-inventory.md)). Every other option here
  exists to not do this.
- **ESP-IDF's `wifi_provisioning` component with Espressif's phone app.** Free SRP6a-secured
  transport over SoftAP or BLE, and it solves the same keyboard problem. Rejected because the ritual
  becomes *"install a developer's app"*, which is an ugly seam in an otherwise handmade object, and
  because it puts a vendor's app store listing on the critical path of hatching an egg.
- **DPP / Wi-Fi Easy Connect.** **Parked, not rejected** — it is the better answer wherever it
  works: the device shows a QR, the phone's WiFi settings push credentials in, and *nobody types
  anything at all*. ESP-IDF supports the enrollee role. Rejected as the primary because it is
  Android-only in practice and its exposure in stock Settings is OEM-dependent **[U]**, so the
  success of hatching would depend on which phone is in the room. Worth revisiting as a second path.
- **Credentials in a `wifi.txt` on the SD card.** Near-zero code. Rejected because it needs a
  computer, which makes re-enrolment away from a desk impossible and roaming effectively
  unsupported — and note 0005 already rules the SD card optional and never a save target.
- **Baked in at flash time via `menuconfig`.** Zero device code, and genuinely defensible for a
  single unit. Rejected because note [`0004`](../research/0004-cloud-backup-and-credentials.md)
  names *us* as the most likely leak — *"the credential gets committed to the repo, pasted into an
  issue, or baked into a firmware image"* — and because re-enrolment would mean a rebuild.
- **The shell owning the enrolment screens.** The honest rival, and simpler: no QR encoder in core,
  no ADR-0004 friction. Rejected because it splits *who draws* in two, which is the thing ADR-0004
  exists to prevent; because the flow would then be untested by construction; and because it would be
  **invisible in #20's simulator**, which is where the failure branches — wrong passphrase, captive
  portal, AP vanished mid-handshake — are otherwise near-unreproducible.
- **A fifth mode.** [ADR-0009](0009-pickup-listens-touch-attends.md)/#11 settled on four and
  deliberately demoted everything else to *conditions*. Enrolment is something the player does while
  standing at the device with the panel up and the full refresh budget available, which is the
  definition of a **Session**.
- **An open enrolment portal.** Rejected because the QR makes WPA2 free: the passphrase rides in the
  QR payload, so the player types nothing either way, and a neighbour can no longer join the portal
  during the minutes it is up and re-point the device at their own network.
- **Association as the success test.** Rejected — see below. It is the option that quietly breaks
  the captive-portal case.
- **Remembering several networks.** Rejected: this is a desk companion, and a remembered set means a
  scrollable list, a delete affordance and an ordering rule on a 7 fps panel. Note
  [`0012`](../research/0012-power-budget.md) removes the only argument that could have decided it on
  other grounds — WiFi is *"not a lever at all"* at 0.7 % of the weekly budget.

## Consequences

- **Both doors converge on one screen, and that is what makes failure cheap.** The timezone confirm
  that [ADR-0011](0011-two-clocks-and-the-life-clock-cannot-step.md) requires and the mandatory
  **quiet hours** setting are the same question wearing two hats — both are one-time civil-time
  questions asked of a player who is standing right there. They merge into a single *"your day"*
  screen, and the two doors differ **only in whether the time field arrives pre-filled**. So a failed
  enrolment needs no error path: it falls through to that screen with an empty field, which *is* the
  manual door.
- **The ritual is egg → doors → "your day" → hatch.** The egg is the first thing on screen and costs
  nothing to hold at zero power; touching it opens the session. The hatch spends the `Clear` that
  ADR-0008 and #11 already earmark for hatching, so the setup screens' ghosting is discharged by the
  ritual itself rather than by a timer — which ADR-0008 said it never wanted to do.
- **Enrolment succeeds on a plausible NTP answer, not on association.** A device can complete a WPA2
  handshake, take a DHCP lease and still be behind a splash page with no route anywhere. Testing
  reachability rather than link **folds the captive-portal case into the success test** instead of
  needing a detector, and ADR-0011 wants that sync at first setup anyway, so the test is free.
  Distinguishing *portal* from *no route* is a `generate_204` probe, worth its ten lines only because
  the two remedies differ.
- **This device can never traverse a captive portal**, and that is a stated non-goal rather than a
  bug. It has no browser and no way to render one. The only honest remedy is a different network, and
  the message says so rather than retrying forever.
- **ADR-0011's plausibility floor has a pre-hatch hole.** It is defined as *"nothing earlier than the
  hatch's own civil time"*, which does not exist at first enrolment. The floor there is the
  **firmware build timestamp** — a compile-time constant, always in the past, always after 2020.
- **Credentials are trialled in RAM and committed to NVS only on success.** A mistyped passphrase is
  therefore never persisted, and the invariant that buys is worth more than the code it costs: **any
  device that boots with credentials in flash knows they worked at least once.**
- **The portal has no timer of its own.** Enrolment holds the session open exactly as an open game
  round does (#11), and the sharpening is that **a joined phone is evidence of attention** — the
  3-minute abandonment timer runs only while no peer is connected. Finding the router password on a
  sticker behind the sofa costs nothing. One timeout concept, not two.
- **The device never raises an AP on its own.** Not on boot, not when its network goes missing, not
  after N failed backups. Power, the no-nag rule below, and the fact that an unattended AP is a
  standing invitation all agree. Enrolment is only ever entered deliberately, from a session.
- **The pet is silent about enrolment, and the paper chrome speaks once.** No **call**, no buzz, no
  animation change, nothing that reads as the *pet* wanting something — #11's alert vocabulary is
  reserved for care. One dated **care log** line at each hatch names what is actually lost, in the
  same register as ADR-0011's *"clock corrected +5h58m"*, and then scrolls away like every other line.
  No chrome glyph: #11 freezes the chrome and hides the hearts precisely so an unattended device
  shows nothing that reads as a demand, and a permanent status icon is a nag wearing a disguise.
- **One network, replaced on re-enrolment.** A moved device silently stops backing up, exactly as
  note 0004 already specifies — and, per ADR-0011, also silently loses DST correction and the silent
  post-**Torpor** clock recovery. Accepted, and recorded as accepted.
- **Re-enrolment needs no new time machinery.** The NTP result that arrives is an ordinary clock
  correction, which ADR-0011 already absorbs into the offset and writes to the care log. The pet does
  not notice; the player reads *"clock corrected +1h00m"* and understands why bedtime was odd all
  autumn.
- **Core gains a QR encoder.** A ~45-character `WIFI:S:…;T:WPA;P:…;;` payload is a small symbol into
  a fixed static buffer, so [ADR-0005](0005-allocation-free-core.md) holds. Someone could reasonably
  argue `join_hint` is a port fact wearing a disguise; the defence is that core never interprets it,
  and the simulator coverage bought is the whole point.
- **[ADR-0011](0011-two-clocks-and-the-life-clock-cannot-step.md)'s final bullet is amended by this
  one.** It concluded that requiring the network at first setup makes provisioning *hatch-blocking*.
  It does not: manual entry is a peer door rather than a fallback, so #19's founding premise —
  *"provisioning must never be a gate on play"* — survives almost intact. Everything else in ADR-0011
  stands.
- **The timezone endpoint is still unnamed.** ADR-0011 says *"a timezone endpoint proposing"* and
  never picks one. On a device with no update path that is an unowned third-party dependency, and it
  is now on the critical path of the connecting door. Split out to its own research ticket.

Three things here are **unverified [U]** and should be settled by building rather than by argument:
whether iOS reliably pops the Captive Network Assistant for our portal (historically finicky; the
fallback is *"open a browser yourself"*), the exact QR version a ~45-character payload lands in, and
that the RAM-trial-then-commit-to-flash dance behaves cleanly in `esp_wifi`. Following ADR-0008 and
ADR-0009: **the model is what this ADR commits to.**
