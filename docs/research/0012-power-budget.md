# Power budget: does this game slate hit one week per charge?

Research for [issue #12](https://github.com/r4stered/papergotchi/issues/12). Depends on #9 (refresh
strategy, closed) and #11 (the ambient/session state machine, closed). Gates the battery target in
the map, and prices the levers for #11, #13 (idle animation) and #10 (the 5 fps spike).

> **#11 has since chosen, and the parametric model can be collapsed** (ADR-0009, ADR-0010).
> Board-off rest state; **one ~1 s animation run every 3 minutes** while awake and unattended, none
> during quiet hours or **Sleep**; a 90 s session timeout with light sleep between touches; bounded
> **attention windows** rather than permanently touch-armed ambient. Against §7.4's named
> configurations that sits between B and C, at roughly **22 days at nominal** — and §9.1.2's
> suggestion to escalate the wake cadence during a call was **declined**: the animation
> *de-escalates* instead, so a pet left calling is now cheaper than a content one rather than
> dearer. The failure shape this note warned about is closed by construction.

**Nothing here was measured on the board.** As with note 0009, no hardware was attached. What this
note delivers is the cost *model*, built from the ESP32-S3 datasheet, M5Stack's own published
board-level figures, the vendor board-support source read at a pinned commit, and note 0009's scan
counts — plus a sensitivity analysis over the free variables, and the short list of bench
measurements that would collapse the error bars. Every claim is tagged:

- **[S]** read directly from source, a datasheet, or a first-party specification
- **[D]** arithmetic derived from those constants — the arithmetic is shown
- **[U]** unverified, needs hardware
- **[2]** second-hand: a community measurement, corroborating but not first-party

Source was read at pinned commits, and every file:line citation is a permalink at those commits:

- **M5GFX** [`729297d`](https://github.com/m5stack/M5GFX/tree/729297d6e3d657ddc1ec5189bac2f2ea68828085) (v0.2.26)
- **M5Unified** [`4fb4447`](https://github.com/m5stack/M5Unified/tree/4fb444784c85791e0b0207701392b42be234b2e7)
- **ESP-IDF** [`0c1e6bd`](https://github.com/espressif/esp-idf/tree/0c1e6bd965302d25b1f3d49ba36425bcdaa55cb3) (`release/v6.0`, 2026-07-31)

Primary documents, in rough order of how much of this note rests on them:

- **M5Stack PaperS3 product documentation**, [docs.m5stack.com/en/core/PaperS3](https://docs.m5stack.com/en/core/PaperS3) — the three whole-board power figures, and the most useful numbers in the investigation
- **PaperS3 schematic V1.0**, [sch_papers3_V1.0.pdf](https://m5stack-doc.oss-cn-shenzhen.aliyuncs.com/517/sch_papers3_V1.0.pdf) — the power tree, read at netlist level
- **ESP32-S3 Series Datasheet v2.2** (2026-03-05), [documentation.espressif.com/esp32-s3_datasheet_en.pdf](https://documentation.espressif.com/esp32-s3_datasheet_en.pdf) — §5.6 Current Consumption
- **E Ink Holdings, ED047TC1 Technical Specification**, doc P-511-710(V:1) Rev 1.0, 2015-03-19, [PDF mirror](https://raw.githubusercontent.com/Xinyuan-LilyGO/LilyGo-EPD47/esp32s3/datasheet/ED047TC1.pdf) — §6-2 gives every panel rail current
- **TI TPS65185 datasheet**, SLVSAQ8G, [ti.com/lit/ds/symlink/tps65185.pdf](https://www.ti.com/lit/ds/symlink/tps65185.pdf) — the only source with EPD rail-generation efficiency curves at `V_IN` = 3.5 V
- **Goodix GT911 datasheet Rev.09** and **Programming Guide Rev.10** — per-mode current, and the sleep command
- Part datasheets: [SY8089AAAC](https://www.mouser.com/datasheet/2/306/SY8089AAAC-3223250.pdf), [ME6203](https://uploadcdn.oneyac.com/attachments/files/brand_pdf/microne/C87842_48D0F6FADAD1904D70E94145E51AAE7B.pdf), [MT9700](https://www.lcsc.com/datasheet/lcsc_datasheet_1809291208_XI-AN-Aerosemi-Tech-MT9700_C89855.pdf), [MT3608](https://www.olimex.com/Products/Breadboarding/BB-PWR-3608/resources/MT3608.pdf), [LGS4056H](https://m5stack-doc.oss-cn-shenzhen.aliyuncs.com/1130/LGS4056H.pdf), [BM8563](https://m5stack.oss-cn-shenzhen.aliyuncs.com/resource/docs/datasheet/core/BM8563_V1.1_cn.pdf), [XM25QH128C](https://www.xmcwh.com/uploads/801/XM25QH128C_Ver2.1.pdf), [PMS150G](https://www.padauk.com.tw/upload/doc/PMS15B,PMS150G%C2%A0datasheet_EN_V008_20230216.pdf), [BMI270](https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bmi270-ds000.pdf)
- Representative cell datasheets (**the actual cell is not published** — see §2.1): [EEMB LP505464 1800 mAh](https://eemb.oss-accelerate.aliyuncs.com//uploads/20230315/858b4689fd14fd5e3731f6f44fb35cf5.pdf), [LiPol LP103450](https://www.li-polymer-battery.com/wp-content/uploads/2021/03/LP103450-3.7V-1800mAh-Datasheet.pdf), [GMB 505068](https://www.powerstream.com/lip/GMB505068.pdf), plus [VARTA CellPac LITE Technical Handbook](https://www.varta-ag.com/fileadmin/varta_storage/publications/Technical_Handbook_CellPac_LITE_Rev._2.6.pdf) for discharge curves

---

## The eight findings that decide the budget

1. **The chip's deep-sleep current is irrelevant on this board.** ESP32-S3 deep sleep is **7 µA**
   **[S]**. The measured PaperS3 in ESP32 deep sleep is **~5.1 mA** **[2]** — 730× higher. Anyone
   who budgets from the Espressif datasheet alone will be wrong by nearly three orders of magnitude.
2. **That 5.1 mA is the GT911 touch controller, and it is a software omission.** M5GFX and
   M5Unified never issue the GT911's sleep command, so it sits on the always-on 3.3 V rail scanning
   at 3.3 mA (Green) or 8 mA (Normal) **[S]**. Sleeping it — `0x05` → register `0x8040` — drops it
   to 70–120 µA and takes the whole board to **~0.4 mA** **[D]**, a **~13× improvement over the
   number this budget was nearly built around**. The rest of the deep-sleep floor is a hard ~200 µA
   of resistors on the `MPWR_EN` net that firmware cannot touch (§3.3).
3. **The board's real low-power state is *off*, not asleep** — main power dropped via the PMS150G
   power MCU, with the BM8563 RTC alarm bringing it back as a **cold boot**. M5Stack publish
   **9.28 µA** for it **[S]**, independently measured at 9.6 µA **[2]**, and a part-by-part sum of
   the schematic gives **11 µA** **[D]** — three routes to the same number. That is **1.6 mAh per
   week, 0.1 % of the budget.** The floor is free.
4. **Therefore the currency is not milliamps, it is awake-seconds.** With the floor at ~zero and
   "main power on" at **154 mA** **[S]**, a week's usable charge buys roughly **9 hours of awake
   time — about 80 minutes a day.** Every design question reduces to "how many seconds a day is
   this board energised, and what is it driving while it is".
5. **The light-sleep-versus-board-off crossover lands at ~3 minutes, right inside the range #11 is
   choosing from.** Faster than that, staying in light sleep (with the GT911 slept, ~0.79 mA) beats
   paying a cold boot every wake; slower, board-off wins **[D]**. **#11's cadence decision and its
   rest-state decision are therefore the same decision.**
6. **Two defaults waste more energy than the entire display does.** `CONFIG_SPIRAM_MEMTEST` is on
   by default and costs *"approximately 1 second per 4 MB"* — **~2 s on this 8 MB part, on every
   wake** **[S]**. And `Power_Class::_powerOff()` pulses `PWROFF_PLUSE` (G44) five times at 50 ms
   low / 50 ms high, so **every board-off wake spends 500 ms at full power just switching itself
   off** **[S]**. Together they are worth **~9 days**, they cost nothing to fix, and neither has
   anything to do with the pet.
7. **This reverses #13's cost-neutrality finding.** Note 0009 §7.4 counted scans and concluded that
   slow-and-constant (one frame every 10 s) is **0.6×** the cost of smooth-and-rare (2 s of 5 fps
   once a minute) — "nearly cost-neutral, choose on aesthetics". In *power* it is **1.0–2.0× more
   expensive** **[D]**, because in the board-off architecture every lone frame needs its own cold
   boot. The ranking flips, and the swing against the scan-counting prediction is **1.7–3.4×**.
8. **WiFi backup is not a lever.** At note 0004's ~5 uploads/day a backup costs **0.7–14 mAh/week
   depending entirely on four `sdkconfig` defaults** (a 1–2 s DHCP ARP check and a *random 0–5000 ms
   SNTP startup delay* are the two biggest terms, both first-party-documented, both switchable)
   **[S]**. Even the worst case is under 1 % of the budget. Note 0004 called upload cadence "the
   cheapest lever on #12"; it is not a lever at all.

**The verdict is in §8.** Short version: one week is real, with room to spare, *provided* the rest
state is board-off (or light sleep with the touch controller asleep) and the idle animation runs at
a cadence measured in minutes rather than seconds. At 1 animation run per minute the model misses by
a wide margin. At 1 per 3 minutes it clears in every column.

---

## 1. Units, and the conversion that everyone gets wrong

The classic error in these budgets is mixing a 3.3 V rail current with a battery mAh figure. Both
directions of the mistake are ~20 %, and they compound. So, explicitly:

**Convention used throughout this note: every current and every mAh is *battery-referred* — charge
drawn from the pack terminals.** Where a source gives a rail-referred figure it is converted, and
the conversion is shown.

| Quantity | Value |
|---|---|
| Pack | 1S Li-ion/LiPo, 3.0–4.2 V, **3.7 V nominal** **[S]** |
| Nameplate charge | **1800 mAh** **[S]** |
| Nameplate energy **[D]** | `1800 mAh × 3.7 V` = **6.66 Wh** = 6660 mWh = **23 976 J** |
| One week | 168 h |
| Average current allowed at nameplate **[D]** | `1800 / 168` = **10.71 mA** |

### 1.1 Rail → battery

For a load on the 3.3 V system rail drawing `I_rail`:

- **Through a buck converter** of efficiency η: power is conserved, so
  **[D]** `I_batt = I_rail × V_rail / (V_batt × η)`.
  At `V_rail = 3.3`, `V_batt = 3.7`, `η = 0.85`: factor = `3.3 / (3.7 × 0.85)` = **1.049**.
  Battery current is *slightly higher* than rail current — the voltage step-down (0.892) is more
  than cancelled by the efficiency loss (1/0.85 = 1.176).
- **Through an LDO**: current is passed straight through, so **[D]** `I_batt = I_rail` exactly,
  factor **1.000**. The penalty is in energy, not charge: `(3.7 − 3.3)/3.7` = 11 % wasted at
  nominal, 21 % at a full 4.2 V.

> **The trap:** multiplying a 3.3 V rail current by `3.3/3.7 = 0.892` to get battery current. That
> is what you would do for a *lossless* buck and it is never right. For a real buck the factor is
> just above 1; for an LDO it is exactly 1. Assume 0.892 and you under-budget by 15–20 %.

The near-unity factor is a convenience, not a licence: at **light load** a buck's efficiency
collapses. A converter drawing tens of microamps in continuous-PWM mode can be under 30 % efficient,
pushing the factor to 3–4×. This is exactly the regime the sleep floor lives in, which is another
reason the floor must be *measured* rather than summed from datasheets. **[U]**

### 1.2 The one figure that dodges all of this

M5Stack quote their three whole-board figures at **DC 4.2 V** at the *battery* **[S]**, so they are
already pack-referred and need no conversion. They are the anchor for everything below, and it is
worth being explicit that they are the most useful numbers in this entire investigation.

| M5Stack published state | Current @ 4.2 V | mAh/week **[D]** |
|---|---|---|
| "Low power mode (main power off, gyroscope in low power mode)" | **9.28 µA** | **1.6** |
| "Standby mode (main power off, gyroscope on)" | **949.58 µA** | **159.5** |
| "Operating mode (main power on)" | **154.02 mA** | 25 875 (i.e. 18× the battery) |

— [docs.m5stack.com/en/core/PaperS3](https://docs.m5stack.com/en/core/PaperS3), repeated verbatim
on the [store listing](https://shop.m5stack.com/products/m5papers3-esp32s3-development-kit).

**These three numbers decompose cleanly, which is why they are trustworthy.** The delta between the
first two is **[D]** `949.58 − 9.28 = 940.30 µA`, and the BMI270 datasheet gives 685 µA for normal
IMU mode and 970 µA for performance mode at max ODR (note 0006 §"The IMU"). So "gyroscope on"
accounts for essentially the whole difference. Working backwards, with the BMI270 in accel-only low
power (4–10 µA), **the entire rest of the board in its off state draws somewhere between 0 and
5 µA** **[D]**. Put the BMI270 in suspend (3.5 µA) and the floor should fall to ~4 µA. There is
nothing left to optimise there.

---

## 2. What a week's charge actually buys

### 2.1 Usable capacity

**No datasheet exists for the actual cell.** M5Stack publish one line — *"3.7V@1800mAh lithium
battery @ charging chip: LGS4056H"* — and no manufacturer, part number, discharge curve or
self-discharge spec, on the docs page, the shop page or in the schematic BOM **[S]**. The battery
header is a 2-pin HY1.25-2P with no NTC pin, so the docs cannot even tell you whether the pack
carries a protection circuit. Everything below therefore comes from **representative 1800 mAh-class
pouch-cell datasheets**, named where used, and the whole section should be read as a modelled band
rather than a measurement.

Nameplate is not spendable. Four deductions apply, and they are multiplicative:

| Deduction | Effect | Basis |
|---|---|---|
| **Grading** — the guaranteed capacity is below the nominal | ×0.944 worst case | EEMB LP505464 grades *minimum* 1700 mAh against *nominal* 1800 **[S]** |
| **Cutoff** — charge below the firmware's cutoff voltage | ×0.98 – 0.99 at 3.5 / 3.4 V | see §2.2 |
| **Ageing** — cycle + calendar | ×0.80 – 1.00 | *"300 cycles ~ 80 % of capacity"* (Jauch LP103450JH); *"≥500 cycles(0.5C5A) / ≥800 cycles(0.2C5A)"* to 80 % (EEMB) **[S]** |
| **Self-discharge over the 168 h window** | ×0.953 – 0.993 | 3 %/month typical → 0.7 %/week; the IEC 28-day *spec floor* is far worse — EEMB requires only ≥90 % after 28 days, PKCELL only ≥80 % **[S]** |

Which gives:

| Case | Composition | Usable | Average current allowed over 168 h **[D]** |
|---|---|---|---|
| Nameplate | — | 1800 mAh | 10.71 mA |
| Optimistic (98 %) | 1800 × 0.99 × 1.00 × 0.993 | **1770 mAh** | 10.54 mA |
| Sourced nominal (88 %) | 1800 × 0.98 × 0.92 × 0.977 | **1590 mAh** | 9.46 mA |
| **Design budget used throughout this note** | *deliberate margin below the above* | **1440 mAh** | **8.57 mA** |
| Conservative (65 %, end-of-life) | 1700 × 0.90 × 0.80 × 0.953 | **1170 mAh** | 6.96 mA |

> **Every table in this note is computed against 1440 mAh.** That is ~10 % below the sourced nominal
> — a deliberate margin, because the cell is unidentified. To read the tables against the other
> cases, scale every runtime: **× 1.10** for the sourced 1590 mAh nominal, **× 0.81** for the
> 1170 mAh end-of-life case, **× 1.23** for a fresh cell at 1770 mAh. **The verdict in §8 is
> reported against all three.**

Also worth recording for #5's crash-safety work and for the map's "flat battery" behaviour: at one
charge per week, 300 cycles takes **5.8 years** **[D]**, so cycle ageing is not the binding
degradation mechanism — calendar ageing is.

### 2.2 Where the bottom of the curve goes — and the good news is the regulator is not the problem

The 3.3 V system rail comes from **U3, an `SY8089AAAC` synchronous buck**, not from the
`ME6203A33M3G` LDO. Confirmed from the schematic feedback network rather than the label: `R18` =
100 K top, `R19` = 22 K bottom, and Silergy's `VREF = 0.600 V` gives **[D]**
`0.600 × (1 + 100/22) = 3.327 V` **[S]**. The `ME6203A33M3G` (U10) makes a *separate*, permanently
enabled 3.3 V rail (`IMU_VDD`) that feeds the BMI270 and nothing else.

That matters a lot, because the buck's datasheet specifies **"100 % dropout operation"** and
**maximum duty cycle 100 %**, with `V_IN` 2.7–5.5 V and UVLO ≤ 2.5 V
([SY8089AAAC datasheet](https://www.mouser.com/datasheet/2/306/SY8089AAAC-3223250.pdf)) **[S]**.
Below dropout it simply passes through at 100 % duty and `SOC_VDD` tracks the cell minus the IR drop
of the series chain — Q3 CJ3401 + `MT9700` (80 mΩ) + the buck's 110 mΩ PFET + inductor DCR,
**[D]** ≈ 0.35 Ω. So the rail holds 3.327 V down to **[D]** ~3.33 V at milliamp loads and ~3.43 V at
300 mA.

> **Had M5Stack put the `ME6203A33M3G` on the main rail this section would read very differently:**
> its dropout is 0.22 V at 10 mA and **1.1 V at 50 mA** **[S]**, which would have stranded 5–7 % of
> the cell at light load and been unusable during a WiFi burst. They did not. The buck was the right
> part and the usable-capacity question is settled by cell grading and ageing, not by the power path.

So the floor is set by the load, not the regulator, and there are two candidates:

- **The ESP32-S3's recommended minimum input voltage is 3.0 V** (Table 5-2, "VDDA, VDD3P3
  recommended input voltage min 3.0 V") **[S]**. Below that Espressif make no promises.
- **The brownout detector will not save you first.** `CONFIG_ESP_BROWNOUT_DET` defaults `y` and
  `ESP_BROWNOUT_DET_LVL_SEL` defaults to `_SEL_7` on `esp32s3` — **the lowest of the seven levels,
  ≈ 2.44 V** ([`Kconfig.power`](https://github.com/espressif/esp-idf/blob/master/components/esp_hw_support/power_supply/port/esp32s3/Kconfig.power))
  **[S]**, with Espressif's own in-source caveat that *"the voltage levels here are estimates"*. It
  fires **below** the regulator's useful range, so it is a last-resort backstop, not the cutoff.

Transient sag is the real risk. An 802.11b TX burst peaks at **340 mA** (Table 5-7) **[S]**, which
across the 0.35 Ω series chain is **[D]** ~120 mV, plus 14 mV across a bare 40 mΩ cell or ~68 mV
across a protected 200 mΩ pack.

**Recommended firmware cutoff: 3.4 V.** **[D]** It costs ~1 % of capacity — at 0.2C a pouch cell
delivers ~99 % of its charge above 3.4 V and ~98 % above 3.5 V (VARTA *CellPac LITE Technical
Handbook* discharge profiles) **[S]**, and we discharge three orders of magnitude slower than that —
while keeping `SOC_VDD` above 3.0 V through a 340 mA burst even on an aged pack.

The map's flat-battery behaviour ("the pet visibly weakens as charge drops; the device buzzes,
takes a final cloud snapshot, and shuts down cleanly") is what makes this a design parameter rather
than an accident: the device chooses its own cutoff, so the last percent is deliberately reserved
for the death ritual.

One more thing the schematic settles: **the LGS4056H is not a power-path charger** — its pinout is
`TEMP`/`PROG`/`GND`/`VCC`/`BAT`/`DONE`/`CHRG`/`CE` with **no `SYS` pin** **[S]**. The system load
sits directly across the cell, so system current subtracts from charge current, and a device that is
busy charges more slowly. Its battery-side leakage with USB absent is ≤ 1 µA **[S]**.

### 2.3 The awake-second currency

This is the reframing that makes the rest of the note tractable. With the sleep floor at
**1.6 mAh/week** — 0.1 % of the budget — essentially *all* of the charge is spent while the board is
energised. So price everything in awake-seconds:

**[D]** `1 second at I mA = I × 1000 / 3600 µAh`

| "Main power on" current | µAh per awake-second | Awake-seconds in 1440 mAh | Per week | Per day |
|---|---|---|---|---|
| 80 mA (CPU busy, RF off, panel idle) | 22.2 | 64 800 s | 18.0 h | 154 min |
| **120 mA (nominal working figure)** | **33.3** | **43 200 s** | **12.0 h** | **103 min** |
| 154 mA (M5Stack's "operating mode") **[S]** | 42.8 | 33 662 s | 9.35 h | **80 min** |

> **One week per charge = roughly 80–150 minutes a day of energised board.** That is the whole
> budget, stated in the only unit that matters. Everything below is an argument about how to spend
> 80 minutes.

The 80 mA end of the band is the ESP32-S3 datasheet's own Modem-sleep figure for 240 MHz, dual core
running 32-bit data access instructions with peripheral clocks enabled — **81.3 mA** (Table 5-9)
**[S]** — plus a little for I²C and the GT911. The 154 mA end is M5Stack's number under unstated
conditions, and it is high enough to suggest the radio or the panel rails are up in their test. Both
are carried because the truth is somewhere between and **it has not been measured** **[U]**.

---

## 3. The floor: which rest state, and what it costs

This is the section that decides the ticket. #6 already established the wake-source facts; what
follows is their price.

### 3.1 The chip is not the problem

ESP32-S3 Series Datasheet v2.2, Table 5-10, "Current Consumption in Low-Power Modes", measured at a
3.3 V supply and 25 °C ambient **[S]**:

| Work mode | Description | Typ (µA) |
|---|---|---|
| Light-sleep | VDD_SPI and Wi-Fi powered down, all GPIOs high-impedance | **240** |
| Deep-sleep | ULP-FSM powered on | 170 |
| Deep-sleep | ULP-RISC-V powered on | 190 |
| Deep-sleep | ULP sensor-monitored pattern (touch at 1 % duty) | 18 |
| Deep-sleep | RTC memory **and** RTC peripherals powered up | **8** |
| Deep-sleep | RTC memory powered up, RTC peripherals powered down | **7** |
| Power off | `CHIP_PU` low, chip shut down | 1 |

Two footnotes matter and both cost us:

- Light-sleep footnote 1: *"For chips embedded with PSRAM, please add corresponding PSRAM
  consumption values, e.g., **140 µA for 8 MB 8-line PSRAM (3.3 V)**"* **[S]**. The PaperS3 is an
  ESP32-S3**R8** with octal PSRAM (note 0002), so our light-sleep chip figure is **[D]**
  `240 + 140 = 380 µA`.
- §5.6.2 preamble: *"The measurements below are applicable to ESP32-S3 and ESP32-S3FH8. Since
  ESP32-S3R2, … **ESP32-S3R8**, … are embedded with PSRAM, their current consumption **might be
  higher**."* **[S]** So even the 7 µA deep-sleep figure is not strictly ours.

Three things the ticket asked about turn out not to exist on this chip, which is worth recording so
nobody goes looking:

- **There is no hibernation mode on ESP32-S3.** The word appears nowhere in the datasheet. The floor
  below deep sleep is "Power off, `CHIP_PU` low, 1 µA". Do not carry an ESP32-classic hibernation
  figure across.
- **RTC slow and fast memory cannot be powered down.** `SOC_PM_SUPPORT_RTC_SLOW_MEM_PD` and
  `SOC_PM_SUPPORT_RTC_FAST_MEM_PD` are undefined for `esp32s3`, so `ESP_PD_DOMAIN_RTC_SLOW_MEM` and
  `ESP_PD_DOMAIN_RTC_FAST_MEM` do not exist **[S]**. Both are always retained, in both sleep modes.
  There is no retention/current trade to make.
- **The only power-domain choice deep sleep actually offers is RTC peripherals**, and it is the 1 µA
  difference between the last two rows. `ext0` and touch wake require them (8 µA); `ext1` does not
  (*"it will work even if RTC peripherals are shut down during sleep"*) and keeps the 7 µA row
  **[S]**. ESP-IDF's default already powers down whatever the enabled wake sources do not need.

**And none of it matters**, because:

### 3.2 The board is the problem

Espressif say so themselves, in the guide on how to measure this:

> *"For ESP32-S3, **using a development board directly to measure current consumption of the
> corresponding module is not recommended, as some circuits still consume power on the board** even
> when you flash the chip with the deep_sleep example."*
> — [current-consumption-measurement-modules](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/current-consumption-measurement-modules.html) **[S]**

On this board it is not "some circuits". It is everything:

| Rest state | Battery current | mAh/week **[D]** | % of 1440 mAh | Tag |
|---|---|---|---|---|
| Board off, BMI270 low power | **9.28 µA** | **1.6** | **0.1 %** | **[S]** M5Stack |
| Board off, measured | 9.6 µA | 1.6 | 0.1 % | **[2]** |
| Board off, BMI270 gyro on | 949.58 µA | 159.5 | 11.1 % | **[S]** M5Stack |
| Board on, ESP32 **deep sleep**, as the vendor libraries leave it | **~5.1 mA** | **857** | **59.5 %** | **[2]** measured |

The 5.1 mA figure is second-hand — a well-known M5Stack community contributor reporting bench
measurements on the
[Paper S3 questions](https://community.m5stack.com/topic/7108/paper-s3-questions) thread: *"M5PaperS3
fully powered off … at the battery I measure about 9.6 uA (which is close to the advertised
9.28 uA)"* and *"in ESP32 deep sleep I get about 5.1 mA (which is about 550 times higher)"*. A second
user on the same forum independently reports *"the device is drawing the battery down about 5-7 % per
day, which is roughly 5 ma"*
([Slow battery performance with M5PaperS3](https://community.m5stack.com/topic/8087/slow-battery-performance-with-m5papers3),
where the same contributor gives *"In ESP32 deep sleep the system consumes about 5 - 6 mA according
to measurements I did a while ago"*).

It is not first-party and it must be re-measured (§10, measurement 1). But **it agrees with
M5Stack's own published figures at the other end of the range**, it is corroborated by a second
independent report, and it is the only number available for the state in question. Treat it as a
working figure with a ±50 % band.

### 3.3 Where 5.1 mA comes from — and it is mostly one fixable thing

Traced against the schematic and the part datasheets. **[D]** `5.1 mA − 7 µA (chip) = 5.09 mA` of
board, and it decomposes:

| Contributor | Draw | Fixable? | Source |
|---|---|---|---|
| **GT911 touch controller, never commanded to sleep** | **3.3 mA (Green) – 8 mA (Normal)** | **Yes — I²C command** | GT911 DS Rev.09 **[S]** |
| **`MPWR_EN` resistor network** — `R28` 33 K to GND plus the `R24`/`R25` 22 K+22 K battery-sense divider, both hung off `MPWR_EN` (≈ V_BAT) rather than off a switched rail | **196 µA @3.7 V, 223 µA @4.2 V** | **No — hardware** | schematic **[S]** |
| `SY8089AAAC` buck quiescent | 55 µA typ (spec measured *not switching*) | No | Silergy DS **[S]** |
| `MT9700` load switch (`U7`, main rail) on | 15 µA typ / 25 µA max | No | Aerosemi DS **[S]** |
| `ME6203A33M3G` LDO feeding `IMU_VDD` — **no enable pin** | 3 µA typ | No | Microne DS **[S]** |
| BMI270 suspend / step counting | 3.5 µA / ~13 µA | Choice | Bosch DS **[S]** |
| PMS150G power MCU (`stopexe`) | 3 µA | No | Padauk DS **[S]** |
| BM8563 timekeeping | 0.25–0.65 µA @3 V | No | BM8563 DS **[S]** |
| LGS4056H charger, VCC absent (anti-backflow) | < 1 µA | No | LGS4056H DS **[S]** |
| `MT9700` (`U8`, boost enable) off + leakage | 1.5 µA typ / 11 µA max | Already off | **[S]** |
| **Flash `XM25QH128C`** — `VCC` is the ESP32's own `VDD_SPI` pin, which `esp_deep_sleep_start()` powers down | **0** | Already free | schematic **[S]** |
| **EPD rails, panel `VDD`, VCOM divider, bleeders** | **0** | Already free | see §5.6 |
| **Total with GT911 asleep** | **≈ 360–490 µA** | | **[D]** |
| **Total as the vendor libraries leave it** | **≈ 3.7 – 8.6 mA** | | **[D]** |

> **The community's 5.1 mA is the GT911, and nothing else.** M5GFX and M5Unified never issue the
> GT911's sleep command, so it sits on the always-on `SOC_VDD` rail scanning at 3.3 mA (Green) or
> 8 mA (Normal). Sleeping it — write `0x05` to register `0x8040`, having first driven `INT` low —
> drops it to **70–120 µA** and takes the whole board's deep-sleep draw to **~0.4 mA**. That is
> **~15× better than the number this note was built around**, and it reopens an option §3.4 would
> otherwise have closed.

Two things are *not* fixable and are worth knowing about:

- **The `MPWR_EN` network is a hard ~200 µA floor whenever the main rail is enabled.** It is
  resistors on the schematic, not a firmware setting. It goes to zero only in hard power-off. **[S]**
- **The `SY8089AAAC`'s real efficiency at a ~400 µA load is unknown.** Silergy's datasheet mentions
  only "proprietary PWM control" — no PFM, pulse-skip or power-save mode is documented, and the
  55 µA Iq figure is specified with the feedback node forced so the converter is *not switching*. A
  fixed-frequency 1 MHz buck at hundreds of microamps can easily be 30–50 % efficient, which would
  multiply everything on `SOC_VDD` by 2–3×. **[U]** This is the largest unquantified term left in
  the note and it is measurement 1 in §10.

### 3.4 The consequence, stated carefully

Three rest states are actually available, not two, and the third only exists because of the GT911
finding above:

| Rest state | Battery current | mAh/week **[D]** | % of 1440 | RAM/PSRAM kept? | Touch wakes? |
|---|---|---|---|---|---|
| **Board off** (PMS150G, BM8563 alarm) | **9.3–11 µA** **[S]/[D]** | **1.6–1.8** | **0.1 %** | No — cold boot | No |
| **Deep sleep, GT911 slept, EPD cut** | ~0.36–0.49 mA **[D]** | 60–82 | 4–6 % | No — bootloader runs | No |
| **Light sleep, GT911 slept** | ~0.73–0.86 mA **[D]** | 123–145 | 9–10 % | **Yes** | No |
| **Light sleep, GT911 in Green** | ~4.0–4.2 mA **[D]** | 672–706 | 47–49 % | Yes | **Yes** |
| *Deep sleep as the vendor libraries leave it* | *~5.1 mA* **[2]** | *857* | *59.5 %* | No | No |

> **ESP32 deep sleep *as shipped* is not a viable ambient rest state.** At 5.1 mA it consumes
> **857 mAh — 60 % of a nominal usable budget — doing nothing**, and running the device to flat while
> permanently asleep gives **[D]** `1800 / 5.1 mA = 353 h = 14.7 days`: it clears one week only in
> the sense that a brick clears one week. **But that is a software omission, not a property of the
> board**, and §4.5 shows the corrected figures make light sleep genuinely competitive at fast
> cadences.

**The decomposition validates itself against M5Stack's published numbers**, which is why it can be
trusted despite none of it being measured. Summing the hard-power-off column — PMS150G 3 µA +
BM8563 0.4 µA + ME6203 3 µA + BMI270 suspend 3.5 µA + LGS4056H < 1 µA + `U7` off 1.5 µA — gives
**[D]** **≈ 11 µA**, against M5Stack's published **9.28 µA** and an independent measurement of
**9.6 µA**. Swap the BMI270 from suspend to accel+gyro low power (+420–940 µA) and you land on
M5Stack's **949.58 µA** "standby, gyroscope on". **Two published figures, three datasheets and a
netlist all agree.** That is as much confidence as this note is going to get without a bench.

The supported alternative is M5Unified's `Power_Class::timerSleep()`, which is not a sleep at all:

```c
void Power_Class::timerSleep( int seconds ) {
  M5.Rtc.disableIRQ();  M5.Rtc.clearIRQ();  M5.Rtc.setAlarmIRQ(seconds);
  esp_sleep_enable_timer_wakeup(seconds * 1000000ULL);
  _timerSleep();                     // → _powerOff(true)
}
```
— [`Power_Class.cpp:1373-1382`](https://github.com/m5stack/M5Unified/blob/4fb444784c85791e0b0207701392b42be234b2e7/src/utility/Power_Class.cpp#L1373-L1382)

`_powerOff()` pulses the power-hold pin and then, if the board somehow survives, deep-sleeps as a
fallback ([`:1154-1212`](https://github.com/m5stack/M5Unified/blob/4fb444784c85791e0b0207701392b42be234b2e7/src/utility/Power_Class.cpp#L1154-L1212)).
M5Stack's own wake-up documentation confirms the semantics: *"the device will restart the entire
program upon waking up"*
([docs.m5stack.com/en/arduino/m5papers3/wakeup](https://docs.m5stack.com/en/arduino/m5papers3/wakeup))
**[S]**.

**So ambient is a cold-boot cycle.** The BM8563 alarm re-energises the board through the PMS150G;
the ESP32-S3 comes up from power-on reset with no RAM, no PSRAM and no peripheral state. That is the
architecture the power budget has to be built on, and it is what makes boot time a first-class cost
(§4).

### 3.5 What the RTC can actually schedule

The BM8563 is a PCF8563-compatible part and M5Unified drives its **countdown timer**, not its clock
alarm, for second-resolution waits:

```c
std::size_t div = 1;  std::uint8_t type_value = 0x82;      // 1 Hz source
if (afterSeconds < 270) { if (afterSeconds > 255) afterSeconds = 255; }
else { div = 60; afterSeconds = (afterSeconds + 30) / div; type_value = 0x83; }  // 1/60 Hz source
writeRegister8(0x0E, type_value);  writeRegister8(0x0F, afterSeconds);
```
— [`PCF8563_Class.cpp:98-124`](https://github.com/m5stack/M5Unified/blob/4fb444784c85791e0b0207701392b42be234b2e7/src/utility/rtc/PCF8563_Class.cpp#L98-L124)

**[D]** So the schedulable range is **1 s to 255 s in 1 s steps, and 1 min to 255 min (4 h 15 m) in
1 min steps**. No cadence #11 might reasonably want is out of reach, and the 255-minute ceiling is
the only hard limit — a sleeping pet that wants an eight-hour night needs two hops. That ceiling is
independently noted by the same community contributor (*"setAlarmIRQ can only handle up to 255
minutes"*) **[2]**, and it follows directly from the register width **[S]**.

### 3.6 The price of the free floor: touch is dead, but motion is not

Mostly from #6, with one correction that matters:

- `TP_INT` is on **G48**, outside the ESP32-S3's RTC GPIO range of 0–21 (Table 2-6) **[S]**, so
  **touch cannot wake the chip from deep sleep** — and with the board off there is no chip to wake.
- The single physical button belongs to the PMS150G and reaches no ESP32 GPIO.
- The only *ESP32-visible* deep-sleep wakes a player can cause are `CHG_STAT` (G4) and `USB_DET`
  (G5) — plugging in a charger.

> **Correction to note 0006: the BMI270's `INT1` *is* wired.** Note 0006 read schematic V1.0 as
> showing no net on either interrupt pin. A closer trace finds `INT1` (pin 4) going to the gate of
> **`Q4` 2N7002W** (with `R47` 33 K gate pull-down), whose drain pulls the **`E_TRG`** net low —
> and `E_TRG` is the gate of `Q1A` in the PMS150G power latch **[S]**. `INT2` (pin 9) is genuinely
> a bare stub. This is exactly the `E_TRG` path a community contributor described and note 0006
> recorded as an unverified second-hand claim; **it is real.**
>
> **What changes and what does not.** The decision-relevant conclusion of #6 stands unaltered: the
> IMU still cannot interrupt the ESP32-S3 in any sleep state, so pickup-to-attend is not a free
> interrupt. But *motion can re-energise a fully powered-off board* through the PMS150G, as a **cold
> boot** — which, in the board-off architecture this note recommends, is *exactly the same event as
> an RTC-timed ambient wake.* **So "pick it up and it wakes" is implementable after all**, at the
> cost of one extra wake per pickup (~41 µAh, §4.4). At even 50 spurious pickups a day that is
> **14 mAh/week, 1 %** **[D]**. #11 should treat this as available, and #6's uncertainty 6 as
> resolved in the affirmative.

So in board-off ambient the pet cannot be woken by *touching* it, but it can be woken by being
*picked up* — which is arguably the better gesture anyway, and it is the one the store page's
"wake-up on lift" was always describing.

---

## 4. Cost per ambient wake

One ambient wake in the board-off architecture is: power-on reset → bootloader → app init → read the
clock and the step counter → `step()` → optionally repaint → pulse the board off. Four of those five
are pure overhead.

### 4.1 Boot, component by component **[D]/[U]**

| Component | Estimate | Basis |
|---|---|---|
| Hardware init before the CPU starts | ~280 µs | **[S]** Espressif: *"For most ESP chips, the initialization time is about 280 us"* |
| **ROM UART logging** | **~6.1 ms** | **[S]** *"For most ESP chips, this ROM printing takes about 6100 us"* — removable via `esp_deep_sleep_disable_rom_logging()` or an eFuse |
| PMS150G re-energises rails; power-on reset settling | 10–50 ms | **[U]** |
| **App image read + SHA-256 verify from flash** | **50–250 ms** | Espressif: *"the entire app binary is read from flash and verified which takes up a **significant portion of the boot time**"*; *"how much time this saves depends on the binary size and the flash settings"* **[S]** |
| Second-stage bootloader UART logging at 115200 | 0–80 ms | **[D]** ~15 lines × ~60 chars × 10 bits ÷ 115200 |
| **Octal PSRAM init + `SPIRAM_MEMTEST` (default `y`)** | **~2000 ms** | **[S]** *"approximately **1 second per 4 MB** of memory tested"* — and this part has **8 MB** |
| Heap poisoning, if comprehensive | +300 ms/4 MiB | **[S]** same page |
| RTC slow-clock calibration | 0–30 ms | **[S]** *"possible to save a small amount of time during boot by disabling RTC slow clock calibration… set `CONFIG_RTC_CLK_CAL_CYCLES` to 0"* |
| M5GFX `Panel_EPD` init: 1.24 MiB PSRAM alloc + zero, 43.5 KB LUT expansion | 30–80 ms | **[D]** from note 0009 §1.8 sizes |
| NVS mount | ~40 ms | **[S]** *"approximately 0.5 seconds per 1000 keys"*, ~80 keys (note 0005) |
| **BMI270 `begin()`: soft reset + 8192-byte config blob over I²C @400 kHz** | **~185 ms** | **[D]** see §4.2 |
| *(radio wakes only)* full PHY calibration | *+100 ms* | **[S]** *"full calibration takes about 100 ms more than partial calibration"* |
| **Total, ESP-IDF stock defaults** | **~2.3 – 2.7 s** | |
| **Total, tuned** | **~0.15 – 0.35 s** | |

> **`CONFIG_SPIRAM_MEMTEST` is the single biggest boot-time item and its default is wrong for us by
> an order of magnitude.** Espressif's own speed guide: *"When external memory is used
> (`CONFIG_SPIRAM` enabled), enabling memory test on the external memory (`CONFIG_SPIRAM_MEMTEST`)
> can have a **large impact on startup time (approximately 1 second per 4 MB of memory tested)**"*
> ([speed guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/performance/speed.html))
> **[S]**, and it defaults to `y`
> ([`Kconfig.spiram.common:89-95`](https://github.com/espressif/esp-idf/blob/release/v6.0/components/esp_psram/Kconfig.spiram.common#L89-L95))
> **[S]**. On an 8 MB R8 that is **~2 seconds of full-power board on every single ambient wake** —
> more than the boot, the refresh and the shutdown put together. **`CONFIG_SPIRAM_MEMTEST=n` is
> worth ~5.6 days on its own** (§8.3). The Kconfig help text's own description of the saving,
> *"slightly faster startup"*, is a considerable understatement.

**The rest of the levers, all one-line `sdkconfig` changes:**

- **`CONFIG_BOOTLOADER_SKIP_VALIDATE_ON_POWER_ON=y`.** Note this is *not* the option everyone
  reaches for. Because our cheap rest state is a **power-on reset**, the usual
  `CONFIG_BOOTLOADER_SKIP_VALIDATE_IN_DEEP_SLEEP` does nothing for us — Espressif's speed guide
  lists the two separately and scopes the first to deep-sleep wake **[S]**. The power-on variant
  carries a real caveat its help text states plainly: *"it's not possible for the bootloader to
  detect if an app image is corrupted in the flash, therefore it's not possible to safely fall back
  to a different app partition"*
  ([`Kconfig.projbuild:347-361`](https://github.com/espressif/esp-idf/blob/0c1e6bd965302d25b1f3d49ba36425bcdaa55cb3/components/bootloader/Kconfig.projbuild#L347-L361))
  **[S]**. On a device that boots ~300 times a day for two weeks, that trade is worth taking; record
  it as a deliberate decision, not a tuning knob.
- **`CONFIG_BOOTLOADER_LOG_LEVEL_NONE=y` and `CONFIG_LOG_DEFAULT_LEVEL_NONE`** — Espressif: these
  have *"a large impact on startup time"* **[S]**.
- **`CONFIG_BOOT_ROM_LOG_ALWAYS_OFF`** for the ~6.1 ms of ROM logging (burns an eFuse bit —
  irreversible; consider `esp_deep_sleep_disable_rom_logging()` instead).
- **`CONFIG_RTC_CLK_CAL_CYCLES=0`.**
- **Do not call `M5.Imu.begin()` on every wake** (§4.2).

**The `t_boot` band used throughout this note is 0.35 / 0.70 / 1.50 s, which assumes a *tuned*
build.** If `CONFIG_SPIRAM_MEMTEST` is left at its default the pessimistic case is 2.5 s, and
scenario B falls from 24.4 to **18.7 days** — still clearing one week, but throwing away a quarter
of the budget on a memory test that runs 300 times a day and has never once found anything.

**One lever that does *not* apply, and it is the one everybody reaches for first.**
`CONFIG_ESP_PHY_CALIBRATION_AND_DATA_STORAGE` (default on) states *"PHY calibration will be skipped
on deep sleep wakeup"* **[S]** — but our rest state is a **power-on reset**, not a deep-sleep wake,
so we pay the full ~100 ms calibration on every wake that brings up the radio. Same trap as
`BOOTLOADER_SKIP_VALIDATE_IN_DEEP_SLEEP` above: **on this board, "deep sleep wake" optimisations are
the wrong family of optimisations.** It only affects backup wakes, so it is worth ~0.1 s × 5/day,
not a headline — but it is the kind of thing that quietly invalidates a copied `sdkconfig`.

### 4.2 The BMI270 config blob is a per-wake tax, and it also destroys the step count

`BMI270_Class::begin()` unconditionally soft-resets the part and re-uploads the whole configuration
image:

```c
writeRegister8(CMD_REG_ADDR, SOFT_RESET_CMD);        // software reset.
...
bool res = _upload_file(bmi270_config_file, 0, sizeof(bmi270_config_file));
writeRegister8(INIT_CTRL_ADDR, 0x01);
```
— [`BMI270_Class.cpp:47-58`](https://github.com/m5stack/M5Unified/blob/4fb444784c85791e0b0207701392b42be234b2e7/src/utility/imu/BMI270_Class.cpp#L47-L58)

`bmi270_config_file` is **exactly 8192 bytes** (counted:
[`BMI270_config.inl`](https://github.com/m5stack/M5Unified/blob/4fb444784c85791e0b0207701392b42be234b2e7/src/utility/imu/BMI270_config.inl))
and `_upload_file` pushes it in a single `writeRegister` burst
([`:19-33`](https://github.com/m5stack/M5Unified/blob/4fb444784c85791e0b0207701392b42be234b2e7/src/utility/imu/BMI270_Class.cpp#L19-L33)).
The shared I²C bus runs at 400 kHz (note 0006).

**[D]** `8192 bytes × 9 bit-times ÷ 400 kHz = 184.3 ms`, before addressing and clock-stretch
overhead. That is **20–50 % of an entire tuned boot**, spent every wake, on a sensor we only need to
read four bytes from.

Two consequences, and the second is a correctness bug rather than a power one:

1. **Our BMI270 port must read `INTERNAL_STATUS` first and skip the reset+upload when the part is
   already initialised.** `IMU_VDD` comes from its own `ME6203A33M3G` LDO fed from `VBUS_PRE` — a
   rail that *survives main-power-off* (note 0006 §"The IMU") **[S]**. So the BMI270 stays configured
   and keeps counting across the cold-boot cycle. The blob needs uploading once per battery
   connection, not once per wake.
2. **The soft reset zeroes the step counter.** #6's BODY step-walk mode accrues steps while the
   board is off; calling `M5.Imu.begin()` on every wake would silently reset that accrual to zero
   every time. This is a finding for #16 as much as for #12.

### 4.3 The 500 ms shutdown tax

```c
// For PaperS3, the power cannot be turned off simply by setting the GPIO to LOW,
// so a loop is performed to ensure that the power is turned off by repeatedly
// outputting a pulse.
for (int i = 0; i < 5; ++i) {
  m5gfx::gpio_lo( pwrHoldPin );  m5gfx::delay(50);
  m5gfx::gpio_hi( pwrHoldPin );  m5gfx::delay(50);
}
```
— [`Power_Class.cpp:1154-1167`](https://github.com/m5stack/M5Unified/blob/4fb444784c85791e0b0207701392b42be234b2e7/src/utility/Power_Class.cpp#L1154-L1167)

**[D]** `5 × (50 + 50) ms = 500 ms` at full "main power on" current, on **every single wake**, purely
to hand the board back to the PMS150G. At 120 mA that is **16.7 µAh per wake**; at 1 wake/minute,
**168 mAh/week — 12 % of the budget spent on switching off.**

It is very likely reducible. The comment says the loop exists "to ensure" the power drops, which
reads as defensive rather than specified — the PMS150G presumably latches on the first valid pulse
and the remaining four are insurance. **[U]** Measuring how many pulses are actually required, and
how short they can be, is measurement 4 in §10, and it is the cheapest win available.

### 4.4 The per-wake cost model

**[D]** `E_wake = (t_boot + t_tick + t_off) × I_awake + N_scans × E_scan`

with `t_tick ≈ 20 ms` for an RTC read, a four-byte step-counter read and one `step()` call — the
simulation itself is an allocation-free pure function over a handful of fields (ADR-0005) and is
noise at this scale.

Using `t_boot ∈ [0.35, 0.70, 1.50] s`, `t_off = 0.50 s`, `I_awake ∈ [80, 120, 154] mA`, and the
per-scan energy band derived in §5:

| Wake shape | Scans | Optimistic | **Nominal** | Pessimistic |
|---|---|---|---|---|
| Tick only, no repaint | 0 | 19.3 µAh | **40.7 µAh** | 86.4 µAh |
| Tick + one `Animate` | 8 | 20.5 µAh | **44.7 µAh** | 98.4 µAh |
| Tick + one `Flip` | 11 | 21.0 µAh | **46.2 µAh** | 102.9 µAh |
| Tick + one `Clear` | 37 | 24.9 µAh | **59.2 µAh** | 141.9 µAh |

> **Note the shape of that table: the refresh is the *small* part.** A `Clear` — the most expensive
> repaint the device has, 37 full-panel scans — costs about the same as the shutdown pulse train
> that follows it (18.5 µAh against 16.7 µAh, nominal), and the two of them together are less than
> the boot. In the board-off architecture the panel is not the enemy; **the boot/shutdown envelope
> is.**

### 4.5 Light sleep versus board-off — the crossover, computed

This is the comparison the ticket asks for, and it is the most decision-relevant arithmetic in the
note, because the answer lands **inside** the range #11 is choosing from.

The standard textbook analysis compares light sleep (higher floor, ~free wake) against deep sleep
(near-zero floor, expensive wake). **On this board that comparison is the wrong one**, twice over:

- **Chip-level**, the light/deep difference is **[D]** `380 − 7 = 373 µA` **[S]** — a real but
  modest number.
- **Deep sleep does not actually avoid the wake cost.** Deep-sleep wake still runs the ROM and
  second-stage bootloaders and loses all SRAM and PSRAM, so on this board it costs essentially the
  same as a cold boot. The only thing it saves is the PMS150G re-energising the rails.
- **Light sleep is the one that is genuinely different**, because it *resumes* — RAM, PSRAM,
  M5GFX's 1.24 MiB of buffers, the BMI270's configuration, the NVS mount, all still there. A
  light-sleep wake costs microseconds, not hundreds of milliseconds.

So the real comparison is **board-off cold boot** (floor ~0, wake expensive) versus **light sleep
with the GT911 commanded to sleep** (floor ~0.79 mA, wake ~free):

**[D]** `T* = E_coldboot / (I_lightsleep − I_off) = 40.7 µAh / (0.793 mA − 0.0093 mA) = 0.052 h = 187 s`

| Case | `E_coldboot` | `I_lightsleep` | Break-even cadence **[D]** |
|---|---|---|---|
| Optimistic | 19.3 µAh | 0.733 mA | **96 s (1.6 min)** |
| **Nominal** | **40.7 µAh** | **0.793 mA** | **187 s (3.1 min)** |
| Pessimistic | 86.4 µAh | 0.863 mA | **364 s (6.1 min)** |

> **Faster than about 3 minutes, stay in light sleep. Slower than that, power the board off.** The
> crossover sits squarely in the middle of the cadence range #11 is choosing from, which means
> **#11's cadence decision and #11's rest-state decision are the same decision** and must be taken
> together.

Checked directly, ambient cost per week with a 2-second animation run per wake, nominal column:

| Cadence | Board-off cold boot | Light sleep, GT911 slept | Winner |
|---|---|---|---|
| 1 / 1 min | 1487 mAh | **1215 mAh** | light sleep |
| 1 / 2 min | 744 | **674** | light sleep |
| 1 / 3 min | 497 | **494** | light sleep, marginally |
| 1 / 5 min | **299** | 350 | board-off |
| 1 / 15 min | **101** | 205 | board-off |
| 1 / 60 min | **26** | 151 | board-off |

Two caveats that keep board-off ahead on merit even near the crossover:

1. **Light sleep with the GT911 asleep still cannot be woken by touch.** It buys cheap *scheduled*
   wakes, not responsiveness. Touch responsiveness needs the GT911 in Green mode, which costs
   3.3 mA — **672 mAh/week, 47 % of the budget** — and loses the argument at every cadence.
2. **The ~0.79 mA light-sleep floor is derived, not measured**, and its largest single term is the
   `SY8089AAAC`'s undocumented light-load efficiency (§3.3). Board-off's 9.3 µA is published by
   M5Stack and independently measured. **One of these two numbers is much better attested than the
   other**, which is a reason to prefer board-off wherever the arithmetic is close.
3. **The crossover moves with `t_boot`, which is a build setting.** On a tuned build it is ~3 min;
   with `CONFIG_SPIRAM_MEMTEST` left at its 8 MB default (§4.1) the cold boot costs ~2.5 s and the
   crossover slides out to **~10 minutes**, at which point light sleep wins across #11's whole
   plausible range. **Fix the boot before choosing the rest state**, or the measurement will answer
   a question about `sdkconfig` rather than about architecture.

Light sleep also earns its keep in a second, unambiguous place: **holding a session open**. Between
touch events inside a session the CPU can light-sleep with `gpio_wakeup_enable(GPIO48, LOW)` armed
([`Power_Class.cpp:1333-1341`](https://github.com/m5stack/M5Unified/blob/4fb444784c85791e0b0207701392b42be234b2e7/src/utility/Power_Class.cpp#L1333-L1341))
**[S]**, dropping from ~120 mA to ~4.1 mA between finger-downs while keeping touch responsive — the
GT911 stays in Green here because responsiveness is the whole point. For a session that is 80–90 %
waiting for the player, that is a **~4× saving on the session line item**. **[U]** — the residency
fraction has to be measured.


---

## 5. Per-scan energy — the acknowledged unknown, bounded

Note 0009 §7.4 handed this ticket a unit and an unknown: *"The unit to multiply is full-panel
scans, not updates and not pixels: 8 per `Animate`, 11 per `Flip`, 37 per `Settle` or `Clear`.
Energy per scan is unmeasured **[U]** and is the top item on the measurement plan."* This section
does not pretend to measure it. It bounds it, and then §7 shows how much the verdict moves across
the bound.

### 5.1 Definition — incremental, to avoid double-counting

**`E_scan` is defined here as the *incremental* charge one full-panel scan costs above an
already-energised, already-awake board.** The CPU baseline is carried by the awake-seconds term
(§2.3); `E_scan` carries only what the refresh *adds*: the EPD rail generation, the panel's driver
bias currents, and the extra CPU/PSRAM load of the blit over an idle core.

Defining it any other way double-counts the SoC and inflates every animation figure by roughly 2×.
This is the second classic error in these budgets, after the voltage conversion in §1.

### 5.2 The panel's switching energy is not the cost

A useful negative result, and it is derivable without the panel datasheet.

The ED047TC1 datasheet §3 gives the active area as **58.32 (H) × 103.68 (V) mm** **[S]** =
**60.47 cm²**. E Ink microcapsule film runs on the order of 1–2 nF/cm², so `C_panel ≈ 60–121 nF`.
The worst case per frame — every pixel flipping polarity across ±15 V — costs `C·V²`:

| Film capacitance | `C_total` | Worst-case switching energy/frame **[D]** |
|---|---|---|
| 1.0 nF/cm² | 61 nF | 13.6 µJ |
| 1.5 nF/cm² | 91 nF | 20.4 µJ |
| 2.0 nF/cm² | 121 nF | 27.3 µJ |

The gate side, at ~100 pF per row over a 42 V `VGH`→`VGL` swing across 540 rows, adds **[D]**
`540 × 100 pF × 42² = 95 µJ`. Total ≈ **115 µJ per frame**, which at 3.7 V is **[D]**
`115 µJ / 3.7 V / 3600 = 0.0086 µAh`.

> **The electrostatic work of actually moving the pigment is under 1 % of a scan's cost.** What a
> scan costs is *bias current* — the source and gate driver ICs' own consumption while they are
> energised, multiplied up by the boost converter — plus the SoC streaming a megabyte out of PSRAM.
> Optimising the *image* therefore cannot make a refresh cheaper; only shortening the time the rails
> are up can. That is the same conclusion note 0009 reached about wall-clock time, arriving from a
> completely different direction.

### 5.3 Boost amplification, which is where the multiplier lives

The rails are generated open-loop from the battery: MT3608 boost → `EPD_VPOS`, an inverting charge
pump → `EPD_VNEG`, zener clamps for `VGH`/`VGL`, resistive VCOM (note 0009 §5.2, schematic-verified)
**[S]**. So every milliamp the panel draws on a high rail is amplified at the battery:
**[D]** `I_batt = I_rail × V_rail / (V_batt × η)`.

**The MT3608 datasheet does not cover our operating point.** Its only low-`V_IN` efficiency data is a
curve at `V_OUT` = 18 V, `I_OUT` = 200 mA (3 V → 80.0 %, 4 V → 90.5 %, interpolating ≈ 87–88 % at
3.7 V), and there is nothing below ~100 mA
([MT3608 datasheet](https://www.olimex.com/Products/Breadboarding/BB-PWR-3608/resources/MT3608.pdf))
**[S]**. Our load is 4–5 mA, where its 1.6 mA PWM quiescent alone is a sixth of the input current.

**So the efficiency figures come from the TI TPS65185** — the reference PMIC for exactly this class
of panel, and the only part in this space that publishes efficiency curves at `V_IN` = 3.5 V, which
is essentially our battery voltage
([SLVSAQ8G](https://www.ti.com/lit/ds/symlink/tps65185.pdf), §9.2.3 Figs. 47–50, 25 °C) **[S]**:

| Stage | 1 mA | 10 mA | 25 mA |
|---|---|---|---|
| VB boost (+16 V) | — | **76.9 %** | 80.8 % |
| VN inverting buck-boost (−16 V) | — | **70.7 %** | 73.2 % |
| **CP1 → VDDH (+22 V) charge pump** | **36.3 %** | 54.1 % | — |
| **CP2 → VEE (−20 V) charge pump** | **32.5 %** | 44.6 % | — |

Chaining boost × charge pump gives battery→`VGH` ≈ **28 %** and battery→`VGL` ≈ **23 %** **[D]**.

| Rail | Voltage | Path efficiency | Battery mW per panel mW **[D]** |
|---|---|---|---|
| `EPD_VPOS` | +15 V | 0.769 × 0.9375 = 0.721 | 1.39× |
| `EPD_VNEG` | −15 V | 0.707 × 0.9375 = 0.663 | 1.51× |
| `VGH` | +22 V | 0.769 × 0.363 = 0.279 | **3.58×** |
| `VGL` | −20 V | 0.707 × 0.325 = 0.230 | **4.35×** |
| `VDD` | 3.3 V | 3.3/3.7 = 0.892 | 1.12× |

> **The gate rails are 18 % of the panel's power draw but 38 % of the battery's.** Applying the
> table to the ED047TC1's typ currents (§5.4): **168.1 mW at the panel becomes 320.4 mW at the
> battery — an overall path efficiency of 52.5 %** **[D]**, not the 80 % a naive single-boost
> assumption would give. **Nearly half the energy of every refresh is burned in the rail generation,
> most of it in the +22/−20 V gate supplies.**

And our board is probably worse than that, because we do not have a TPS65185: `VGH`/`VGL` are made
with **shunt zener clamps** (note 0009 §5.2) **[S]**, and a shunt clamp burns
`(V_source − V_zener) × I_bias` continuously while the rails are up. At a plausible 1–3 mA of clamp
bias at 22 V that is **22–66 mW — 1.4× to 4× the panel's own 15.4 mW `VGH` draw**, all of it waste.
**[U]** This is the largest unquantified term in the refresh cost, it is a property of *our*
schematic rather than of the panel, and it is directly measurable (§10).

### 5.4 The bound

Combining: a scan lasts one frame period — 11–20 ms depending on `CONFIG_FREERTOS_HZ` (note 0009
§1.6) **[D]** — and adds an increment `ΔI` over the awake baseline.

**And the panel's own datasheet gives the rail currents outright**, which turns most of this from
guesswork into arithmetic. E Ink Holdings *Technical Specification, Model No. ED047TC1*, doc
P-511-710(V:1), §6-2 "Display Module DC characteristics"
([PDF](https://raw.githubusercontent.com/Xinyuan-LilyGO/LilyGo-EPD47/esp32s3/datasheet/ED047TC1.pdf))
**[S]**:

| Rail | Symbol | Voltage | I typ | I max | P typ **[D]** | P max **[D]** |
|---|---|---|---|---|---|---|
| Logic supply | `VDD` / `IVDD` | 3.3 V | 0.7 mA | 1.65 mA | 2.3 mW | 5.4 mW |
| Gate negative | `VGL` / `IGL` | −20 V | 0.77 mA | 3.1 mA | 15.4 mW | 62.0 mW |
| Gate positive | `VGH` / `IGH` | +22 V | 0.7 mA | 1.6 mA | 15.4 mW | 35.2 mW |
| Source negative | `VNEG` / `INEG` | −15 V | **4.57 mA** | **20 mA** | 68.6 mW | 300 mW |
| Source positive | `VPOS` / `IPOS` | +15 V | **4.43 mA** | **22 mA** | 66.5 mW | 330 mW |
| Common | `VCOM` / `ICOM` | adjusted | 0.08 mA | — | ~0 | ~0 |
| **Panel Power** | `P` | | | | **170 mW typ** | **950 mW max** |
| **Standby power** | `PSTBY` | | | | | **0.1 mW max** |

**[D]** The per-rail typ figures sum to **168.1 mW** against the stated 170 mW — they reconcile, so
the table is being read correctly. The conditions are stated too: max is *"measured using 85 Hz
waveform … from repeated 1 consecutive black scan lines followed by 1 consecutive white scan line to
that of repeated 1 white followed by 1 black"* — a full-screen worst-case line-inversion flip. Typ
is *"85 Hz waveform … from horizontal 4 gray scale pattern to vertical 4 gray scale pattern"*.
**Our `Animate` on a ~5 %-of-screen pet window is well below the typ condition**, so 170 mW is a
conservative working figure and 950 mW belongs only to a full-screen `Clear` of high-contrast
content.

Also from the same page, and both useful: **`ICOM` inrush is up to 480 mA** — that is the rail
bring-up transient, and an averaging meter will miss it — and **standby power is ≤ 0.1 mW**, i.e.
`27 µA` at 3.7 V **[D]** even with the rails up and the panel idle. With `PWR` (G46) low it is zero.

| Component of `ΔI` | Optimistic | Nominal | Pessimistic | Basis |
|---|---|---|---|---|
| Panel input power | 100 mW (small region) | **168 mW** (datasheet typ) | **755 mW** (Σ per-rail max) | **[S]** |
| → referred to battery at the §5.3 per-rail efficiencies | 51 mA | **86.6 mA** | **366.6 mA** | **[D]** |
| Blit: CPU + PSRAM read over idle | 30 mA | 60 mA | 100 mA | **[D]** datasheet Table 5-9 spread, 32-bit → 128-bit access at 240 MHz |
| Frame period | 11 ms | 14 ms | 20 ms | **[D]** note 0009 §1.6 |
| **`E_scan`, derived** | **0.25 µAh** | **0.57 µAh** | **2.59 µAh** | **[D]** |
| **Band used in the tables below** | **0.15** | **0.50** | **1.50 µAh** | see note |

> **Reconciling the two rows.** The tables in §6–§8 were computed with **0.15 / 0.50 / 1.50 µAh**.
> The datasheet-derived nominal, **0.57 µAh**, is 14 % above the 0.50 used — immaterial. The
> derived pessimistic, **2.59 µAh**, is above the 1.50 used, and it corresponds to a *full-screen
> worst-case line-inversion flip*, not to the small-region `Animate` the design actually issues.
> **§7.3 sweeps `E_scan` to 2.5 µAh explicitly and the verdict still clears one week with 88 %
> headroom**, so nothing downstream needs restating — but read the pessimistic column of every
> other table as slightly optimistic on this one term.

Which gives, per refresh intent **[D]**:

| Intent | Scans | Optimistic | **Nominal** | Pessimistic | Datasheet worst case |
|---|---|---|---|---|---|
| `Animate` | 8 | 1.2 µAh | **4.0 µAh** | 12.0 µAh | 18.7 µAh |
| `Flip` | 11 | 1.7 µAh | **5.5 µAh** | 16.5 µAh | 25.7 µAh |
| `Settle` / `Clear` | 37 | 5.6 µAh | **18.5 µAh** | 55.5 µAh | 86.5 µAh |

**Sanity check against the one measured whole-board number we have, and it lands.** M5Stack's
"operating mode" is **154.02 mA at 4.2 V = 647 mW** **[S]**. Subtract the ESP32-S3 at 240 MHz
(66.2–81.3 mA at 3.3 V = 218–268 mW, Table 5-9) and ~33 mW for the flash **[S]**, and
**[D]** ~**346–396 mW is left for the panel subsystem** — against the 320 mW this section models
from the ED047TC1's typ currents and the TPS65185's efficiency curves. **Two completely independent
routes agree within ~20 %.**

That also answers a question §2.3 left open: **M5Stack's "operating mode" figure is the board awake
with the EPD rails up**, not with the radio associated. The 154 mA end of the `I_awake` band is
therefore a *refreshing* board, and the honest awake-idle figure is nearer the 80–110 mA end. The
model is conservative by roughly the panel term, which is the right direction to be wrong in.

For scale in the other direction, other boards driving this panel report: LilyGo T5-ePaper-S3
*"Working: 90–230+ mA (240 MHz, WiFi on); Sleep: ~380 µA"*, and epdiy V5 *"< 13 µA deep sleep"*
after `epd_deinit()` **[S]**. The ~380 µA LilyGo sleep figure is a useful independent corroboration
of §3.3's ~0.4 mA estimate for a PaperS3 in deep sleep with the touch controller dealt with.

### 5.5 Why the verdict is *not* very sensitive to this

The honest thing to say about a number this uncertain is how much it matters, and here the answer is
reassuring: **swinging `E_scan` across its whole datasheet-bounded 12× range moves the headline
verdict by about −46 % / +17 %, and never changes it** (§7.3). The reason is structural: in the
board-off architecture a refresh happens inside an awake window that already costs
`t_boot + t_off ≈ 1.2 s` of full-power time, and 8 scans of `Animate` add 4 µAh to a 40 µAh
envelope. **The panel is a passenger; the boot/shutdown envelope is the vehicle.**

This is a genuinely different answer from the one #12 was expected to produce, and it changes what
should be measured first: **the shutdown pulse train and the boot path are higher-value measurements
than the panel current**, which is the opposite of note 0009's priority order — correctly, because
0009 was budgeting *time* and this note is budgeting *energy*.

### 5.6 What M5GFX does with the rails, at source

Confirmed rather than assumed, because "does the panel rail actually collapse when we are not using
it" is the question note 0009 §6.2 left open for this ticket.

```c
bool Bus_EPD::powerControl(bool flg_on) {
  if (_pwr_on != flg_on) {
    _pwr_on = flg_on;  wait();
    if (flg_on) {
      gpio_hi(pin_oe);   delayMicroseconds(100);
      gpio_hi(pin_pwr);  delayMicroseconds(100);   // GPIO46
      gpio_hi(pin_spv);  delay(1);
    } else {
      delay(1);
      gpio_lo(pin_pwr);  delayMicroseconds(10);    // GPIO46
      gpio_lo(pin_oe);   delayMicroseconds(100);
      gpio_lo(pin_spv);
    }
  }
  return true;
}
```
— [`Bus_EPD.cpp:77-97`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Bus_EPD.cpp#L77-L97)

Three things follow **[S]**:

1. **The rails come up once per update *burst*, not once per scan.** `powerControl(true)` sits at
   the top of the scan loop
   ([`Panel_EPD.cpp:1044`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L1044))
   and is a no-op when already on; `powerControl(false)` runs only when `remain == false`, i.e. when
   the whole queue has drained
   ([`:1067-1068`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L1067-L1068)).
   **A back-to-back animation run therefore pays one rail cycle, not ten.** Queueing frames rather
   than issuing them one at a time is a real saving, and it is a constraint on how #13's animation
   effects should be emitted.
2. **The bring-up and tear-down each cost one FreeRTOS tick** (`lgfx::delay(1)`). At the stock
   100 Hz that is **10 ms each**; at `CONFIG_FREERTOS_HZ=1000` (which ADR-0008 already mandates)
   it is **1 ms each** **[D]**. Another reason the tick-rate setting is correctness-adjacent.
3. **The datasheet's ordered power-down is *not* honoured.** ED047TC1 §7 specifies the sequence
   `VSS→VDD→VNEG→VPOS→VCOM` and `VSS→VDD→VGL→VGH` on the way up with `T_np`, `T_eg` ≥ 1000 µs, and
   on the way down "begin to turn off VGL power after VNEG and VPOS are completely or almost
   discharged", `T_ed` ≥ **0.5 s**, "discharged point @ −7.4 V" **[S]**. M5GFX drops `PWR`, then
   `OE` 10 µs later, then `SPV` 100 µs after that — ~110 µs total. Whether the discrete rail network
   self-sequences on discharge is **[U]**, but it is worth recording as a candidate contributor to
   the unattributed panel-damage reports in note 0009 §6.5, and as a reason *not* to cycle the rails
   more often than necessary. **Corollary for the animation policy: batching a run into one rail
   cycle is a durability argument as well as a power one.**

**Correcting the pin semantics, which notes 0002 and 0006 both got slightly wrong.** M5GFX names its
two EPD control pins `PWR` (GPIO46) and `OE` (GPIO45). The schematic says what they actually do
**[S]**:

| M5GFX name | GPIO | Net | What it switches |
|---|---|---|---|
| `pin_oe` | **45** [strap] | `EPD_PWR` | The **panel's own logic supply** — `EPD1` pin 7 `VDD`, plus pin 12 `XOE`, pin 23 `TEST`, pin 38 `MODE1`. Driven straight off the GPIO, with `R39` 5.1 K pull-down and `C43` 1 µF. |
| `pin_pwr` | **46** [strap] | `BST_EN` | The `MT9700` load switch (`U8`) gating `SYS_MAIN → BST_VIN`, i.e. **the entire high-voltage generator**. `R31` 10 K pull-down. |
| — | 44 | `PWROFF_PLUSE` | `R46` 1 K → `D1` 1N4148 into the PMS150G power latch. |

So M5GFX's `powerControl(true)` — OE high, 100 µs, PWR high — is *"energise the panel logic, then
enable the boost"*, which is a coherent and correct sequence rather than the arbitrary one the pin
names suggest.

**And the rail can be cut completely, with no leakage path.** **[S]** The `MT3608`'s own `EN` is
tied to its `IN` on this schematic, so it can never be disabled independently — but it does not need
to be, because `EPD_VPOS` and `EPD_VNEG` are **AC-coupled** off the switch node through series 1 µF
capacitors (`C36`, `C32`). There is **no DC path** from `BST_VIN` to any EPD rail. With `U8` off,
`VPOS`/`VNEG`/`VGH`/`VGL`/`VCOM` all collapse and are actively drained by four 120 K bleeders
(`R30`/`R33`/`R34`/`R40`). **EPD parasitic in any sleep state is zero.** That is the ticket's
question — *"whether the panel's power rail can be fully cut in sleep"* — answered: **yes,
completely.**

> **Two things not to do.** (a) **Never `gpio_deep_sleep_hold_en()` GPIO45 high.** It is the
> `VDD_SPI` voltage-select strap (low → 3.3 V, high → 1.8 V) **[S]**; `R39` guarantees it reads low
> at reset, and a held-high pad would bring the board up with `VDD_SPI` at 1.8 V and fail to boot
> its 3.3 V flash. (b) **You do not need to hold them at all** — `R31` and `R39` pull both nets to
> the *off* state when the pads go high-impedance in sleep, which is exactly why M5GFX and M5Unified
> contain zero calls to `gpio_hold_en` / `gpio_deep_sleep_hold_en` **[S]**.

**Two costs that exist only while the rails are up**, and both are pure schematic waste: `R39`
(5.1 K on `EPD_PWR`) draws **647 µA**, and the VCOM divider `R37`+`R38` (6.6 K across ~15 V) draws
**~2.3 mA**, plus ~0.6 mA in the bleeders **[D]**. Together **~3.5 mA** — small against a refresh
but a reason not to leave the rails energised between updates. M5GFX already does not.

**One hardware constraint the datasheet is explicit about: inrush.** ED047TC1 §6-2 note — *"The
maximum `I_COM` inrush current is about **480 mA**"* **[S]**. Community reports on the same panel
class describe ~0.6–0.9 A at `epd_init()` browning out an undersized regulator. It does not change
average energy, but it is why a rail cycle is not free and why bulk capacitance matters.

---

## 6. The line items

All figures battery-referred, nominal column, for a week. Percentages are of the **1440 mAh nominal
usable** capacity from §2.1.

| # | Line item | Driver | Nominal mAh/week | % of 1440 | Tag |
|---|---|---|---|---|---|
| 1 | **Sleep floor — chip** | ESP32-S3 deep sleep, 7 µA | *(does not apply — board is off)* | — | **[S]** |
| 2 | **Sleep floor — board, off** | 9.28 µA whole board | **1.6** | **0.1 %** | **[S]** |
| 2b | *(alt.)* deep sleep, GT911 slept, EPD cut | ~0.42 mA | *71* | *4.9 %* | **[D]** |
| 2c | *(alt.)* light sleep, GT911 slept, state retained | ~0.79 mA | *133* | *9.3 %* | **[D]** |
| 2d | *(alt.)* deep sleep as the vendor libraries leave it | 5.1 mA | *857* | *59.5 %* | **[2]** |
| 3 | **Ambient wakes** | 2 s animation run every 5 min | **297.0** | **20.6 %** | **[D]** |
| 3a | — of which boot | 0.70 s × 120 mA × 2016 | 47.0 | 3.3 % | **[D]** |
| 3b | — of which the animation itself (awake time) | 2.0 s × 120 mA × 2016 | 134.4 | 9.3 % | **[D]** |
| 3c | — of which panel scans | 80 scans × 0.50 µAh × 2016 | 80.6 | 5.6 % | **[D]** |
| 3d | — of which **shutdown pulse train** | 0.50 s × 120 mA × 2016 | **33.6** | **2.3 %** | **[S]/[D]** |
| 3e | — of which `step()` + I²C | 0.02 s × 120 mA × 2016 | 1.3 | 0.1 % | **[D]** |
| 4 | **Sessions** | 4/day × 90 s, ~500 scans | **99.1** | **6.9 %** | **[D]** |
| 4a | — with 80 % light sleep between touches | | *35.0* | *2.4 %* | **[U]** |
| 5 | **Game rounds** | INSTINCT, 30 s @ 5 fps, 2/day, 1.6 mAh each | **22.4** | **1.6 %** | **[D]** |
| 6 | **WiFi backup** | 5/day, 7 s radio | **10.2** | **0.7 %** | **[D]** |
| 7 | **Flash writes (NVS)** | ~30 saves/day | **< 0.1** | **< 0.01 %** | **[D]** |
| 8 | **Battery self-discharge** | 3 %/month typical (spec floor 10–20 %/mo) | **12.6** | **0.9 %** | **[U]** |
| 8a | — pack protection circuit, ~7 µA | module-level datasheet figure | 1.2 | 0.1 % | **[S]** |
| 9 | **Usable-capacity derating** | *(applied to the denominator, not a line)* | −360 | −20 % | **[U]** |
| | **Total (scenario B, sessions at full power)** | | **~414** | **28.7 %** | |

### 6.1 Notes on the small lines

**Flash writes are noise, and this closes a question #5 raised.** Note 0005 flagged "flash write
costs per wake may matter". They do not. The ESP32-S3 datasheet gives, from the flash vendor,
4 KB sector erase **typ 70 ms / max 500 ms** and page program **typ 0.8 ms / max 5 ms**
(Table 5-11) **[S]**. NVS's append-only log reduces erases by a factor of 126 (note 0005), so at
~30 saves/day the overwhelming majority are single-page programs. **[D]** `30 × 5 ms × ~20 mA
= 3 mA·s = 0.83 µAh/day = 0.006 mAh/week`. Even charging every save a full 500 ms worst-case erase
gives 0.9 mAh/week. **Delete this line from the conversation.** The reason #5's instinct was right in
spirit is that a *save* forces a wake — and the wake, not the write, is what costs.

**WiFi is smaller than self-discharge.** Note 0004 called upload cadence "the cheapest lever on
#12". The arithmetic says it is not a lever at all: at 5 uploads/day and a generous 7 s radio
session, backup costs **10.2 mAh/week, 0.7 %** — less than the cell leaks by itself. Even at
24 uploads/day it is 2.1 % **[D]**. See §6.2 for why the connect time band is what it is.

**The buzzer** is a few tens of milliseconds of a piezo at a few mA, a handful of times a day. Below
the resolution of this model. Not modelled.

**The BMI270** in accel-only low power with the step counter running is 10 µA at 25 Hz, or 4 µA at
lower ODR (note 0006 §"The IMU", from the Bosch datasheet) **[S]** — and it is already *inside*
M5Stack's 9.28 µA figure, so it is not a separate line. Turning the gyro on costs 940 µA
(**159.5 mAh/week, 11.1 %**) and is therefore a session-only expense, exactly as #6 concluded.

### 6.2 WiFi per backup, in more detail

**The cost of a backup is radio-on *time*, not bytes.** A <8 KiB snapshot (note 0004) plus TLS
records and TCP overhead is ~20 KB on the wire, which at 802.11n HT20 MCS7 is **[D]** ~2.2 ms of
airtime — under 1 % of the burst. Everything else is waiting.

TX peak is **340 mA** (802.11b 1 Mbps @21 dBm) or **283 mA** (802.11n HT20 MCS7 @18.5 dBm), RX is
**88 mA** — ESP32-S3 datasheet Table 5-7 at 3.3 V / 25 °C, *"TX current consumption is rated at a
100 % duty cycle"*, and the column is headed **Peak**, not average **[S]**. Note the datasheet has
**no "associated but idle" row for the S3** at all, so the working figure below — ~121 mA for
radio-on with the CPU active — is composed: RX 88 mA plus the active-CPU delta
(66.2 − 32.9 = 33 mA from Table 5-9) **[D]**.

**Espressif publish almost no connect-phase timings.** What they *do* publish turns out to be the
two biggest costs, and both are ESP-IDF defaults that can simply be turned off:

| Phase | Duration | Source |
|---|---|---|
| Per-channel scan dwell | **120 ms active / 360 ms passive**; `home_chan_dwell_time` 30 ms | [`esp_wifi` API reference](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/network/esp_wifi.html) **[S]** |
| Full-channel scan | *"11 × 120 + 2 × 360 = **2040 ms**"* | [ESP-FAQ](https://docs.espressif.com/projects/esp-faq/en/latest/software-framework/wifi.html) **[S]** |
| **DHCP ARP check** | *"this process lasts **1 – 2 seconds**"* | `components/lwip/Kconfig`, `LWIP_DHCP_DOES_ARP_CHECK` (default **on**) **[S]** |
| **SNTP startup delay** | *"a random number of milliseconds between 0 and this value"*, `LWIP_SNTP_MAXIMUM_STARTUP_DELAY` default **5000 ms** | `components/lwip/Kconfig` (default **on**) **[S]** |
| PHY full vs partial calibration | *"full calibration takes about **100 ms** more"* | `components/esp_phy/Kconfig` **[S]** |
| RSA-2048 public op (TLS cert verify) | **≤ 18 ms** | ESP-IDF CI threshold, `idf_performance_target.h` (esp32s3, v5.5) **[S]** |
| auth + assoc + WPA2 4-way handshake | — | **no first-party figure exists** **[U]** |
| TLS handshake wall clock | — | **no first-party figure exists** **[U]** |

> **On ESP-IDF defaults, a backup spends up to 2 s in a DHCP ARP check and up to 5 s doing nothing
> at all waiting out the SNTP startup delay.** Those two defaults alone can be most of the burst.
> Set `CONFIG_LWIP_DHCP_DOES_NOT_CHECK_OFFERED_IP=y` (or use a static lease) and
> `CONFIG_LWIP_SNTP_STARTUP_DELAY=n`.

| Configuration | Radio-up | Per backup **[D]** | 5/day, per week |
|---|---|---|---|
| Stock ESP-IDF defaults | 2.5 – 12.2 s | 0.08 – 0.40 mAh | 2.8 – 14.1 mAh |
| **This note's working figure** | **7 s** | **0.29 mAh** | **10.2 mAh (0.7 %)** |
| Tuned: BSSID+channel pinned, static IP, no SNTP delay, cached B2 tokens | 0.7 – 2.1 s | **0.02 – 0.06 mAh** | **0.7 – 2.1 mAh (0.1 %)** |

**So the line item as modelled is ~5× conservative.** Left in deliberately — the whole term is under
1 % either way, and it is better to be wrong in this direction.

Three notes for whoever implements it:

- **ESP32-S3 has no ECC/ECDSA hardware.** AES-128/256, SHA-1…512, RSA/MPI to 4096 bits and HMAC are
  all accelerated **[S]**, but `SOC_ECC_SUPPORTED` is undefined for `esp32s3` and there is no ECC
  chapter in the TRM. **Every ECDHE key agreement in a TLS 1.2/1.3 handshake runs in software on
  this chip.** RSA cert verification is the fast part (≤18 ms); the key exchange is the slow part
  and nobody publishes a number for it.
- **PHY calibration is skipped on deep-sleep wake but not on power-on reset.**
  `CONFIG_ESP_PHY_CALIBRATION_AND_DATA_STORAGE` defaults on and states *"PHY calibration will be
  skipped on deep sleep wakeup"* **[S]** — but our rest state is a **cold boot**, so we pay the
  ~100 ms full-calibration penalty on every backup. Worth folding into `t_boot` for radio wakes
  specifically.
- **B2's three-call dance means two TLS handshakes to two different hosts.**
  `b2_authorize_account` and `b2_get_upload_url` go to `api.backblazeb2.com`; the upload URL points
  at a different pod. **Caching the auth token and upload URL across wakes (valid 24 h, note 0004)
  collapses the burst to one handshake and one POST, and is worth more than every Kconfig tweak
  combined.**

Note 0004's own recommendation — *"flush the queue when the device is awake anyway — during a
Session, or on the wake that a forced snapshot triggers"* — removes the boot and shutdown envelope
too, a further ~40 µAh per backup **[D]**.

> **Conclusion for #4: stop worrying about upload cadence.** Back up on every state change if you
> like. The lever is not there.


---

## 7. The parametric model, and the sensitivity tables

### 7.1 Why this is parametric and not a point estimate

**#12 is blocked by #11, and #11 is still open.** The ticket says "multiply by the cadence from the
state machine ticket" — but that cadence does not exist yet, and if this note picked one it would be
inventing #11's answer and then validating it, which is worthless.

So the model is built with the four things #11 and #13 will decide left as **free variables**:

| Symbol | Free variable | Owner | Range explored |
|---|---|---|---|
| `T_amb` | ambient wake period | **#11** | 1 – 60 min |
| `t_anim`, `k` | idle animation run length and frame count | **#13** | 0 – 4 s, 0 – 20 updates |
| `n_sess`, `t_sess` | sessions/day and session idle timeout | **#11** | 2 – 10/day, 30 – 300 s |
| `n_bk` | backups/day | **#4** | 1 – 24/day |

and three hardware constants left as **measurement bands**: `I_awake` (80 / 120 / 154 mA), `t_boot`
(0.35 / 0.70 / 1.50 s), `E_scan` (0.15 / 0.50 / 1.50 µAh). The three columns of every table below
are those bands taken together — *optimistic*, *nominal*, *pessimistic*.

**This is a more useful deliverable to #11 than a single number**, because it lets #11 choose a
cadence knowing the price of each option rather than discovering it afterwards. The complete model
is four lines:

```
E_ambient_wake = (t_boot + t_tick + t_anim + t_shutdown) × I_awake  +  8·k × E_scan
E_session      = t_sess × I_session_eff + N_scans × E_scan + (t_boot + t_shutdown) × I_awake
E_backup       = t_radio × I_radio + (t_boot + t_shutdown) × I_awake
Total/week     = 168 h × I_floor + Σ(rate × E_event) + self-discharge
```

### 7.2 Ambient cadence — the dominant term

Each wake carries one 2-second idle animation run at 5 fps (10 `Animate` updates, 80 scans).

| Ambient cadence | Wakes/week | Optimistic | **Nominal** | Pessimistic |
|---|---|---|---|---|
| 1 / **1 min** | 10 080 | 764 mAh | **1485 mAh** | 2943 mAh |
| 1 / **2 min** | 5 040 | 382 | **743** | 1472 |
| 1 / **3 min** | 3 360 | 255 | **495** | 981 |
| 1 / **5 min** | 2 016 | 153 | **297** | 589 |
| 1 / **10 min** | 1 008 | 76 | **149** | 294 |
| 1 / **15 min** | 672 | 51 | **99** | 196 |
| 1 / **30 min** | 336 | 25 | **50** | 98 |
| 1 / **60 min** | 168 | 13 | **25** | 49 |

Against a **1440 mAh** budget. The same sweep with **no animation** — a bare simulation tick and no
repaint — costs far less, and the gap between the two rows is exactly what the idle animation is
buying:

| Ambient cadence | Optimistic | **Nominal** | Pessimistic |
|---|---|---|---|
| 1 / 1 min | 195 mAh | **410 mAh** | 871 mAh |
| 1 / 5 min | 39 | **82** | 174 |
| 1 / 15 min | 13 | **27** | 58 |
| 1 / 60 min | 3.2 | **6.8** | 14.5 |

### 7.3 How much does the unmeasured per-scan energy actually matter?

Scenario B (below) at nominal, sweeping `E_scan` across and beyond its whole derived band:

| `E_scan` | Weekly total | Runtime | vs. nominal |
|---|---|---|---|
| 0.15 µAh (optimistic bound) | 352 mAh | **28.6 days** | +17 % |
| **0.50 µAh (nominal)** | **414 mAh** | **24.4 days** | — |
| 1.00 µAh | 501 mAh | 20.1 days | −18 % |
| 1.50 µAh (pessimistic bound) | 589 mAh | 17.1 days | −30 % |
| 2.50 µAh (beyond the bound) | 764 mAh | 13.2 days | −46 % |

> **A 10× swing in the least-known constant in this note moves the answer by 1.7×, and never
> changes the verdict.** Even at 2.5 µAh/scan — well above anything the physics of §5 supports —
> scenario B clears one week with 88 % headroom. **`E_scan` is worth measuring, but it is not what
> the verdict hangs on.** What the verdict hangs on is the ambient cadence, and that is #11's to
> choose.

### 7.4 Four named configurations

Sessions costed at full power (no light sleep between touches). Self-discharge and floor included.

| Configuration | Optimistic | **Nominal** | Pessimistic |
|---|---|---|---|
| **A — "lively"**: animation 1/min, 6 sessions/day × 180 s, 6 backups | 964 mAh → **10.5 d** | 1784 mAh → **5.6 d** | 3355 mAh → **3.0 d** |
| **B — "balanced"**: animation 1/5 min, 4 sessions/day × 90 s, 5 backups | 235 → **42.9 d** | 414 → **24.4 d** | 746 → **13.5 d** |
| **C — "frugal"**: animation 1/15 min, 4 sessions/day × 60 s, 2 backups | 108 → **93.3 d** | 179 → **56.4 d** | 302 → **33.4 d** |
| **D — "still"**: no idle animation, tick 1/15 min, 4 × 60 s, 2 backups | 70 → **143.8 d** | 107 → **94.3 d** | 164 → **61.5 d** |

Bold-face reading: **A fails, B, C and D pass in every column.**

Configuration D is the interesting control. It is what the device would be *without* the thing the
battery was traded for, and it runs for three months. **The gap between D (94 days) and A (5.6 days)
is the entire cost of idle animation** — a 17× swing, and it is the single largest term in the whole
budget. Everything else is rounding.


---

## 8. The verdict

### 8.1 Does it hit one week?

**Yes — comfortably, and by a wide margin — on three conditions, two of which are just "do the
obvious thing" and one of which is a real design trade.**

> **Condition 1 (a bug fix, not a trade): if the board stays powered in any rest state, the GT911
> must be commanded to sleep.** Left scanning, as both vendor libraries leave it, it costs
> 3.3–8 mA — 550–1400 mAh/week, most or all of the budget. One I²C write (`0x05` → `0x8040`, with
> `INT` driven low first) removes it. **This is the single highest-value line of code in the whole
> power model**: it moves "ambient in ESP32 deep sleep" from 7.9 days to 20.9 days, **+13 days**
> (§8.3). In the board-off architecture it costs nothing either way — the controller is unpowered —
> so write it regardless and stop the choice of rest state from depending on it.
>
> **Condition 2 (architectural): ambient rests with the board off** (PMS150G + BM8563 alarm, cold
> boot on wake) **at cadences slower than ~3 minutes, or in light sleep with the GT911 asleep at
> cadences faster than that** (§4.5). Board-off is the better-attested of the two — 9.28 µA is
> published *and* independently measured *and* reproduced by summing the schematic — so prefer it
> where the arithmetic is close.
>
> **Condition 3 (the real trade): the idle animation runs no faster than about one run every three
> minutes.** At 2 seconds of 5 fps per run, the break-even cadence that exactly consumes a
> 1440 mAh week is:

| Animation run | Optimistic | **Nominal** | Pessimistic |
|---|---|---|---|
| 1 s @ 5 fps (5 updates) | 1 per 0.35 min | **1 per 0.72 min** | 1 per 1.49 min |
| **2 s @ 5 fps (10 updates)** | 1 per 0.56 min | **1 per 1.12 min** | **1 per 2.29 min** |
| 4 s @ 5 fps (20 updates) | 1 per 0.98 min | **1 per 1.93 min** | 1 per 3.91 min |
| no animation, bare tick | 1 per 0.14 min | **1 per 0.31 min** | 1 per 0.68 min |

**So: a 2-second animation run once every 3 minutes clears one week in all three columns.** Once
every 5 minutes clears it with 3.5× headroom at nominal and 1.9× at pessimistic. Once a *minute*
misses at nominal (5.6 days) and misses badly at pessimistic (3.0 days).

**Restated against the three usable-capacity cases** (§2.1), since the tables above use a 1440 mAh
design budget that is deliberately ~10 % below the sourced nominal:

| Configuration (nominal hardware column) | 1170 mAh (end of life) | **1440 mAh (this note)** | 1590 mAh (sourced nominal) |
|---|---|---|---|
| A "lively" — animation 1/min | 4.6 d ❌ | **5.6 d ❌** | 6.2 d ❌ |
| Animation 1/3 min | 13.4 d ✅ | **16.5 d ✅** | 18.2 d ✅ |
| **B "balanced" — animation 1/5 min** | **19.8 d ✅** | **24.4 d ✅** | **26.9 d ✅** |
| C "frugal" — animation 1/15 min | 45.8 d ✅ | **56.4 d ✅** | 62.3 d ✅ |

**Configuration A fails in every capacity case; everything at 1/3 min or slower passes in every
capacity case.** The verdict does not depend on which cell M5Stack fitted.

### 8.2 Where the model misses, and by how much

The one configuration that fails is A, "lively" — and it is worth being precise about it, because it
is the configuration a designer would naturally reach for:

| | Optimistic | **Nominal** | Pessimistic |
|---|---|---|---|
| A "lively" (animation 1/min, 6 × 180 s sessions, 6 backups) | 10.5 d ✅ | **5.6 d ❌** | 3.0 d ❌ |
| Shortfall against 7 days | +50 % | **−20 %** | −57 % |
| Overspend against 1440 mAh | — | **+344 mAh (+24 %)** | +1915 mAh (+133 %) |

**At nominal, configuration A misses one week by 1.4 days, and needs to shed 344 mAh — 24 % — to
make it.** That is a small enough gap that it can be closed several ways, which is the point of the
next section.

### 8.3 The levers, with numbers attached

From scenario B at nominal (414 mAh/week, 24.4 days), changing exactly one thing:

| Lever | Owner | Weekly | Runtime | Δ days |
|---|---|---|---|---|
| **Ambient/animation cadence 1/5 min → 1/1 min** | **#11** | 1602 mAh | 6.3 d | **−18.1** |
| Ambient cadence 1/5 min → 1/2 min | #11 | 859 | 11.7 d | −12.6 |
| Ambient cadence 1/5 min → 1/3 min | #11 | 612 | 16.5 d | −7.9 |
| **Ambient cadence 1/5 min → 1/15 min** | **#11** | 216 | 46.8 d | **+22.4** |
| Ambient cadence 1/5 min → 1/60 min | #11 | 141 | 71.3 d | +47.0 |
| Animation run 2 s → 4 s @ 5 fps | #13 | 629 | 16.0 d | −8.3 |
| **Animation run 2 s → 1 s @ 5 fps** | **#13** | 306 | 32.9 d | **+8.6** |
| Idle animation removed entirely | #13 | 199 | 50.8 d | +26.4 |
| **Session idle timeout 90 s → 300 s** | **#11** | 624 | 16.2 d | **−8.2** |
| Session idle timeout 90 s → 30 s | #11 | 353 | 28.6 d | +4.2 |
| Sessions 4/day → 10/day | player | 552 | 18.3 d | −6.1 |
| **Light-sleep 80 % of session dwell** | port | 350 | 28.8 d | **+4.5** |
| Backups 5/day → 24/day | #4 | 453 | 22.3 d | −2.1 |
| Backups 5/day → 1/day | #4 | 405 | 24.9 d | +0.5 |
| Shutdown pulse train 500 ms → 100 ms | port | 386 | 26.1 d | +1.8 |
| Boot 700 ms → 350 ms (`sdkconfig`) | build | 389 | 25.9 d | +1.5 |
| **Boot and shutdown both tuned** | port + build | 362 | 27.9 d | **+3.5** |
| **`CONFIG_SPIRAM_MEMTEST` left at its default `y`** (boot 0.7 → 2.5 s) | build | 538 | 18.7 d | **−5.6** |
| `E_scan` 0.50 → 1.50 µAh (panel worse than modelled) | hardware | 589 | 17.1 d | −7.3 |
| `I_awake` 120 → 154 mA (M5Stack's figure is the truth) | hardware | 499 | 20.2 d | −4.2 |
| **Ambient in ESP32 deep sleep, GT911 left scanning (as shipped)** | **#11 / port** | 1270 | 7.9 d | **−16.4** |
| Ambient in ESP32 deep sleep, **GT911 commanded to sleep** | port | 483 | 20.9 d | −3.5 |
| Ambient in **light sleep**, GT911 slept (state retained, no boot) | #11 | 465 | 21.7 d | −2.7 |
| Ambient in light sleep, **GT911 in Green so touch wakes it** | #11 | 1020 | 9.9 d | −14.5 |

Note the last four rows together: **the entire 16-day penalty of "just use deep sleep" is one
missing I²C write.** With the GT911 slept, every rest state is within a few days of every other at
this cadence, and the choice becomes a UX question — which is where it belongs.

**Read in priority order, the levers that matter are:**

1. **Ambient wake cadence** (#11) — worth ±18 to ±47 days. Nothing else is close. **A minute is
   expensive; five minutes is cheap; fifteen minutes is free.**
2. **Rest state: board-off vs ESP32 deep sleep** (#11) — worth 16.4 days, and it is a yes/no rather
   than a dial. It costs touch-to-wake.
3. **Idle animation richness** (#13) — halving run length is worth +8.6 days; doubling it costs
   −8.3 days. Cheaper per unit than cadence, so **if the animation must be visible often, make each
   run shorter rather than the cadence slower** — a 1-second run every 2.5 minutes costs the same as
   a 2-second run every 5 minutes but reads as twice as alive.
4. **Session idle timeout** (#11) — worth ±4 to ±8 days, and it is the lever with the least UX cost,
   because a session the player has walked away from is pure waste. **Timing out at 60–90 s rather
   than 5 minutes is nearly free to the player and worth days.**
5. **Light sleep inside a session** (port) — +4.5 days for one API call, and it is the *only* place
   light sleep pays on this board.
6. **Two `sdkconfig` lines and one experiment** — `CONFIG_SPIRAM_MEMTEST=n` alone is worth
   **+5.6 days** against the ESP-IDF default (§4.1), and boot + shutdown tuning together another
   +3.5. Unconditional wins with no design cost; do them regardless of what #11 decides.
7. **Backup cadence** (#4) — ±0.5 to −2 days. **Not a lever.** Stop treating it as one.

### 8.4 What the headroom can buy

Scenario B leaves **1026 mAh of the 1440 mAh unspent** at nominal. Stated in the currency of §2.3
that is **~8.5 hours of extra awake time per week, ~73 minutes a day.** Concretely, on top of
scenario B, the budget still affords any *one* of:

- **[D]** ~96 000 extra `Animate` frames a week — 13 700/day — **added to the existing runs at
  10.7 µAh each** (8 scans plus 0.2 s of awake time at 5 fps). Spread across scenario B's 2016
  wakes that is ~48 extra frames per run, i.e. **lengthening every idle run from 2 seconds to
  roughly 11**;
- **[D]** ~8.5 hours a week of extra session time — **73 extra minutes a day** of the player
  holding the device;
- **[D]** ~640 extra INSTINCT rounds a week at 30 s and 5 fps — **92 rounds a day**, which is far
  more than the pet's 2–4 care events/day life clock implies anyone would play;
- **[D]** ~29 000 extra `Clear` refreshes, awake time included.

> **The battery is not the constraint on the game slate.** #10's INSTINCT round costs **1.6 mAh**
> — 0.11 % of a week's charge. Two rounds a day for a fortnight costs 45 mAh. If catch-the-falling-
> food turns out to be fun, power is not the thing that stops us shipping it. **The trade the map
> made — "battery down from weeks to one week, to buy idle animation and generous game sessions" —
> was correct, but it bought the wrong thing on the invoice: sessions and games are cheap, and
> *ambient animation cadence* is what actually consumes the week.**


---

## 9. Answers the downstream tickets can build on

Following note 0009 §7.4's convention.

### 9.1 #11 — the ambient/session state machine

This is the ticket that inherits the whole note. Six things fall out as constraints rather than
suggestions:

1. **The state set needs a state note 0009 and #6 both gestured at but did not name: `Off`.** The
   two-mode split (ambient / session) is not enough, because the cheapest ambient is *not a sleep
   state on the ESP32 at all* — it is the board de-energised with a BM8563 countdown armed, and the
   ESP32 coming back through a **cold boot** with no RAM and no PSRAM. That has consequences #11
   must design for: **there is no in-RAM state between ambient wakes.** Everything the pet knows
   must round-trip through NVS (#5) on every wake, and the wake path is a `load → step → render →
   save → power off` pipeline, not an event loop. That is a *good* fit for the pure-`step()`
   architecture ADR-0004 already chose — the core is already a function from persisted state — but
   it needs saying out loud.
2. **Wake cadence is the single most expensive decision in the product.** §8.3: 1/min costs 18 days
   against 1/5 min. #11's line "a sleeping pet at 3am should not wake as often as a hungry one at
   noon" is exactly right and is worth *real* money — a state-dependent cadence is the correct
   design and the model rewards it strongly. As a starting point:

   | Pet state | Suggested cadence | Weekly cost at nominal **[D]** |
   |---|---|---|
   | **Sleep** (pet asleep, quiet hours) | 1 / 30 min, tick only, no repaint | 13.7 mAh |
   | **Ambient**, no pending call | 1 / 5 min, animation run | 297 mAh |
   | **Ambient**, call pending (escalating alert) | 1 / 1 min, animation + `Flip` | 1485 mAh *while it lasts* |
   | **Session** | continuous, light sleep between touches | 0.5–2.0 mAh per session-minute |

   Note the third row is a *rate*, not a budget: an hour of unattended calling costs **8.8 mAh**,
   which is fine. A permanently-calling pet is not. **The escalating alert model should escalate
   the wake cadence and then de-escalate it**, which it naturally does if the buzz phase is bounded.
3. **The rest-state decision and the cadence decision are one decision** (§4.5). The crossover
   between board-off-with-cold-boot and light-sleep-with-state-retained sits at **~3 minutes**
   (1.6–6.1 min across the hardware band). Faster than that, do not power the board off; slower,
   do. Since the recommended animation cadence is 3–5 minutes, **#11 lands right on the crossover
   and either choice is defensible on energy** — so choose on the other axes: board-off has the
   better-attested floor and loses all in-RAM state; light sleep keeps M5GFX's buffers, the NVS
   mount and the BMI270 config alive and wakes in microseconds.
4. **Touch-to-attend is affordable after all, and pickup-to-attend is nearly free.** Two routes,
   both now priced:
   - **Touch**: light sleep with the GT911 in Green (3.3 mA) and `gpio_wakeup_enable(48, LOW)`
     armed. At a 1/5 min animation cadence that is **1020 mAh/week → 9.9 days** **[D]** — it still
     clears one week, though it spends most of the headroom. A **bounded** touch-armed window is
     much better: 20 minutes of Green-mode light sleep after each of 4 sessions costs
     **[D]** `4 × (1/3) h × 4.1 mA × 7 = 38 mAh/week, 2.7 %`. That turns the hardware limitation
     into a UX rule — *"it's awake for a bit after you use it"* — rather than a flat "it never wakes
     to touch".
   - **Pickup**: the BMI270's `INT1` reaches the PMS150G power latch (§3.6), so **motion can
     cold-boot a fully powered-off board**, which in this architecture is the same event as a timed
     ambient wake. At 50 spurious pickups a day that is **14 mAh/week, 1 %** **[D]**. #6 recommended
     "don't promise pickup-to-attend"; that recommendation can now be reversed, and pickup is the
     cheaper of the two gestures by a wide margin.
5. **Session idle timeout is the cheapest real lever** (§8.3, ±4 to ±8 days) and it costs the player
   almost nothing. Recommend 60–90 s of no touch, and *do not* extend it after a game round — the
   post-round moment is exactly where note 0009 rule 6 wants a `Clear` anyway, so ending the session
   there is free in both currencies.
6. **`Clear` costs 18.5 µAh — 0.001 % of the budget.** ADR-0008's "clearing refreshes are events,
   never a timer" is safe from a power objection. Spend them freely at the moments the policy names.

### 9.2 #13 — idle animation vocabulary and its refresh cost

**The ticket's central question gets a different answer than note 0009 gave it, and #13 should use
this one.**

Note 0009 §7.4 compared *scans per minute* and found slow-and-constant (one frame every 10 s,
48 scans/min) cheaper than smooth-and-rare (2 s of 5 fps once a minute, 80 scans/min) by 0.6× —
"nearly cost-neutral, choose on aesthetics". In **power**, per hour of ambient:

| Approach | Optimistic | **Nominal** | Pessimistic |
|---|---|---|---|
| Smooth-and-rare: 2 s @ 5 fps, once a minute | 4.01 mAh/h | **8.84 mAh/h** | 17.52 mAh/h |
| Slow-and-constant: one frame every 10 s | 4.19 mAh/h | **16.08 mAh/h** | 35.43 mAh/h |
| **Ratio** | 1.04× | **1.82×** | 2.02× |

> **Smooth-and-rare wins on power, by up to 2×, where scan-counting predicted it would lose by
> 0.6×.** The mechanism is entirely the boot/shutdown envelope: a lone frame drags ~1.2 s of
> full-power board behind it, and a 2-second run amortises one envelope across ten frames.

Three concrete constraints for the animation vocabulary:

- **Batch, always.** Emit an animation run as a *queued sequence* of repaint effects, not as one
  effect per wake. M5GFX raises the EPD rails once per drained queue, not once per update (§5.6), so
  a batched run pays one rail cycle; ten separate updates pay ten. This composes with ADR-0008
  rule 3 (every run begins and ends on its rest frame) — a batched, rest-framed run is cheap *and*
  DC-neutral, and the two properties want the same code.
- **Run length is a cheaper dial than cadence.** Halving the run (2 s → 1 s) buys +8.6 days; halving
  the cadence (1/5 → 1/10 min) buys +22.4 days but halves how often the pet appears to move at all.
  **A 1-second run every 2.5 minutes and a 2-second run every 5 minutes cost the same and the former
  looks twice as alive.** That is the trade #13 should be exploring on hardware.
- **The budget, stated for #13 to design against: at 1440 mAh usable, with sessions, games, backups
  and floor taking ~115 mAh/week, idle animation may spend up to ~1300 mAh/week.** At 10.7 µAh per
  animation frame delivered inside an existing run **[D]**, that is **~120 000 frames a week, about
  720 an hour** — but only ~2000 *wakes* a week, because the wakes are what cost. **Frames are
  cheap; wakes are expensive.** Design a vocabulary of short bursts, not a slow drift.

### 9.3 #10 — the 5 fps INSTINCT spike

**Power is not a reason to cut this game.** A 30-second round at 5 fps is 150 `Animate` updates =
1200 full-panel scans, and costs **[D]** `30 s × 120 mA + 1200 × 0.5 µAh = 1.6 mAh` — **0.11 % of a
week's charge**. Even at the pessimistic per-scan energy it is 3.1 mAh. The spike's "what does a
round cost in mAh?" question has an answer and the answer is "nothing".

Note 0009 §7.4 told #10 that its real questions were "is it worth the power" and "does a round fit
in 80 seconds of animation budget". **The first of those is now closed: yes, trivially.** The
remaining questions are the ghosting budget and, above all, whether it is fun.

Corollary that #10 should carry into its verdict: if INSTINCT is cut, **the battery target does not
get better in any interesting way**, because games were never the expense. Cutting the *idle
animation* would buy 26 days; cutting the *games* buys 0.3.

### 9.4 #4 — cloud backup cadence

Closed as a power question. §6.2: 5 uploads/day is **10.2 mAh/week (0.7 %)**; 24/day is
**49 mAh/week (3.4 %)**. Note 0004's "if upload frequency turns out to cost real battery, the lever
is the coalescing interval" — it does not, and the interval can be set on whatever grounds #4
prefers. Its other recommendation, **"flush the queue when the device is awake anyway"**, remains
worth doing, because it removes the boot and shutdown envelope from the backup entirely (~15 % of
the line item) and because it is free.

### 9.5 #17 — layout

One addition to note 0009's answer. Region size buys panel wear, CPU and ghosting budget, not
milliseconds — and now, not milliamps either. **[S]** The blit reads the full panel width every line
of every frame regardless of the dirty rect (note 0009 §1.7), so `E_scan` does not scale with region
area any more than time does. Size the pet window for composition and for cumulative panel stress.
The one power-relevant layout consequence is indirect: a layout that lets the pet animate without
also driving the chrome keeps the *staging* loop small, which shortens `t_anim` slightly — worth
tens of milliseconds per run, not a design driver.


---

## 10. The measurement plan

Everything above is model. This is the bench session that turns it into fact, in priority order.
Priority is set by **how much the verdict moves**, not by how uncertain the number is — which is why
the panel current, the most-flagged unknown in note 0009, is fourth rather than first.

Rig: a battery-side current probe capable of resolving 5 µA to 400 mA. A shunt plus a log-scale
current monitor, or a purpose-built low-power analyser, in series with the cell — **not** on the USB
side, because USB present changes the charger's behaviour and defeats the whole measurement.

1. **The floor, in five rest states, with the GT911 sleep command as the independent variable.**
   Board off with a BM8563 alarm armed; ESP32 deep sleep with the GT911 left scanning; ESP32 deep
   sleep with `0x05` written to GT911 register `0x8040`; light sleep with the GT911 slept; light
   sleep with the GT911 in Green and `gpio_wakeup_enable(48, LOW)` armed. Expect
   **~9 µA / ~5 mA / ~0.4 mA / ~0.8 mA / ~4 mA** **[D]**. **This is the measurement that decides
   the architecture**, and the third state is the one nobody has ever published. If it comes in near
   0.4 mA, touch-armed ambient becomes affordable and #11's design space widens considerably; if it
   comes in near 2 mA, the `SY8089AAAC`'s light-load efficiency (§3.3) is the culprit and the answer
   is board-off, unconditionally.
2. **The wake envelope, on a scope.** Trigger on the RTC alarm; capture battery current from
   re-energise to the board dropping again, with GPIO markers at `app_main` entry, first draw,
   `display()` return, and the first `_powerOff` pulse. **This yields `t_boot`, `t_off` and
   `I_awake` in one capture, and those three are 80 % of the budget.** Run it twice — once at
   ESP-IDF defaults, once with `CONFIG_SPIRAM_MEMTEST=n`,
   `CONFIG_BOOTLOADER_SKIP_VALIDATE_ON_POWER_ON=y`, `CONFIG_BOOTLOADER_LOG_LEVEL_NONE=y`,
   `CONFIG_RTC_CLK_CAL_CYCLES=0` and the BMI270 blob upload skipped — and report the delta as a
   single number, because that number is worth ~9 days and costs nothing. Espressif's own
   recommended technique for this is on the
   [deep-sleep stub page](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/deep-sleep-stub.html):
   a GPIO as the wake source, a second GPIO toggled as early as possible, and a logic analyser.
3. **How many `PWROFF_PLUSE` pulses does the PMS150G actually need, and how short can they be?**
   Binary-search from 5 pulses down. **Worth up to +1.8 days for an afternoon's work**, and it is
   pure waste today. Care needed: a board that fails to power off does not announce it — it silently
   falls through to `esp_deep_sleep_start()` and costs 5 mA, so verify with the probe, not the
   screen.
4. **Per-scan energy, `E_scan`, and the zener clamp bias.** Integrate battery charge across an
   isolated `Animate`, `Flip` and `Clear` on a settled board, minus the same window with no update
   issued — the subtraction is what makes it the *incremental* figure §5.1 defines. Then, separately,
   **measure the current into `BST_VIN` with the rails up and no update in flight**: that is the
   zener-clamp bias plus the VCOM divider plus the bleeders, and §5.3 says it could be 22–66 mW of
   pure waste. Capture the bring-up transient too — the datasheet warns of **480 mA `I_COM` inrush**
   **[S]** and an averaging meter will miss it.
5. **`I_awake`, decomposed.** M5Stack's 154 mA versus the datasheet's 81–108 mA is a 45–70 mA gap
   that §5.4 argues is the panel rails. Measure: CPU idle, board on, panel down, radio off; then CPU
   busy; then panel rails up; then radio associated. **Resolves whether §2.3's "80 minutes a day" is
   really 150 minutes**, which is worth ±8 days on its own.
6. **Light-sleep residency inside a session.** Instrument a scripted 90-second session and record
   what fraction of it is spent waiting for a finger. Validates the +4.5-day light-sleep lever and
   tells #11 what a session-minute really costs.
7. **A real WiFi backup, end to end.** Cold associate → NTP → TLS → `b2_upload_file` → disconnect,
   with charge integrated. Confirms the 2–9 s band and settles whether `esp_wifi` fast-connect
   (cached BSSID + channel) and a static lease are worth configuring. Low priority: the whole line
   item is 0.7 %.
8. **A seven-day soak.** Flash scenario B, charge to full, and let it run with the care loop driven
   by a script. The only measurement that catches everything the model forgot — leakage paths,
   charger standby, an unnoticed always-on peripheral, the GT911 waking spuriously. **This is the
   one that actually answers issue #12**; everything above just tells us where to look when it
   disagrees.

Three cheap checks to fold in while the probe is attached:

- **Does the EPD rail genuinely collapse when `BST_EN` (G46) is low?** Probe `EPD_VPOS` with the
  board awake and the panel idle. §5.6 argues from the schematic that it must — the rails are
  AC-coupled off the boost switch node with no DC path and 120 K bleeders — but note 0009 §6.2 asked
  for the confirmation and it is a Waveshare-documented damage mode, not only a power question.
- **Confirm the BMI270 `INT1` → `E_TRG` → PMS150G path** (§3.6) by configuring an any-motion
  interrupt on a powered-off board and seeing whether shaking it boots the device. This is a
  five-minute test that either unlocks pickup-to-attend for #11 or closes it for good.
- **Confirm the BMI270 survives a board power-off cycle with its configuration and step count
  intact.** `IMU_VDD` comes from an always-on LDO with no enable pin (§3.3), so it should — which is
  what makes §4.2's "skip the 8 KB blob upload" optimisation safe. Read `INTERNAL_STATUS` and
  `SC_OUT_*` before and after a `timerSleep()` cycle.
- **Does the BM8563's countdown timer flag survive the PMS150G power cycle?**
  [ADR-0009](../adr/0009-pickup-listens-touch-attends.md) needs the device to tell a **pickup** from
  a timed wake, since both arrive as the same power-on reset. Comparing now against the persisted
  **wake deadline** always works and is the primary route; reading the PCF8563-compatible timer flag
  would be more direct, and M5Unified already clears it, but nothing establishes that it survives the
  rails dropping. Arm a countdown, let it fire, and read the flag on the far side of the cold boot.
- **Is there an ambient light sensor on this board?** No part in the schematic BOM or the product
  documentation suggests one, and note 0009 established the same austerity for temperature.
  [ADR-0010](../adr/0010-alerts-escalate-animation-de-escalates.md) leans on its absence: quiet hours
  suppress the idle animation on the argument that e-ink emits no light, so a pet moving in a dark
  room is unobserved and the device cannot know better. If a sensor does exist, that becomes a
  cheaper and more accurate trigger than the clock.

---

## What remains uncertain

Flagged plainly. Anything load-bearing that could not be traced to a first-party source is here.

### Numbers that are second-hand, not first-party

1. **The 5.1 mA ESP32-deep-sleep figure is a community measurement**, from one contributor on
   M5Stack's own forum, corroborated by a second user's "about 5 ma" inferred from a 5–7 %/day
   discharge rate **[2]**. It is the second-largest term in the note after the awake-second rate,
   and **it is used to reject an entire architecture**. M5Stack publish no deep-sleep figure. It
   must be re-measured (§10, measurement 1). Its direction is not really in doubt — 9.28 µA
   board-off versus milliamps board-on is M5Stack's own framing — but its magnitude is.
2. **M5Stack's three published figures have no stated method.** "Operating mode: DC4.2V/154.02 mA
   (main power on)" does not say what the CPU, the radio or the panel were doing. The two-decimal
   precision suggests a real bench reading of a real unit, and the internal consistency of the
   9.28/949.58 pair against the BMI270 datasheet (§1.2) is strong evidence they mean what they say —
   but 154.02 mA is used here as the pessimistic `I_awake` and it may be describing a state we never
   enter.

### Numbers derived rather than measured

3. **`E_scan` is bounded, not measured.** The *panel* half is first-party — ED047TC1 §6-2 gives
   every rail current and a 170 mW typ / 950 mW max panel power, with stated test patterns, and the
   per-rail sum reconciles with the headline figure to within 1 % (§5.4) **[S]**. Three things are
   still inference: (a) **the rail-generation efficiency**, taken from the TPS65185's curves at
   `V_IN` = 3.5 V because the MT3608 datasheet publishes nothing below ~100 mA at any input voltage
   — a different topology, and it multiplies the whole panel term; (b) **the zener-clamp bias
   current on `VGH`/`VGL`**, which is pure inference (§5.3) and could plausibly be 1.4–4× the
   panel's own gate-rail draw; (c) **where between typ and max a small-region `Animate` sits** — the
   datasheet's typ condition is a full-screen 4-grey transition and ours drives ~5 % of the screen,
   but the driver ICs' bias current is not obviously area-proportional. §7.3 shows the verdict
   survives the whole range including the 950 mW worst case, which is why this sits at #4 in the
   measurement plan rather than #1.
4. **`t_boot` is a sum of estimates, not a measurement, and Espressif publish no ESP32-S3 wake
   latency at all.** Five of its components now have first-party backing (the ~280 µs hardware init,
   the ~6.1 ms ROM log, the ~1 s/4 MB PSRAM memtest, the NVS mount, and the BMI270 blob's I²C
   duration); the app-image verification time is described only as *"a significant portion of the
   boot time"* and *"depends on the binary size and the flash settings"*. **Beware the numbers that
   look like S3 wake latencies and are not:** the "12734 µs wakeup cost" on Espressif's own ESP32-S3
   deep-sleep-stub page is pasted from an **ESP32-C3** boot log (the banner reads
   `ESP-ROM:esp32c3-api1-20210207`), and the 6505 µs in the matching example README is an
   **ESP32-C6**. Neither is ours, and this note does not use either. **[U]**
5. **`I_awake` at 120 mA nominal is a midpoint between two figures measured under different, partly
   unknown, conditions.** It multiplies almost every line item in the note.
6. **The light-sleep and GT911-slept deep-sleep floors (~0.79 mA and ~0.42 mA) are arithmetic**,
   summed from ten datasheet typicals plus two exact resistive terms. **Nobody has published a
   measurement of either configuration on this board**, and §4.5's whole crossover analysis rests on
   the first of them. The largest single term inside them is item 7 below.
7. **The `SY8089AAAC`'s real no-load input current is unbounded by its datasheet.** Silergy's
   document mentions PFM, pulse-skipping and power-save **nowhere** — only *"proprietary PWM
   control"* — and its 55 µA Iq figure is specified with the feedback node forced to 105 % of
   `V_REF`, i.e. deliberately *not switching*. Its efficiency curves start at 10 mA. At a
   ~400 µA load the rail could plausibly be 30–50 % efficient, which would multiply every
   `SOC_VDD` term in the sleep floor by 2–3× and could move the §4.5 crossover from 3 minutes to
   1 minute. **This is the single biggest unquantified number left in the note.** **[U]**

### Hardware questions this note could not settle

8. **No datasheet for the actual 1800 mAh cell exists in public.** M5Stack specify capacity, nominal
   voltage and the charger part, and nothing else — no manufacturer, no part number, no discharge
   curve, no self-discharge spec, and the battery header is 2-pin so the docs cannot even say whether
   the pack carries a protection circuit. Every cell figure in §2.1 comes from **representative**
   1800 mAh-class pouch datasheets (EEMB LP505464, Jauch LP103450JH, LiPol, PKCELL, GMB) and the
   discharge curves from a VARTA handbook for a *different* cell. The self-discharge line is 0.9 % of
   the budget so that error is small; the derating sets the *denominator* of every percentage in the
   note, which is why §8.1 reports the verdict against three capacity cases rather than one. **[U]**
9. **The self-discharge rate is a guess bracketed by two very different numbers.** Typical LiPo
   behaviour is 2–3 %/month, but the *specified* IEC 28-day floors on the representative cells are
   10 %/month (EEMB, *"Discharge Time ≥ 4.5 h"* against a 5 h nominal) and 20 %/month (PKCELL,
   *"residual capacity is above 80 %"*) **[S]**. Over a one-week window that is 0.7 % versus 4.7 % of
   the pack — immaterial here, but it would dominate a shelf-life or long-storage calculation, and it
   is the number to reach for if anyone asks how long a boxed device holds charge. **[U]**
10. **The pack's protection-circuit quiescent draw is estimated at ~7 µA**, from a module-level
    figure (GMB-ML131: *"Current consume in normal operation — 7.0 µA Type, 12.0 µA Max"*) rather
    than the bare protection-IC number (DW01A 3.0 µA typ, ABLIC S-8261 3.5 µA typ) **[S]** — the
    bias resistors roughly double the IC figure. At the board-off floor that is comparable to the
    entire rest of the board. It is almost certainly already inside M5Stack's 9.28 µA, since they
    measured at 4.2 V at the battery connector, but "almost certainly" is doing work there.
    Irrelevant at 0.1 % of the budget; recorded because it is the sort of thing that surprises
    people. **[U]**
11. **Several part datasheets are specified at voltages this board does not run them at**, and every
    one of these extrapolations is in the pessimistic direction: the BM8563 is specified at 2.0/3.0/
    5.0 V and runs at 3.0–4.2 V here; the BMI270 at `VDD` = 1.8 V and runs at 3.3 V; the GT911 at
    `AVDD` = 2.8 V and runs at 3.3 V; the PMS150G at 3.3 V and runs at 3.0–4.2 V. The sleep-floor
    sums in §3.3 will therefore read low. **[U]**
12. **The PMS150G's latch behaviour is a black box.** It is a *custom pre-programmed* Padauk part
    (the schematic annotates it `定制带程序芯片`) with no published firmware. Every component of the
    latch network is identified — `Q1A`/`Q1B` 2N7002DW, `Q2` YJL2101W, `D1`/`D2`/`D3`/`D15` 1N4148WT,
    `R7`/`R11` 100 K, `R8`/`R10` 5.1 K, `C30` 1 µF — but the state machine that decides when a pulse
    on `PWROFF_PLUSE` actually drops `MPWR_EN` is not derivable from the netlist. This is why §4.3's
    "how few pulses do we need" is a bench question rather than a reading question. **[U]**
13. **The charge current does not reconcile.** `R15` = 5.1 K on the LGS4056H's `PROG` pin, and the
    part's own `900/R_PROG` formula gives ~176 mA, against M5Stack's published *"DC 5V@331.5mA"*
    **[S]**. Either the docs figure is USB *input* current, or the formula constant differs by
    silicon revision. Immaterial to the discharge budget; it would matter to any "time to full"
    display. **[U]**
14. **The ESP-IDF brownout detector's default threshold** on `esp32s3` is `ESP_BROWNOUT_DET_LVL_SEL`
    `_SEL_7` ≈ **2.44 V** **[S]** — but Espressif's own in-source comment says *"the voltage levels
    here are estimates, more work needs to be done to figure out the exact voltages"* and *"there may
    be some variation of brownout voltage level between each ESP32-S3 chip"*. So the exact voltage
    at which the chip resets is a per-unit unknown. It sits well below the SY8089's useful range
    (§2.2), so it does not bind the usable capacity — but a brownout reset mid-save is exactly the
    torn write #5 built the A/B slots to survive, and it is the reason the firmware cutoff should be
    a chosen 3.4 V rather than "whatever happens first". **[U]**

### Things the model deliberately does not include

15. **Charging.** Time on the charger is time not on the budget, and the map already says the pet
    presents differently when plugged in. The LGS4056H charges at 331.5 mA **[S]**, so a full
    recharge is ~5.4 h **[D]** — worth knowing for the UX, irrelevant to the discharge model.
16. **Temperature.** Note 0009 established there is no sensor and no compensation. Cold slows the
    panel's waveform (up to +21 % frames at 15–18 °C) and reduces LiPo capacity, and both effects
    push the same way. A cold room is worth maybe −20 % on the whole budget and is not modelled.
    **[U]**
17. **The buzzer, the system LED on G0, and the microSD slot.** All small, all bounded by their duty
    cycle, none modelled. The LED is the one to watch: M5Unified drives G0 as a PWM output
    (note 0006), and a status LED left on is tens of milliamps — **[D]** 20 mA continuous would be
    **3360 mAh/week**, more than double the battery. Not a modelling gap so much as a thing to
    assert is off.
18. **PSRAM active current.** The ESP32-S3 datasheet's PSRAM table (5-12) contains **no current
    figures at all** — only voltage and clock, "sourced from the memory vendor datasheet" **[S]**.
    The only PSRAM numbers Espressif publish anywhere are the *light-sleep* adders (140 µA for our
    8 MB octal part at 3.3 V). The blit streams ~1 MB per frame out of that PSRAM (note 0009 §1.7),
    so it is inside the `I_awake` band by construction but is not separately known. **[U]**
19. **Two R8-specific sleep constraints were found but not costed.**
    `CONFIG_ESP_SLEEP_POWER_DOWN_FLASH` *"can only be enabled if there is no SPIRAM configured"*
    because *"due to the shared power pins between flash and PSRAM, cutting power to PSRAM would
    result in data loss"* — so on this part it is unavailable in light sleep. And
    `CONFIG_ESP_SLEEP_PSRAM_LEAKAGE_WORKAROUND` is enabled by default with `CONFIG_SPIRAM` and
    *"will increase the sleep current about 10 uA"* **[S]**. Both are inside the light-sleep
    estimate's error bars, neither is broken out. **[U]**
20. **Every low-power figure in the ESP32-S3 datasheet is for the *non-PSRAM* part.** §5.6.2's
    preamble says so explicitly: *"Since ESP32-S3R2, … **ESP32-S3R8**, … are embedded with PSRAM,
    their current consumption **might be higher**"* **[S]**. The 7 µA deep-sleep figure is therefore
    not strictly ours, and Espressif publish no deep-sleep PSRAM adder (only a light-sleep one).
    Immaterial here, because the board dominates by two orders of magnitude — but it is the kind of
    footnote that matters on a board where it would not. **[U]**

### Carried forward, unchanged

21. **Nothing has been built against ESP-IDF v6**, and nothing in this repository has run on the
    hardware. Note 0002's uncertainty 2 and note 0009's uncertainty 5 remain open, and this note
    adds a third instance of the same sentence. It is still the first thing to do.
