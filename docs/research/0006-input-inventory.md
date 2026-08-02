# What input verbs the M5Paper S3 hardware actually gives us

Research note for [#6](https://github.com/r4stered/papergotchi/issues/6). Everything below is
about the **M5PaperS3** (`board_M5PaperS3`, ESP32-S3R8). The original M5Paper (v1, ESP32) is a
different board with different parts and a different pinout; where a fact is v1-only it is
labelled as such and must not be carried over.

## The question

The layout (#17), all four games (#10, #15, #16), and the ambient/session state machine (#11)
are all downstream of one thing: which player actions the silicon can actually observe, how
much each costs, and which of them can reach a sleeping device. The marketing answer — "two-point
touch **and gestures**", "**wake-up on lift**" — is not the engineering answer. This note
establishes the engineering answer from the schematic, the vendor board-support source, the part
datasheets and the ESP-IDF sleep documentation, then proposes a verb mapping against it.

---

## Established facts

### Sources

| Short name | What it is |
|---|---|
| **[Docs]** | M5Stack PaperS3 product documentation — <https://docs.m5stack.com/en/core/PaperS3> |
| **[Sch]** | PaperS3 schematic V1.0 — <https://m5stack-doc.oss-cn-shenzhen.aliyuncs.com/517/sch_papers3_V1.0.pdf> |
| **[Store]** | M5Stack shop listing — <https://shop.m5stack.com/products/m5paperS3-esp32s3-development-kit> |
| **[M5U]** | m5stack/M5Unified @ `4fb4447` — <https://github.com/m5stack/M5Unified> |
| **[M5GFX]** | m5stack/M5GFX @ `729297d` — <https://github.com/m5stack/M5GFX> |
| **[GT911-PG]** | Goodix *GT911 Programming Guide* Rev.10 — <https://www.lcd-module.de/fileadmin/eng/pdf/zubehoer/GT911_Programming_Guide_Rev.10.pdf> |
| **[GT911-DS]** | Goodix *GT911 Datasheet* Rev.09, as hosted by M5Stack — <https://m5stack.oss-cn-shenzhen.aliyuncs.com/resource/docs/datasheet/core/m5paper/gt911_datasheet.pdf> |
| **[BMI270]** | Bosch Sensortec *BMI270 Datasheet*, document BST-BMI270-DS000-08 — <https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bmi270-ds000.pdf> |
| **[IDF]** | ESP-IDF v6.0, ESP32-S3 *Sleep Modes* — <https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/system/sleep_modes.html> |

### The GPIO map, as wired

Read off the ESP32-S3R8 module symbol (U2) in **[Sch]**, cross-checked against the pin tables in
**[M5U]** `src/M5Unified.cpp` and **[Docs]**. Only the pins that matter for input are listed.

| GPIO | Net | Role | RTC-capable? |
|---|---|---|---|
| G0 | `GPIO0_SYS_LED` | boot strap; also the system LED, driven as PWM by **[M5U]** `src/utility/Power_Class.cpp:1003` (`M5PaperS3 : LED = GPIO0`) | yes |
| G3 | `ADC_VBAT` | battery sense (ADC1) | yes |
| G4 | `CHG_STAT` | charger status from the LGS4056H | yes |
| G5 | `USB_DET` | USB/charger present | yes |
| G41 / G42 | `SYS_SDA` / `SYS_SCL` | shared I²C: GT911, BM8563 RTC, BMI270 | yes |
| G44 | `PWROFF_PLUSE` *(sic)* | power-off pulse to the PMS150G power controller; `power_hold` pin in **[M5U]** `src/M5Unified.cpp:258` | no |
| G48 | `TP_INT` | GT911 touch interrupt | **no** |

That last row is the single most consequential fact in this note. ESP32-S3's RTC GPIOs are
**"0-21"** **[IDF]**, and `esp_sleep_enable_ext0_wakeup()` returns `ESP_ERR_INVALID_ARG` "if the
selected GPIO is not an RTC GPIO" **[IDF]**. G48 is outside that range.

### Touch — GT911

**What is fitted.** GT911 two-point capacitive panel on the shared I²C bus at address `0x14` or
`0x5D`, INT on G48, **no reset line to the host** **[M5GFX]** `src/M5GFX.cpp:2146-2165`
(`cfg.pin_int = GPIO_NUM_48; cfg.pin_sda = GPIO_NUM_41; cfg.pin_scl = GPIO_NUM_42; cfg.freq =
400000`; `cfg.pin_rst` is left at its `-1` default from `src/lgfx/v1/Touch.hpp:50`). The schematic
agrees: on the 8-pin touch FPC (J-connector) `TP_RST` is pulled to `SOC_VDD` through R51 10K with
R45 marked `NC`, and no `TP_RST` net reaches the module **[Sch]**. Panel coordinate space is
540 × 960 portrait with `offset_rotation = 1` **[M5GFX]** `src/M5GFX.cpp:2153-2158`.

**Which gestures the silicon reports.** The GT911 *does* have a hardware Gesture mode. Per
**[GT911-PG]** §6.1(c): the host enters it by "sending I²C command 8 to `0x8046`, then to
`0x8040`"; thereafter "when GT911 detects any finger swipe (for a sufficiently long distance),
double-tap or writing of specified letters on touch screen, INT will output a pulse for longer
than 250us or output high level". The gesture type is read from `0x814B`, the trajectory features
from `0x814D`–`0x816C`, and the raw trajectory points from `0x9420`–`0x951F` **[GT911-PG]** §8.
The gesture vocabulary is enumerated in the config registers `0x8075` `Gesture_Switch1` (swipe
left / swipe up / swipe right, plus letters `w o m e c`) and `0x8076` `Gesture_Switch2` (letters
`z s ^ > V`, double-tap, swipe down) **[GT911-PG]** §3. `0x8056` bits 7–4 set the gesture wakeup
pulse width in units of 250 µs.

**Which gestures we actually get.** None of that is used. The M5GFX driver reads exactly two
things — the buffer-status byte at `0x814E` and *n* × 8 bytes of point data — and writes zero back
to `0x814E` to release the buffer **[M5GFX]** `src/lgfx/v1/touch/Touch_GT911.cpp:28,152-207`. It
never touches `0x814B`, `0x8046` or the gesture switches. Everything the application sees as a
"gesture" is **synthesised in software** by `m5::Touch_Class`: `touch_begin`, `touch_end`, `hold`,
`flick`, `drag` and a click counter, with a 500 ms hold threshold and an 8 px flick threshold
**[M5U]** `src/utility/Touch_Class.hpp:14-34,102-118`. The vendor stack polls at a floor of 4 ms
(`TOUCH_MIN_UPDATE_MSEC`) **[M5U]** `src/utility/Touch_Class.hpp:40`.

So the honest inventory is: **two simultaneous points with x/y/size/track-id, plus
software-derived tap, double-tap-by-count, hold, flick and drag.** Hardware gesture mode is
available in principle but would have to be enabled by us, is config-dependent, and — see below —
buys nothing we can use, because its output pin cannot wake this chip.

**Scan rate and power state.** "GT911 can switch between Normal mode and Low Power mode
automatically by default… If no touch is detected within that period, GT911 enters Low Power mode
(low-speed scan)" **[GT911-PG]** §6.1. Normal mode: "fastest coordinates refreshing cycle is
5ms-20ms (subject to configuration. One step is 1ms)". Green mode: "the scanning cycle for GT911
is about 40ms. It automatically enters Normal mode if any touch is detected". The idle timeout
before Green is configurable 0–15 s in 1 s steps.

**Sleeping the panel.** `0x8040 = 0x05` is "Screen off" **[GT911-PG]** §3.1, and that is what
`Touch_GT911::sleep()` sends, after driving INT low **[M5GFX]**
`src/lgfx/v1/touch/Touch_GT911.cpp` (`sleep()`). Getting back out is the awkward part: "the host
can employ INT high-level wakeup or reset wakeup… host drives INT output high for 2ms~5ms, and
then drives INT input floating… The time interval between issuing the screen-off command and
wakeup should be longer than 58ms" **[GT911-PG]** §6.1(d). Because `TP_RST` is not wired to the
host on this board, **INT high-level wakeup is the only route back**, and a sleeping GT911 by
definition raises no interrupt of its own. *A slept touch panel cannot be the thing that ends
ambient.*

**Can touch wake the device?** From **light sleep**, yes; from **deep sleep**, no. M5Unified says
so in a comment and works around it only in the light-sleep path:

> `// M5PaperS3 touch interrupt pin (GPIO48) is not RTC IO`
> `// and therefore not supported in EXT0 wakeup`

**[M5U]** `src/utility/Power_Class.cpp:1333-1341`, which then falls back to
`gpio_wakeup_enable(48, GPIO_INTR_LOW_LEVEL)` + `esp_sleep_enable_gpio_wakeup()`. **[IDF]** lists
that API under "GPIO Wakeup (Light-sleep Only)". Note the trap: `Power_Class::deepSleep(us,
touch_wakeup = true)` has **no** such guard — it calls `esp_sleep_enable_ext0_wakeup(48, false)`
unconditionally **[M5U]** `src/utility/Power_Class.cpp:1279-1290`, which will return
`ESP_ERR_INVALID_ARG` and silently leave the device with no touch wake source. Any port we write
must not inherit that bug.

### Physical buttons

**Exactly one, and the ESP32 cannot read it.**

**[Docs]** lists "Button: 1× physical (control, power, reset, download)" and "Click the side button
to power on, double-click the side button to power off". The schematic shows why that is the whole
story: S1 (`TS_015` tactile switch) pulls the `SW_PWR` net down against a 10K to `SYS_BAT`, and
`SW_PWR` goes to **pin 1, `PA4/CIN+`, of U1 — a `PMS150G-U06`** annotated *定制带程序芯片*
("custom pre-programmed chip") **[Sch]**. The PMS150G's four signal pins are `PA3` = `MPWR_EN`,
`PA4` = `SW_PWR`, `PA5` = `GPIO0_SYS_LED`, `PA6` = `nINT_STAT_TRIG`. `SW_PWR` appears nowhere
else in the netlist and reaches no ESP32-S3 GPIO.

Consistently, **[M5U]**'s `M5Unified::update()` has no GPIO button read for `board_M5PaperS3` at
all **[M5U]** `src/M5Unified.cpp:2880-3200`. The only `BtnA`/`BtnB`/`BtnC` it can offer are
*virtual* — a strip along one edge of the touch panel, height 0 unless the application opts in
**[M5U]** `src/M5Unified.cpp:2900-2937, 3234-3243`.

**Deep-sleep wake sources on this board, exhaustively:**

- **RTC timer** (`esp_sleep_enable_timer_wakeup`) — always available **[IDF]**.
- **ext0/ext1 on an RTC GPIO 0–21** **[IDF]**. Of the pins actually routed to something a player
  can affect, that is `G4 CHG_STAT` and `G5 USB_DET` — i.e. *plugging in a charger*. Not touch
  (G48), not the button (not wired), not the IMU (not wired).
- **Full power-off and cold boot.** This is the board's real low-power state. `nINT_STAT_TRIG`,
  the BM8563 RTC's `INT` output, is diode-OR'd (D2/D3/D15, 1N4148WT, 5.1K pull-ups) onto the
  PMS150G's `PA6` and **not** onto any ESP32 GPIO **[Sch]** — hence `_rtcIntPin` is left unset for
  this board in **[M5U]** `src/utility/Power_Class.cpp:262-268`. `Power_Class::_powerOff()` pulses
  G44 low/high five times at 50 ms, with the comment "For PaperS3, the power cannot be turned off
  simply by setting the GPIO to LOW, so a loop is performed to ensure that the power is turned off
  by repeatedly outputting a pulse" **[M5U]** `src/utility/Power_Class.cpp:1156-1167`. The
  PMS150G then drops `MPWR_EN`; the RTC alarm or the side button brings it back — as a **cold
  boot**, not a sleep wake.

> **v1 contrast, do not carry over.** M5Paper v1 has three real side buttons on GPIO37/38/39
> **[M5U]** `src/M5Unified.cpp:2962-2966`, and its touch INT is on GPIO36 — an RTC GPIO on the
> original ESP32, so `_wakeupPin = GPIO_NUM_36` there really does arm ext0 **[M5U]**
> `src/utility/Power_Class.cpp:408-414`. Neither is true on the S3.

### The IMU — BMI270

**The part.** `U11` is a **BMI270** on the shared I²C at `0x68` **[Sch]**, **[Docs]**. Confirmed in
software: `board_M5PaperS3` reaches `BMI270_Class`, which checks `CHIP_ID == 0x24` **[M5U]**
`src/utility/imu/BMI270_Class.cpp:35-46`. Its `CSB` pin is tied to `IMU_VDD` (I²C mode) and
`SCx`/`SDx` go to `SYS_SCL`/`SYS_SDA` **[Sch]**. `IMU_VDD` comes from its **own** LDO (U10,
`ME6203A33M3G`) fed from `VBUS_PRE`, i.e. a rail that survives main-power-off **[Sch]** — which is
why **[Store]** can quote "Low power mode: DC4.2V/9.28uA (main power off, gyroscope in low power
mode)" and "Standby mode: DC4.2V/949.58uA (main power off, gyroscope on)".

**The interrupt lines go nowhere the CPU can see them.** On schematic V1.0, U11's `INT1` (pin 4)
and `INT2` (pin 9) carry **no net label and no connection** — every other pin of the part has one
(`SCx`→`SYS_SCL`, `SDx`→`SYS_SDA`, `CSB`→`IMU_VDD`, `VDDIO`/`VDD`→`IMU_VDD`) **[Sch]**. There is
no `E_TRIG` net anywhere in the V1.0 netlist. A community thread on M5Stack's own forum states
the same conclusion by a different route — "the IMU interrupt line (INT1) is **not** connected to
any ESP32S3 GPIO and therefore it **cannot** wake up ESP32S3 from deep or light sleep" — while
claiming it reaches the PMS150G via a signal called `E_TRIG`
(<https://community.m5stack.com/topic/7748/paper-s3-wake-on-imu>; secondary source, one
contributor, not M5Stack staff). **Both readings agree on the decision-relevant point:** the IMU
cannot interrupt the ESP32-S3 in any sleep state. At most, motion can ask the PMS150G to
re-energise the board, which is a cold boot. The "wake-up on lift" in **[Store]** means exactly
that, and nothing weaker.

**What it can do on-chip, without the CPU.** This is the good news, and it is what the BODY
step-walk mode needs. **[BMI270]** §1 lists the feature set: "Significant motion / Any motion /
Motion detect / No motion / Stationary detect / Wrist wear wakeup / Wrist worn step counter and
detector / Activity change recognition / Push arm down / Pivot up / Wrist jiggle / Flick in/out".
Relevant details:

- **Step counter.** A free-running 32-bit count, readable over I²C with no interrupt involved:
  `SC_OUT_0` at register `0x1E`, "Step counting value byte-0" **[BMI270]** §5.2.30, continuing
  through `0x1F`/`0x20`/`0x21`; the same value is mirrored in the feature window at page 0 offsets
  `0x30`/`0x32` **[BMI270]** §4.8.8. Enabled by `SC_26.en_counter`; `SC_26.reset_counter` clears
  it; `SC_26.watermark_level` optionally fires an interrupt every *n* steps **[BMI270]** §4.8.8.
  The algorithm is explicitly "optimized for high accuracy in wrist use-case applications"
  **[BMI270]** §4.8.8 — see uncertainties.
- **Activity recognition.** still / walking / running / unknown, with change interrupts
  **[BMI270]** §4.8.5.
- **Any-motion, no-motion, significant motion.** Significant motion implements the Android
  `SIGNIFICANT_MOTION` semantics, with a configurable block size "expressed in 50 Hz samples
  (20 ms). Default value is 0xFA=5sec" **[BMI270]** §4.8.4.
- **Cost of running the feature engine.** With `ACC_CONF.acc_filter_perf = 0` (low power), "the
  ODR must be set to minimum 50 Hz" for the features to run at their designed rate **[BMI270]**
  §4.7 minimum-bandwidth note; below that "the features are still evaluated, the same number of
  samples are evaluated, but they are sampled at the lower rate".

Current consumption, from the electrical-characteristics table **[BMI270]** §2 and the power-mode
table §4.5:

| Mode | Condition | Typ. |
|---|---|---|
| Suspend (A+G) | — | **3.5 µA** |
| Low power, accel only | ODR 25 Hz | **10 µA** |
| Low power, accel only | "down to 4 µA", ODR-dependent | 4 µA |
| Low power, IMU (A+G) | ODR 25 Hz | 420 µA |
| Normal, accel only | ODR max | 210 µA |
| Normal, IMU (A+G) | ODR max | 685 µA |
| Performance, IMU (A+G) | ODR max | 970 µA |
| Configuration mode | — | 120 µA |

So: **accelerometer-only low-power at 50 Hz with the step counter running is a low-tens-of-µA
budget**, and the CPU's only involvement is a four-byte I²C read whenever it happens to be awake.
That is genuinely cheap. Running the **gyro** — which a tilt-maze needs for anything better than
gravity-vector tilt — costs 400–970 µA and is a session-only expense.

**What the vendor library gives us.** `BMI270_Class` uploads the 8 KB Bosch config blob and
enables the feature engine (`_upload_file(bmi270_config_file, …)`, then `INIT_CTRL = 0x01`), and
it defines the constants we need (`SC_OUT_0_ADDR = 0x1E`, `FEAT_PAGE_ADDR = 0x2F`,
`INT1_MAP_FEAT_ADDR = 0x56`, `PWR_CONF_ADDR = 0x7C`) **[M5U]**
`src/utility/imu/BMI270_Class.cpp:35-88`, `src/utility/imu/BMI270_Class.hpp:13-70`. But its public
API is only `getAccel`/`getGyro`/`getTemp`/`getImuRawData` — **there is no step-count accessor**,
and `begin()` sets `PWR_CONF = 0x00`, explicitly *disabling* advanced power save. Our BMI270 port
will have to configure the low-power/step-counter path itself.

### Battery and charge detection

- **Voltage.** `ADC_VBAT` on **G3**, ADC1, 12-bit, 12 dB attenuation, with ESP-IDF calibration
  (curve-fitting where supported), and a divider ratio of **2.0** **[M5U]**
  `src/utility/Power_Class.cpp:262-268` (`_batAdcCh = ADC1_GPIO3_CHANNEL; _batAdcUnit = 1;
  _adc_ratio = 2.0f;`) and `src/utility/Power_Class.cpp:1400-1450`. 12-bit over the ~0–3.1 V
  12 dB range is ≈0.76 mV/LSB at the pin, ≈1.5 mV/LSB referred to the battery. **The quantisation
  is irrelevant; ADC accuracy and calibration error dominate** and are worth tens of mV. Treat it
  as a coarse gauge, not a fuel gauge — there is no coulomb counter on this board.
- **Charging.** `CHG_STAT` on **G4**, read as a plain input; `is_charging` is `gpio_in(G4) ==
  false` **[M5U]** `src/utility/Power_Class.cpp:40, 262-263, 1987-1988`. This is the LGS4056H's
  open-drain `CHRG` pin with a 10K pull-up **[Sch]**, so it means *actively charging* and will
  release at charge termination even while still plugged in.
- **On a charger.** `USB_DET` on **G5**; **[Docs]** gives the threshold as "G5 (>0.2V = USB
  connected)". This is the signal that means "plugged in" regardless of whether the cell is still
  taking current.
- Both G4 and G5 are RTC GPIOs, so **"plugged in" and "charging started/stopped" are the only
  player-observable events that can wake this board from true deep sleep** **[IDF]**.

### Whole-board power figures, for calibration

**[Docs]** / **[Store]**: low power `9.28 µA` (main power off, gyro in low power), standby
`949.58 µA` (main power off, gyro on), operating `154.02 mA` (main power on). These are M5Stack's
own numbers under M5Stack's own conditions and belong to #12, not here — but they confirm the
architecture: the cheap state is *board off with the BMI270 alive*, not *ESP32 asleep*.

---

## Proposed verb mapping

The Gen1 verbs preserved by [ADR-0001](../adr/0001-verb-level-gen1-fidelity.md) are: feed a meal,
feed a snack, scold, clean, medicate, play, dim the lamp, check the status.

| Verb / game | Input | Mode | Justification |
|---|---|---|---|
| **Attend the device** (enter session) | Any touch; GT911 INT on G48 ends **light** sleep | ambient → session | Touch is the only rich input that can end a sleep state at all, and only light sleep. |
| **Check the status** | Implicit in entering a session — the hearts become visible; tap the care log region to scroll it | session | Matches `CONTEXT.md`: hearts are hidden in ambient, visible in session. No separate verb needed. |
| **Feed a meal** | Single tap on the meal target in the food tray | session | One tap, one care event. Distinct target rather than a meal/snack toggle, because Gen1's meal-vs-snack choice is a *choice*, and a refused meal (Hunger full) must be legible as a refusal, not as a mis-tap. |
| **Feed a snack** | Single tap on the snack target | session | As above. Never refused, so no failure mode. |
| **Clean** | Horizontal **flick** across the poop (software flick, ≥8 px, `Touch_Class`) | session | Sweeping is the one care event whose real-world gesture is literally a sweep. Cheap: it is a synthesised state, not a hardware gesture. |
| **Scold** | **Double-tap** on the pet (`getClickCount() == 2`) | session | Must be deliberate and must not collide with an idle tap on the pet, because scolding a genuine **call** costs a Happy heart. Double-tap is the cheapest distinct-from-tap gesture we have. |
| **Medicate (one dose)** | Single tap on the medicine target; the **course** is two doses hours apart | session | The two-dose structure ([ADR-0001](../adr/0001-verb-level-gen1-fidelity.md), #3) already supplies the deliberateness; the input does not need to. |
| **Dim the lamp** | **Hold** (≥500 ms default) on the depicted lamp | session | Dimming a wide-awake pet is a **care mistake** ([ADR-0002](../adr/0002-demand-driven-sleep.md)). A verb that can log a mistake must not be reachable by a stray tap. Hold is already synthesised for free. |
| **Play** | Tap the pet, or a play affordance in the paper chrome, to open the game selector | session | |
| **MIND — nonogram** | Taps to fill; flick across a run to fill/mark a line | session | Turn-based, zero timing pressure, indifferent to the ~5 fps refresh ceiling. Pure touch. |
| **INSTINCT — catch the falling food** | **Drag** along the bottom strip; the paddle follows the touch x | session | Continuous absolute position beats discrete taps at 5 fps: the player's finger is the state, so a dropped frame costs nothing. |
| **BOND — left/right guess, hide-and-seek** | Two large tap targets; tap to reveal | session | Binary and untimed. The most touch-frugal of the four; good for a quick care event. |
| **BODY — tilt-maze** | BMI270 polled over I²C at normal-mode ODR while the session is open | session only | No interrupt line exists, so tilt costs a CPU awake and 210–685 µA of IMU. That is affordable for a bounded session and unaffordable otherwise. |
| **BODY — step-walk** | BMI270 hardware step counter, accel-only low power ≥50 Hz; firmware reads the 32-bit `SC_OUT_*` delta on each wake | **ambient (background accrual)** | The one input that costs no CPU. Not a "game session" at all: steps accrue while the device sleeps or is carried, and the pet banks them. |
| **On a charger** | `USB_DET` (G5) level, optionally as an ext0/ext1 deep-sleep wake | ambient | Only player-caused event that can end true deep sleep. Use it for "resting/charging" presentation, buzzer suppression and a free full refresh. |
| **Power on / off** | Side button → PMS150G. **Not a care verb.** | — | Unreadable by firmware; a double-click kills the board. Mapping any game or care event to it is impossible. |

---

## Recommendation

**1. Touch carries every care verb; the physical button carries none.** There is one button, it is
owned by a separate power-management MCU, and the ESP32-S3 cannot read it. Any layout sketch or
state machine that reserves a button for "confirm" or "back" is designing for hardware that isn't
there. Corollary: the pet must be recoverable from any UI state by touch alone.

**2. Do not use GT911 hardware gesture mode.** Its only advantage over software synthesis is that
it can raise an interrupt from a low-power scan state — and that interrupt lands on G48, which
cannot wake this chip from deep sleep **[IDF]**. Software flick/hold/drag from `Touch_Class` gives
us a richer, better-documented, panel-config-independent vocabulary for free. Enabling gesture mode
would add an undocumented dependency on M5Stack's shipped GT911 config for no reachable benefit.

**3. Ambient must be light sleep, not deep sleep, whenever we want touch to end it.** This is the
hard constraint the mode state machine (#11) inherits, and it has a real power cost that #12 must
price. The plausible shape is a three-tier ladder rather than two modes:

- **session** — CPU awake, touch polled at 4–20 ms, IMU in normal mode if a BODY game is open;
- **ambient** — light sleep with `gpio_wakeup_enable(48, LOW)` armed, GT911 left scanning in Green
  mode (~40 ms cycle), BMI270 in accel-only low power with the step counter running, woken
  periodically by the RTC timer to animate;
- **deep rest** (overnight / **sleep** / flat battery) — full power-off via the G44 pulse with a
  BM8563 alarm set; touch is dead, the side button and the RTC are the only ways back, and it is a
  cold boot. This is exactly the state [ADR-0002](../adr/0002-demand-driven-sleep.md) predicted
  would be "the lowest-refresh state on the device", and it turns out to be the lowest-*power*
  state too — but at the price that a sleeping pet genuinely cannot be woken by touch.

**4. Split BODY into two mechanically different things, and say so.** Tilt-maze is a session game
that costs a CPU and a gyro. Step-walk is not a game session at all — it is ambient accrual of a
hardware counter, near-free, and it is the only way the pet observes the player when the screen is
untouched. That asymmetry is a feature: it gives BODY a **play style** signature that no other
branch has (you feed it by carrying the device, not by sitting at it), and it needs no new
hardware capability. It does mean the two halves of BODY have very different failure modes.

**5. Trade-offs we are accepting.**

- *Touch-only means mis-taps are care events.* Scold, and dimming the lamp, can both log a **care
  mistake**; both are therefore behind double-tap and hold rather than plain taps. This costs
  discoverability and will need a first-boot teach (#1's "first boot and hatching" fog).
- *Light-sleep ambient costs power we have not measured.* If the measured light-sleep floor is bad
  enough to threaten the one-week target, the fallback is an RTC-timed cold-boot cadence in which
  touch simply does not wake the device — a materially worse desk companion. #12 decides this;
  #6's job is to say the choice exists and what it costs.
- *No hardware wake-on-motion means "pick it up to attend it" is not implementable as designed.*
  The marketing "wake-up on lift" refers to re-powering a fully-off board via the PMS150G. If we
  want pickup-to-attend, the only supported route is polling the accelerometer during light-sleep
  RTC wakes, which is a latency/power trade, not a free interrupt. Recommend: don't promise
  pickup-to-attend; make touch the attend gesture and let motion feed BODY only.
- *Charger events are the only true deep-sleep wake we control.* Cheap, reliable, and worth
  spending: plugging in should always produce a visible, satisfying response.

---

## What remains uncertain

Flagged honestly, because several of these cannot be settled by any document.

**Needs a bench measurement, not a datasheet:**

1. **Touch latency, end to end.** No source gives it. The GT911's own scan cycle is 5–20 ms in
   Normal and ~40 ms in Green **[GT911-PG]** §6.1, and the vendor poll floor is 4 ms **[M5U]**, but
   the number that matters — finger-down to first e-ink pixel change, including the Green→Normal
   transition, the I²C round trip and the waveform — has to be measured. It couples directly to #9
   and #10.
2. **Light-sleep idle current with GT911 awake and armed on G48.** Neither Goodix nor M5Stack
   publishes a GT911 per-mode current figure that I could obtain: the datasheet M5Stack hosts
   **[GT911-DS]** is a scanned/image PDF whose body text is not extractable, and its accessible
   table of contents lists the operating modes but no consumption table. **[Store]**'s 9.28 µA and
   949.58 µA figures are both "main power off", i.e. neither describes the state we actually want
   ambient to be. This is the single biggest open number for #12.
3. **Green-mode wake reliability.** Whether a first touch on a GT911 sitting in Green mode reliably
   produces a clean INT low edge that light sleep catches, or whether the first touch is
   occasionally swallowed by the Green→Normal transition.
4. **Step-counter behaviour for a desk object.** **[BMI270]** §4.8.8 is explicit that the algorithm
   is "optimized for high accuracy in wrist use-case applications". A device sitting on a desk, in
   a bag, or carried in a hand is not a wrist. Whether the count is usable, needs the 25 SC
   parameters retuned, or should be replaced by *activity recognition* (still/walking/running,
   §4.8.5) as the actual signal, is a measurement question that #16 must answer before the
   step-walk mode is specified.
5. **Battery accuracy.** How well a 12-bit calibrated ADC through a 2:1 divider tracks state of
   charge across a 1800 mAh cell's discharge curve, and whether it is stable enough to drive the
   "pet visibly weakens as charge drops" behaviour without oscillating.

**Needs verification against the physical unit or a newer schematic:**

6. **Whether BMI270 `INT1` is genuinely unconnected on production hardware.** Schematic V1.0
   **[Sch]** shows no net on `INT1`/`INT2` and no `E_TRIG` net anywhere; the forum claim describes
   an `INT1 → PMS150G` path via `E_TRIG`. Both agree the ESP32-S3 cannot see it, so no design
   decision here changes either way — but if we ever want motion-triggered *cold boot*, the answer
   matters. Settle it by probing the part, or by asking M5Stack for a revision-current schematic.
7. **What the shipped GT911 configuration actually contains** — the idle-to-Green timeout
   (`0–15 s`), the Normal refresh rate, and whether the gesture switches at `0x8075`/`0x8076` are
   populated at all. Read back `0x8047`–`0x80FE` on the device.
8. **Whether the PMS150G exposes any state to the ESP32.** `GPIO0_SYS_LED` is shared between the
   PMS150G's `PA5/PRSTB` and the host's G0 through a link marked `0R(TBD)` **[Sch]**, and
   **[M5U]** drives G0 as a PWM LED output. If `PA5` ever drives that line, G0 — an RTC GPIO —
   could in principle carry a button-press signal into deep sleep. This is speculative and unproven;
   do not design on it, but it is the one remaining avenue by which the side button could become
   readable.

**Established only for M5Paper v1, explicitly NOT true here:** three physical buttons on
GPIO37/38/39; touch INT on an RTC-capable GPIO36 usable for ext0 deep-sleep wake; the ESP32 (not
S3) RTC GPIO set. All three appear in M5Paper tutorials and forum posts and would poison this
ticket if imported.
