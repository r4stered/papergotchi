# Refresh strategy: waveform modes, framerate, and the ghosting budget

Research for [issue #9](https://github.com/r4stered/papergotchi/issues/9). Follows #2, which chose
M5GFX. Feeds the mode state machine (#11), the power budget (#12), the idle animation vocabulary
(#13), the layout (#17) and the 5 fps game spike (#10).

**Nothing here was measured.** The ticket asks for timings on this board and no board was attached
to this session. What this note delivers instead is the cost *model* derived from source and from
the panel's own datasheet, a refresh policy that follows from it, and a measurement plan short
enough to run in an afternoon. Every claim is tagged:

- **[S]** read directly from source or a specification
- **[D]** arithmetic derived from source constants — the arithmetic is shown
- **[U]** unverified, needs hardware

Source was read at pinned commits, and every file:line citation is a permalink at those commits:

- **M5GFX** [`729297d`](https://github.com/m5stack/M5GFX/tree/729297d6e3d657ddc1ec5189bac2f2ea68828085) (v0.2.26)
- **epdiy** [`7c30780`](https://github.com/vroland/epdiy/tree/7c3078092479d9eabcf1dadcd928b5f3e446284e) (v2.1.3)
- **ESP-IDF** [`0c1e6bd`](https://github.com/espressif/esp-idf/tree/0c1e6bd965302d25b1f3d49ba36425bcdaa55cb3) (`release/v6.0`, 2026-07-31)
- **M5Unified** [`4fb4447`](https://github.com/m5stack/M5Unified/tree/4fb444784c85791e0b0207701392b42be234b2e7)

---

## The six findings that change the design

1. **Update time does not scale with region height.** Confirmed in both libraries, at source, with
   no config knob anywhere. The premise the animation design rested on is gone. #2 flagged this;
   this note closes it.
2. **`CONFIG_FREERTOS_HZ` is the single biggest lever we have, and ESP-IDF's default is the wrong
   one.** At the stock 100 Hz every waveform frame costs 20 ms. At 1000 Hz it costs ~11–14 ms.
   Every refresh on the device roughly halves for one line of `sdkconfig.defaults`.
3. **The panel is rated at 85 Hz maximum.** We found the real ED047TC1 datasheet. We are not trying
   to go faster than the hardware — we are trying to stop undershooting the FreeRTOS tick.
4. **The fast modes are exactly DC-balanced over a black↔white cycle; the quality mode is not.**
   So animation is electrically cheap and a periodic "cleanup" full refresh is electrically
   *expensive*. This inverts the obvious policy — and no vendor publishes per-mode DC balance for
   any panel, so our LUT decode in §3.2 is the only source that exists for these waveforms.
5. **The ghosting budget should be accumulated driven *area*, and there is shipping code that does
   exactly this.** The Rockchip EBC driver budgets `Σ(update area)` against `20 × screen_area`.
   For a pet window at ~5 % of the screen that is **~400 fast updates between clearing refreshes**,
   which is a real and quite generous number.
6. **There is no temperature sensor on this board at all.** Not omitted from the driver — absent
   from the schematic. Temperature compensation is not a software decision.

---

## 1. The cost model

### 1.1 Bytes per line — 248, confirmed [S][D]

```c
const size_t write_len = (memory_w / 4) + me->_config_detail.line_padding;
```
— [`Panel_EPD.cpp:901`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L901)

`960 / 4 + 8 = 248`. The `/4` is 2 bits per pixel, 4 pixels per byte — verifiable rather than
assumed: `blit_dmabuf` runs `(960 + 15) >> 4 = 60` iterations
([`Panel_EPD.cpp:1046`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L1046)),
each emitting one `uint32` from 8 source entries
([`Panel_EPD.cpp:649-651`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L649-L651)),
each entry covering 2 pixels
([`Panel_EPD.cpp:279`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L279)):
60 × 4 = 240 B = 960 px. The trailing 8 padding bytes are zeroed once at init and never rewritten
([`Panel_EPD.cpp:252-253`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L252-L253))
— 32 no-op pixel slots per line.

epdiy computes the same thing independently: `lcd.line_bytes = display_width / 4`
([`lcd_driver.c:446-447`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/output_lcd/lcd_driver.c#L446-L447)).

16 MHz is the **byte** clock. `pclk_prescale = resolution_hz / pclk_hz` where
`resolution_hz = src_clk_hz / 2`
([`esp_lcd_panel_io_i80.c:324`](https://github.com/espressif/esp-idf/blob/0c1e6bd965302d25b1f3d49ba36425bcdaa55cb3/components/esp_lcd/i80/esp_lcd_panel_io_i80.c#L324),
[`:637`](https://github.com/espressif/esp-idf/blob/0c1e6bd965302d25b1f3d49ba36425bcdaa55cb3/components/esp_lcd/i80/esp_lcd_panel_io_i80.c#L637),
[`esp_lcd_common.h:28`](https://github.com/espressif/esp-idf/blob/0c1e6bd965302d25b1f3d49ba36425bcdaa55cb3/components/esp_lcd/priv_include/esp_lcd_common.h#L28)):
**[D]** `160/2 = 80`, `80/16 = 5` exactly, so pclk = 16.000 MHz, one byte per strobe.

### 1.2 The 4 µs nobody costed [S]

The existing estimate in note 0002 — `540 × 15.5 µs ≈ 8.4 ms/frame` — is the **data phase only**.
ESP-IDF's i80 driver busy-waits inside every transaction:

```c
// delay 4us is sufficient for DMA to pass data to LCD FIFO
esp_rom_delay_us(4);
lcd_ll_start(bus->hal.dev);
```
— [`esp_lcd_panel_io_i80.c:752-774`](https://github.com/espressif/esp-idf/blob/0c1e6bd965302d25b1f3d49ba36425bcdaa55cb3/components/esp_lcd/i80/esp_lcd_panel_io_i80.c#L752-L774)

M5GFX issues **one transaction per scan line**
([`Bus_EPD.cpp:101-110`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Bus_EPD.cpp#L101-L110)),
and it cannot queue ahead: CKV is dropped in the trans-done ISR
([`Bus_EPD.cpp:37-44`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Bus_EPD.cpp#L37-L44)),
so a queued second transaction would collapse the CKV low time to zero. The 4 µs cannot be
amortised.

| Per-line component | Value | Tag |
|---|---|---|
| Data phase, 248 B ÷ 16 MHz | 15.500 µs | **[D]** |
| 1 front + 1 back blank cycle | ≤ 0.125 µs | **[S]** |
| `esp_rom_delay_us(4)` | 4.000 µs | **[S]** |
| 2 × ISR dispatch, queue ops, DMA remount | unquantified | **[U]** order 3–10 µs |
| **Floor** | **19.625 µs** | **[D]** |

Plus a 23 µs SPV/CKV frame preamble
([`Bus_EPD.cpp:48-64`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Bus_EPD.cpp#L48-L64)):

- **Floor [D]:** `23 µs + 540 × 19.625 µs = 10.62 ms`
- **Realistic band [U]:** `11.9 … 16.2 ms`

> **The 8.4 ms/frame figure recorded in note 0002 is a lower bound on the data phase, not a frame
> time.** The true floor is ≥ 10.6 ms — which happens to sit just above the default tick period,
> with expensive consequences.

### 1.3 `vTaskDelay(1)` is real, and it is additive [S]

```c
for (uint_fast16_t y = 0; y < mh; y++) { ... bus->writeScanLine(...); }

vTaskDelay(1);
bus->scanlineDone();
```
— [`Panel_EPD.cpp:1048-1065`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L1048-L1065)

It is **not** overlapped with DMA in any useful sense: when it is reached, exactly one transaction
is in flight — the last line's, worth 15.5 µs, about 0.15 % of the frame. The scan loop restarts
immediately after the tick boundary, so it phase-locks to the tick:

**[D]** `frame_period = ceil(scan_time / tick_period) × tick_period`

The default tick rate is 100 Hz, verified in Kconfig, with **no esp32s3 override**:

```
config FREERTOS_HZ
    int "configTICK_RATE_HZ"
    range 1 1000
    default 100
```
— [`components/freertos/Kconfig:33-40`](https://github.com/espressif/esp-idf/blob/0c1e6bd965302d25b1f3d49ba36425bcdaa55cb3/components/freertos/Kconfig#L33-L40)

By contrast `arduino-esp32` *forces* 1000 Hz — it is a hard `FATAL_ERROR` in its
[`CMakeLists.txt:426-430`](https://github.com/espressif/arduino-esp32/blob/master/CMakeLists.txt#L426-L430),
and `CONFIG_FREERTOS_HZ=1000` is set in
[`defconfig.common:45`](https://github.com/espressif/esp32-arduino-lib-builder/blob/master/configs/defconfig.common#L45).

> **Every anecdotal PaperS3 refresh figure published online was measured under Arduino, at
> 1000 Hz. Our pure-IDF build defaults to 100 Hz and will be about twice as slow.** This alone
> would have produced a baffling 2× shortfall against "the numbers everyone quotes", and is very
> likely part of where the map's ~200 ms folklore came from.

**Do not simply delete the `vTaskDelay(1)`.** The EPD task runs at priority 2, pinned
([`Panel_EPD.cpp:287-293`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L287-L293)),
and its only other wait is a `taskYIELD()` that will not yield to a lower-priority task. This is
the sole preemption point in the scan loop; removing it starves the idle task on that core.
Raising the tick rate keeps the property at a tenth of the cost.

### 1.4 The panel's own limit — 85 Hz [S]

From the E Ink Holdings *Technical Specification, Model No. ED047TC1*, doc P-511-710(V:1) Rev 1.0,
2015-03-19, §6-3, obtained as a byte-identical PDF from two independent mirrors
([LilyGo-EPD47](https://raw.githubusercontent.com/Xinyuan-LilyGO/LilyGo-EPD47/esp32s3/datasheet/ED047TC1.pdf),
[T5S3-4.7-PRO](https://raw.githubusercontent.com/Xinyuan-LilyGO/T5S3-4.7-e-paper-PRO/H752-01/hardware/ED047TC1.pdf)):

> "The module ED047TC1 is applied at a **maximum screen refresh rate of 85Hz**."

**[D]** 85 Hz ⇒ minimum frame period **11.76 ms**. Other AC limits from the same section: `XCL`
cycle time ≥ 16.67 ns (60 MHz ceiling — the bus is nowhere near it), data setup/hold 8 ns, `XLE`
high pulse ≥ 300 ns, output settling ≤ 20 µs.

This 85 Hz is the same figure E Ink's waveform specification uses as the basis for all its published
mode timings (§4.1), which is why those millisecond numbers transfer to this panel directly.

Consequences:

- M5GFX at 16 MHz, `HZ=100` → 20 ms period = **50 Hz**. In spec, 1.7× slower than the panel allows.
- M5GFX at 16 MHz, `HZ=1000` → 11–16 ms = **63–91 Hz**. Straddles the ceiling. **[U] measure.**
- M5GFX at 20 MHz would be **[D]** `23 µs + 540 × 16.525 µs = 8.95 ms` = 112 Hz — **out of spec.
  The pixel clock is not the lever; do not raise it.**
- epdiy would run this panel at **[D]** `544 × 13 µs = 7.07 ms` = **141 Hz, 66 % above rated**, with
  no tick quantisation to hold it back. **[U]** Whether that contributes to the panel-line failures
  in §6 is a hypothesis, not a finding — but it is a striking coincidence.

Two other electrical facts from the same datasheet worth carrying: the source rails are specified
for DC symmetry — **`VASM = VPOS + VNEG`, min −800 mV / typ 0 / max +800 mV** — so DC balance is a
*hardware-spec'd* quantity, not only a waveform convention. And power-down is sequenced: "Begin to
turn off VGL power after VNEG and VPOS are completely or almost discharged", `Ted` min 0.5 s.

### 1.5 Frames per mode — counted from the arrays [S][D]

The LUTs are at
[`Panel_EPD.cpp:82-156`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L82-L156).
Every table is padded with all-no-op `~0u` rows and closed by a `0u` terminator, and **every step,
padding and terminator included, costs a full-panel scan.**

| Mode | Driving rows | `~0u` pad | `0u` term | `lut_*_step` | + eraser (4) | + trailing idle scan | **Total scans** |
|---|---|---|---|---|---|---|---|
| `epd_quality` | 15 | 16 | 1 | 32 | ✔ | ✔ | **37** |
| `epd_text` | 12 | 19 | 1 | 32 | ✔ | ✔ | **37** |
| `epd_fast` | 8 | 1 | 1 | 10 | ✘ | ✔ | **11** |
| `epd_fastest` | 5 | 1 | 1 | 7 | ✘ | ✔ | **8** |

> **`epd_text` and `epd_quality` take identical wall-clock time.** Both are padded to exactly 32
> steps, deliberately, in
> [`031dbe2`](https://github.com/m5stack/M5GFX/commit/031dbe2dd26a672116327ae3713abbf71ffa550d)
> ("tweak for PaperS3 refresh", 2025-09-30). `epd_text` is *gentler*, not faster. Note 0002's
> four-mode table implied a quality/text speed tradeoff that does not exist.

The trailing idle scan exists because `remain` can only be evaluated after a full pass
([`Panel_EPD.cpp:1047-1052`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L1047-L1052)).

### 1.6 The timing tables [D]

One isolated update, `scans × frame_period`. Add 0–10 ms for `display()`'s own `vTaskDelay(1)` and
~10.2 ms for the idle→powered transition at `HZ=100`
([`Bus_EPD.cpp:88`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Bus_EPD.cpp#L88)).

| Mode | Scans | `HZ=100` (20 ms) | `HZ=1000` (11 ms floor) | `HZ=1000` (14 ms **[U]**) | E Ink reference @85 Hz |
|---|---|---|---|---|---|
| `epd_fastest` | 8 | **160 ms** | 88 ms | 112 ms | 94 ms |
| `epd_fast` | 11 | **220 ms** | 121 ms | 154 ms | 129 ms (cf. A2 = 120 ms) |
| `epd_text` | 37 | **740 ms** | 407 ms | 518 ms | 435 ms |
| `epd_quality` | 37 | **740 ms** | 407 ms | 518 ms | 435 ms (cf. GC16 = 450 ms) |

The right-hand column is `scans ÷ 85 Hz` — the time the panel is *designed* for, and it lines up
with E Ink's published mode times (§4.1). **At `HZ=1000` M5GFX lands within ~15 % of E Ink's own
design point; at `HZ=100` it is ~1.7× slow.**

Sustained animation rate is bounded by waveform completion, `1 / (lut_step × frame_period)`:

| Mode | LUT steps | `HZ=100`, 20 ms | `HZ=1000`, 11 ms | `HZ=1000`, 14 ms |
|---|---|---|---|---|
| `epd_fastest` | 7 | **7.1 fps** | 13.0 fps | 10.2 fps |
| `epd_fast` | 10 | **5.0 fps** | 9.1 fps | 7.1 fps |

**So the map's "~5 fps full-screen 1-bit" is roughly right by accident.** It is right at the stock
tick rate for `epd_fast`, and comfortably beaten by `epd_fastest` at `HZ=1000`.

### 1.7 A second bottleneck nobody has costed **[U]**

`blit_dmabuf` reads `_step_framebuf` from PSRAM for the **full panel width, every line, every
frame**, regardless of the dirty rect: **[D]** 1 920 B/line × 540 = **1 036 800 B read per frame**.
To hold a 19.6 µs line budget that needs **[D]** `1920 / 19.6 µs ≈ 98 MB/s` sustained from octal
PSRAM against a theoretical 160 MB/s ceiling. The DMA buffers themselves are internal
(`MALLOC_CAP_DMA`,
[`Panel_EPD.cpp:239-240`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L239-L240))
so only the CPU touches PSRAM, but **the blit may co-determine the frame time and must be
measured.**

Related: M5Stack's spec table says "PSRAM 8MB **Quad**", which contradicts the `ESP32-S3R8` part
marking and M5GFX's hard OPI requirement
([`M5GFX.cpp:2096-2100`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/M5GFX.cpp#L2096-L2100)).
Treat the docs row as an error — but if it were true, quad PSRAM's ~40 MB/s would make the blit the
dominant cost by 2.5×.

### 1.8 Memory — correcting note 0002 [S][D]

| Allocation | Size |
|---|---|
| `_step_framebuf` (PSRAM) | 1 036 800 B |
| `_buf` 4 bpp framebuffer (PSRAM) | 259 200 B |
| `_dma_bufs[2]` (internal DMA) | 496 B |
| `_lut_2pixel` (internal DMA) | `85 × 256 × 2 = 43 520 B` |

**Total 1.24 MiB PSRAM + ~44 KB internal DMA RAM.** Note 0002 said "~1 MB DMA LUT in internal RAM";
that is **wrong by ~24×** — `lut_total_step` is 85, not thousands. Only half the LUT allocation is
even used (a `sizeof(uint16_t)` in the malloc against a `uint8_t*` fill loop,
[`Panel_EPD.cpp:228`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L228),
[`:258-284`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L258-L284)).

---

## 2. Does update time scale with region height? No.

### 2.1 M5GFX [S]

```c
const size_t mh = (me->_cfg.memory_height + magni_h - 1) / magni_h;
```
— [`Panel_EPD.cpp:903-906`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L903-L906)

`memory_height == panel_height == 540` ⇒ `magni_h = 1` ⇒ **`mh = 540`**. These lines sit *outside*
the `for(;;)` loop — they run once at task entry, so `mh` cannot even change at runtime. The scan
loop reads only `mh`
([`Panel_EPD.cpp:1046-1061`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L1046-L1061));
**`new_data.x/y/w/h` appear nowhere in it.** They are used only in the staging loop.

Exhaustive config check — there is no scan-range knob: `Panel_EPD::config_detail_t` holds four LUT
pointers, four step counts, `line_padding`, `task_priority`, `task_pinned_core`
([`Panel_EPD.hpp:42-58`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.hpp#L42-L58));
`Panel_Device::config_t` has no vertical-range field; the private members are task/queue handles and
buffers. `_lut_remain_table` is written but never read — dead code, not a hidden lever.

The one escape hatch is init-time and total: setting `memory_height = panel_height = N` yields a
coherent 960×N panel with everything scaling together — **[D]** N = 270 ⇒ 5.3 ms scan. But the
display is then permanently 960×270, top-of-panel anchored with no offset, and in the landscape
orientation apps use, panel row 0 is the *bottom* of the image. **[U]** Whether the gate driver
tolerates a truncated scan cleanly is unverified.

### 2.2 epdiy — same answer, different code [S]

`lines_total = rounded_display_height()` = 544
([`render.c:170`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/render.c#L170),
[`:87-89`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/render.c#L87-L89)),
never a function of the crop. Lines outside `[min_y, max_y)` are pushed zero-filled but **still
clocked**
([`render_lcd.c:199-226`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/output_lcd/render_lcd.c#L199-L226)).
Horizontal cropping is asserted away outright
([`render_lcd.c:194`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/output_lcd/render_lcd.c#L194)).

And the high-level API forecloses even vertical selection — `epd_hl_update_area` **overwrites the
crop rect with the full screen** before drawing:

```c
diff_area.x = 0; diff_area.y = 0;
diff_area.width  = epd_width();
diff_area.height = epd_height();
```
— [`highlevel.c:269-284`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/highlevel.c#L269-L284)

### 2.3 What *does* shrink with region size [D]

| Cost | Scales? | Detail |
|---|---|---|
| Panel scan / frame time | **No** | 540 lines always |
| Blit per frame | **No** | ~1 MB PSRAM read per frame always |
| Staging loop | **Yes, linearly** | Full screen `540 × 240 = 129 600` iterations ≈ 2.3 MB PSRAM traffic, **[U]** order 20–30 ms. A 200×100 box is `100 × 50 = 5 000` — **1/26th** |
| Panel electrical stress | **Yes** | Only staged pixels are ever driven |
| Ghosting footprint | **Yes** | Confined to staged pixels — and this is what buys the budget in §4.4 |

> **To make a refresh cheap in *time*, pick a shorter mode. To make it cheap in *panel wear*, *CPU*
> and *ghosting budget*, pick a smaller box. These are different levers and #17 must not confuse
> them.**

---

## 3. The waveform modes, decoded

### 3.1 The encoding [S]

```c
// 値の意味は 0 == end of data / 1 == to black / 2 == to white / 3 == no operation
```
— [`Panel_EPD.cpp:76-81`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L76-L81)

Each `uint32_t` is one frame; each of 16 two-bit fields is the action for one grey **column**
(0 = black, 15 = white). Independently corroborated by epdiy, which names the same bit patterns:
`CLEAR_BYTE 0b10101010` "make lighter", `DARK_BYTE 0b01010101` "make darker"
([`lut.h:6-9`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/output_common/lut.h#L6-L9)),
`0x00` = no-op. Two independent codebases agree: `01` = darken, `10` = lighten, `00`/`11` = no drive.

### 3.2 DC balance, summed from the arrays [D]

**No vendor publishes per-mode DC balance for any panel** — we looked hard, and it is stated
nowhere in E Ink's waveform specification, the ED047TC1 datasheet, or any controller datasheet. The
general principle is documented (§4.2); the per-mode attribution is not. **So the table below is the
only source of this information that exists for these waveforms, and it was obtained by decoding the
arrays.**

"Net" is (#white − #black) in frame-units. Frame-count sums are a valid proxy for net charge here
*only* because every frame has identical duration and fixed ±V; they ignore state-dependent physics.
Treat as relative indicators.

The mode LUTs are indexed by **target** grey; the eraser by **current** grey. So one transition
costs `eraser(from) + mode(to)`.

**Black↔white cycle — the animation case:**

| Mode | white→black | black→white | **cycle total** |
|---|---|---|---|
| `epd_fastest` | −3 (no eraser) | +3 | **0 — exactly balanced** |
| `epd_fast` | −4 (no eraser) | +4 | **0 — exactly balanced** |
| `epd_text` | −3 | +3 | **0 — exactly balanced** |
| `epd_quality` | −9 | +6 | **−3 — imbalanced** |

**Refresh-in-place, `f(g) = eraser(g) + mode(g)` — what a "cleanup" pass costs:**

| grey | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `f_quality` | −5 | −4 | −2 | −1 | −2 | −2 | −2 | +2 | +2 | +4 | +1 | 0 | −2 | 0 | +1 | **+2** |
| `f_text` | +1 | +4 | +4 | +3 | +3 | +4 | +2 | +3 | +4 | +5 | +6 | −2 | +3 | +4 | +3 | (skipped) |

> **This is the finding that sets the policy.** Animating a silhouette black↔white in `epd_fastest`
> is DC-neutral over every complete cycle, with a bounded ±3 residual if it stops mid-cycle. But
> **every `epd_quality` pass over static content pushes white pixels +2 and black pixels −5.** A
> periodic full refresh "to clean up ghosting" is itself a cumulative DC source. Full refreshes are
> for *fixing* ghosting, never for preventing it on a timer.

In the fast modes only columns 0 and 15 carry non-no-op entries, consistent with the drawing side,
which binarises when the mode is fast: `readbuf[i] = (sum + (b<<4)) < 248 ? 0 : 0xF;`
([`Panel_EPD.cpp:491-496`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L491-L496)).
This mirrors E Ink's own restriction of A2 to transitions between saturated states (§4.1) and is a
good sign the LUTs were derived with the right model in mind.

> **API trap [S]:** `_epd_mode` is read at *draw* time for dithering and again at `display()` time
> to stamp the waveform. Draw in `epd_quality`, then `setEpdMode(epd_fastest)` before `display()`,
> and the fastest LUT indexes columns 1–14 and emits no-ops — **nothing happens.** Always set the
> mode before drawing.

### 3.3 The eraser is 4 frames, not 2 [S]

`lut_eraser` is a file-static
([`Panel_EPD.cpp:149-156`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L149-L156)):
2 driving rows + 1 pad + 1 terminator = `lut_eraser_step = 4`. Note 0002's "2-frame eraser" counted
driving rows; it costs **4 scans**. Used by `epd_quality` and `epd_text`; skipped entirely by the
fast modes, which jump straight to step 0
([`Panel_EPD.cpp:948-956`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L948-L956)).
Its own comment describes it as shifting the pixel from its current grey toward mid-grey; it exists
because the mode LUTs know only the target grey, so the eraser normalises the starting state to make
a target-only LUT well-defined. It is **not** pluggable.

Worth noting how unusual this is. Every other stack surveyed handles ghosting with a *periodic
clearing refresh*; M5GFX instead pays a per-pixel 2-frame erase-to-mid-grey on every changed pixel of
every greyscale update. That is a different bargain — no flash, but the greyscale modes are 4 scans
more expensive than their frame count suggests, and it is part of why `epd_text`'s refresh-in-place
impulse is uniformly white-ward.

### 3.4 The real concurrency model [S]

`setEpdMode` is **global**, not per-region
([`Panel.hpp:79`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/Panel.hpp#L79),
[`:98`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/Panel.hpp#L98)),
and it is a no-op on non-EPD panels — which is why the SDL simulator silently ignores it.

But the *state* is per-pixel. `_step_framebuf` holds two `uint16` per pixel-pair — an active slot
and a reserved slot — each `(global_step_index << 8) | two_4bit_greys`, read as `int16_t` so bit 15
means "inactive"
([`Panel_EPD.cpp:230-232`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L230-L232),
[`:830-836`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L830-L836)).
Because the step index is global across the concatenated eraser+4-mode table, one blit pass advances
every pixel along whatever step of whatever mode it is on. **There is exactly one scan loop, always
full-panel, and each pixel independently walks its own waveform — so the pet region genuinely can
be mid-`epd_fastest` while the chrome is mid-`epd_text`.** Note 0002 got this right.

**Overlap behaviour, which is easy to get wrong:**

| Mode | Unchanged pixel inside the dirty rect |
|---|---|
| `epd_quality` | **Always re-driven** ([`:1000-1013`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L1000-L1013)) |
| `epd_text` | Re-driven **unless pure white** ([`:962-982`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L962-L982)) |
| `epd_fast` / `epd_fastest` | **Skipped** ([`:930`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L930)) |

Exactly what the maintainer describes in
[issue #166](https://github.com/m5stack/M5GFX/issues/166) — "Areas with no gradation changes are
left unprocessed" — with "fast and fastest" load-bearing.

**Changing mode forces a full re-drive of the dirty rect**, because `lut_offset` changes so every
pixel compares unequal. Also from #166: "Immediately after changing the mode, a full refresh will
be performed." Verified in source.

> **There is no waveform queueing. A second update to a pixel already mid-waveform aborts and
> restarts it** — and an aborted waveform has applied an arbitrary partial impulse, DC-imbalanced by
> construction. **Never issue an update to a region faster than its mode's LUT completes**
> (7 frames for `epd_fastest`, 10 for `epd_fast`, 36 for quality/text). Gate on `waitDisplay()`;
> `displayBusy()` is **not** "is it refreshing" — it returns true only when the 8-slot queue is full
> ([`Panel_EPD.cpp:312-319`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L312-L319)).

### 3.5 Custom LUTs — pluggable, with three undocumented constraints [S]

`config_detail_t` exposes `lut_quality`/`lut_text`/`lut_fast`/`lut_fastest` and their step counts;
`init()` substitutes built-ins only when the pointer is null
([`Panel_EPD.cpp:180-197`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L180-L197)).
Genuinely pluggable, no fork. Layout: `const uint32_t[]`, one element per frame in time order,
16 two-bit fields indexed by target grey, values 0/1/2/3 as above. `lut_*_step` is the **total array
length including padding and terminator**, not the driving-frame count.

1. **The terminator must be an all-zero row.** End-of-data is tested on the packed 2-pixel byte, so
   both pixels must reach 0 in the same step. Mixing `3` with `0` in one byte **never terminates**.
2. **Omitting the terminator walks into the next mode's waveform.** No bounds check.
3. **Hard total-step budget of 128, undocumented** — the step lives in the high byte of an `int16_t`
   whose bit 15 is the inactive flag, so `step ≤ 127`. Current total is **[D]** `4+32+32+10+7 = 85`,
   leaving **43 steps of headroom across all four modes combined.**

**The obvious use for that headroom: M5GFX has no DU-class mode.** E Ink's mode ladder runs A2
(~10 frames, *Medium* ghosting) → **DU (~22 frames, *Low* ghosting, any grey → black or white)** →
GC16 (~38 frames). M5GFX jumps straight from 11 scans to 37, so our only mono modes are both
A2-class. If measurement shows `Flip` ghosts too much for the care log and hearts, authoring a
~22-frame DU-equivalent into `lut_fast` is the cheapest fix available and fits the step budget
comfortably. Recorded as an option, not a recommendation — it needs hardware first.

---

## 4. Ghosting: mechanisms, and what the budget actually is

### 4.1 E Ink's own mode table [S]

From **E Ink Corporation doc. 800-1101 Rev 01, 30 Apr 2015, "AF 16 Tone Grayscale 5-Bit Waveform
Flash File Product Specification"** — marked "E Ink Confidential" but publicly redistributed by
Waveshare ([PDF](https://www.waveshare.com/w/upload/c/c4/E-paper-mode-declaration.pdf)) and
corroborated by E Ink's own ED060KC1 kit manual on
[shopkits.eink.com](https://shopkits.eink.com/en/download/0223013011w2578422/User%20Manual%20-%206''ePaper%20Display%20Kits%20(ED060KC1).pdf).
This is the only E Ink-authored waveform document available in the open, and it is the origin of the
whole industry mode taxonomy. Its column header is verbatim "Typical update time at 25 C 85 Hz (ms)"
— the same 85 Hz our panel is rated at.

| Mode | Ghosting | @25 °C (ms) | Frames **[D]** | Nearest M5GFX mode |
|---|---|---|---|---|
| INIT | N/A | 2000 | ~170 | — |
| DU | **Low** | 260 | ~22 | **— (gap, see §3.5)** |
| GC16 | **Very Low** | 450 | ~38 | `epd_quality` (37 scans) |
| GL16 | Medium | 450 | ~38 | `epd_text` (37 scans) |
| GLR16 / GLD16 | Low | 450 | ~38 | — (Regal is host-side preprocessing, not a waveform) |
| A2 | **Medium** | 120 | ~10 | `epd_fast` (11), `epd_fastest` (8) |
| DU4 | Medium | 290 | ~25 | — |

Frame counts are division by 85 Hz, not published. Two independent cross-checks confirm the method:
epdiy's conversion of a genuine **ED047TC2 vendor waveform blob** gives GC16 = 38 phases in the
24–27 °C band, and Modos Labs' worked GC16 example is a 38-frame sequence.

**M5GFX's mode structure maps onto E Ink's own families**, which is a strong independent signal that
its hand-derived LUTs are sane — with the DU-shaped hole noted above. Useful supporting definitions
from the same document: DU "supports transitions from any graytone to black or white only"; A2's
usage is stated as **"Fast page flipping at reduced contrast"**, which is E Ink naming the animation
trade-off in one phrase.

### 4.2 The three mechanisms

**Cumulative quantised grey error — the everyday ghost.**
[US9613599B2](https://patents.google.com/patent/US9613599B2/en) (Nook Digital, 2015): errors arise
from "the quantized nature of the drive (i.e., some integer number of frame periods at +V, −V or
0V), the manufacturing variance of the EPD material, its driving history" and — the key sentence —
"**Since the errors are most pronounced at changing pixels, the pixel errors often resemble prior
displayed images; for this reason these cumulative pixel errors are termed ghosting.**" Epson says
the same in a hardware spec (S1D13521B01 §15): the flashing clear waveform exists for "removing any
build up compounded error from previous operations", while the non-flashing one "may not reduce
ghosting and may add cumulative pixel transition error."

**Remnant voltage / edge ghosting.** E Ink
[US11107425B2](https://patents.google.com/patent/US11107425B2/en): "a primary cause of remnant
voltage is **ionic polarization** within the materials of the various layers." E Ink
[US11568827B2](https://patents.google.com/patent/US11568827B2/en) names the visible result: "Remnant
voltages may give rise to '**edge ghosting**,' a type of ghosting in which an outline (edge) of a
portion of a previous image remains visible." Decay is fast —
[US8558783B2](https://patents.google.com/patent/US8558783B2/en) measures ±3 V immediately after a
15 V pulse falling to ±1 V one second later.

**DC imbalance — the cumulative, serious one.** This is what matters over a two-week life. E Ink's
standard formulation, repeated verbatim across
[US10852568B2](https://patents.google.com/patent/US10852568B2/en) and
[US11557260B2](https://patents.google.com/patent/US11557260B2/en):

> "The electro-optic properties and **the working lifetime of displays may be adversely affected**
> if the drive schemes used are not substantially DC balanced (i.e., if the algebraic sum of the
> impulses applied to a pixel during any series of transitions beginning and ending at the same gray
> level is not close to zero)."

US9613599B2 states the failure mode: repetitive imbalanced drive causes "**permanent ghosting, image
burn-in, and/or image sticking**". E Ink
[WO2005054933A2](https://patents.google.com/patent/WO2005054933A2/en): "**DC imbalances cause
long-term lifetime degradation of electrophoretic displays.**"

Note the definition's shape: balance is judged over **a series of transitions beginning and ending
at the same grey level**. That is a loop, not a count — and it is exactly the property our animation
can be designed to have.

### 4.3 Is there a documented "N partials before a full refresh"? Not from E Ink.

**Negative finding, and it is the important one: E Ink publishes no maximum consecutive fast-update
count.** Not in 800-1101, not in the ED047TC1 datasheet, not in the Epson Broadsheet spec, IT8951's
documentation or the SSD1677 datasheet. The widely-repeated "N partials then a full" has no E Ink
origin.

What E Ink *does* specify for A2 is a bracketing ritual:

> "The recommended update sequence to transition into repeated A2 updates is shown in Figure 1.
> **The use of a white image in the transition from 4-bit to 1-bit images will reduce ghosting** and
> improve image quality for A2 updates." … "**It is also recommended to use a white image after a
> sequence of A2 updates** as shown in Figure 2."

Entry is `4-bit image → white image → 1-bit image → A2`; exit is `A2 → white image → GC16`. This is
not folklore: **Allwinner's production BSP implements `A2_IN` and `A2_OUT` as first-class waveform
modes**, and the Rockchip EBC driver has `prepare_prev_before_a2` ("Convert prev buffer to bw when
switching to the A2 waveform"). Silicon vendors ship E Ink's Figure 1/2 as code.

The count-based rules that do exist are driver convention and module-vendor guidance:

| Source | Rule | Kind |
|---|---|---|
| Allwinner `eink200` BSP | `DEFAULT_GC_COUNTER 6` — after 6 *consecutive* non-clearing updates, force a **full-screen** GC16; hard ceiling 20 | count |
| KOReader `uimanager.lua` | `DEFAULT_FULL_REFRESH_COUNT = 6`; only `"partial"` counts toward promotion | count |
| Kobo sunxi ioctl `DISP_EINK_SET_GC_CNT` | user-settable 0–20, "only affects consecutive GU16 updates" | count |
| **Rockchip EBC (PineNote)** | `refresh_threshold = 20` in **screen-area multiples**; `area_count += (x2-x1)*(y2-y1)` per update | **accumulated area** |
| Good Display | "After every 5 partial updates, perform a full update"; "at least once every 24 hours" | count + time |
| i.MX EPDC | **no counter at all** — `WAVEFORM_MODE_AUTO` is a pure content histogram | none |
| M5GFX `Panel_EPD` | **no counter, no periodic clear** — a per-pixel eraser pre-pass instead (§3.3) | none |

Two independent codebases converged on 6; two independent sources cap at 20. That is as close to a
defensible industry number as exists — but it is convention, and it is shaped for full-screen page
turns, which is not what we do.

### 4.4 The right model is accumulated driven area, and there is shipping code for it

Ghosting is a **per-pixel** property. US9613599B2's charge history is per pixel and errors are "most
pronounced at changing pixels"; E Ink's GC16 description notes that under partial update "the only
pixels with changing graytone values will update". Pixels never driven accumulate nothing.

**The Rockchip EBC driver is the one implementation that budgets accordingly** — `Σ(update area)`
against `20 × screen_area`, rather than counting updates. Allwinner and KOReader count updates and
are therefore size-blind; KOReader even knows this is the weak point, warning that "making sure your
stuff only applies to the proper region is key to avoiding spurious large black flashes".

Applying Rockchip's constant to our layout gives the first real number in this note:

> **[D]** A pet window at ~5 % of the screen gets `20 / 0.05 = ~400` fast updates per clearing
> refresh — about **80 seconds of continuous 5 fps animation**, or a great many idle blinks.

That is generous for idle animation and a genuine constraint for a minigame round, which argues for
clearing at natural boundaries (a round starting or ending) rather than mid-animation. It is a
starting point borrowed from a different panel and a different waveform, and measurement 5 in §8
exists to replace it.

A refinement in our favour: our fast modes are DC-balanced (§3.2) where the A2 waveforms these
budgets were tuned against may not be. If anything 400 is conservative.

### 4.5 Why black-and-white animation accumulates less than mid-grey

US9613599B2 gives the mechanism: the material can be driven to "**optical saturation** (e.g., a
nearly full black or white state) where driving it harder into a black or white rail will have a
negligible effect on the grayshade", which "allows the driving system to better know the pixel state
with reasonable certainty" and "can counter grayscale error accumulation".

At the rails the pigment is packed against the electrode and the optical response saturates, so
timing and quantisation error cannot integrate into *optical* error — the pixel self-corrects toward
a known state on every drive. A mid-grey has no such stop. **This is why the pet is a 1-bit
silhouette and the chrome is greyscale, and it retroactively justifies the art direction on physics
rather than taste.**

Important caveat: this is about *grey drift*, not charge. A black↔white cycle still deposits net
charge unless the waveform is balanced — which is precisely why §3.2 matters and why rule 3 below is
the load-bearing one.

### 4.6 How much drift, and when is it visible

E Ink [US11557260B2](https://patents.google.com/patent/US11557260B2/en) gives the accumulation model:

> "imagine that temperature dependence results in a 0.2 L* error in the positive direction on each
> transition. **After fifty transitions, this error will accumulate to 10 L*.** …suppose that the
> average error on each transition… is ±0.2 L*. **After 100 successive transitions, the pixels will
> display an average deviation from their expected state of 2 L*.**"

Systematic error grows linearly; random error grows as √N. **Our design goal is to make our error
random rather than systematic — which is exactly what DC balance buys.**

For an acceptance threshold, SinoCrystal panel spec
[SCP075001-V01](https://www.displaysino.com/upload/portal/20230814/535b7278c9cdbcc06e65a2a072dbfeb1.pdf)
§7.2.2 measures ghosting as CIEDE2000 ΔE00 with limits of **≤ 2** (≤ 4 for two of six cases). So
**ΔE00 ≈ 2 is the industry line for "no visible ghosting"**, and it is what our hardware measurement
should test against.

### 4.7 Border ghosting — the best-attested number in the whole investigation

Pervasive Displays, *COG Driver Interface Timing*, Doc. 4P008-00, §1
([PDF](https://files.seeedstudio.com/wiki/Small_e-Paper_Shield/res/4P008-00_02_COG_Driver_Interface_Timing_for_smallPlussize.pdf)),
repeated in Rev 03 and in 4P015-00:

> "Around the active area of the EPD is a 0.5mm width blank area called the border. It should be
> connected to VDL (-15V) to keep the border white. **After approximately 10,000 updates with the
> constant voltage, the border color may degrade to a gray level that is not as white as the active
> area.** To prevent this phenomenon, PDI recommends turn on and off border to avoid the
> degradation."

Three documents, two revisions, same figure. It is the cleanest published proof that
firmware-applied sustained DC causes cumulative, visible, permanent degradation. The ED047TC1
datasheet does list a distinct **Border VCOM** rail whose value is given only as "Adjusted", and the
PaperS3's panel connector carries a `BORDER` pin. **[U]** Whether M5GFX drives it or ties it off was
not established, and on a device that will run for weeks this should be checked.

---

## 5. Temperature: there is no sensor, so there is no decision

### 5.1 M5GFX has none [S]

`grep -rniE "temperature|thermal|ntc|sht30|therm"` across `src/` returns three hits — the unused
epdiy shim and an unrelated panel. `Panel_EPD.cpp`, `Panel_EPD.hpp`, `Bus_EPD.cpp`: **zero.**
`git log --all -i --grep="temperat"` across M5GFX's entire history: **zero commits.**

epdiy is no better on this board: its V7 and Lilygo-S3 board definitions **hardcode**
`return 20;` for ambient temperature, and its built-in ED047TC1 waveform ships exactly one
temperature band (`{ .min = 20, .max = 30 }`). Its own docs concede the consequence: "if your room
temperature is significantly different from ~22 °C, grayscale accuracy might be affected when using
the builtin waveform." **Neither available stack compensates for temperature on this panel.**

### 5.2 The board has none either — verified at netlist level [S]

Against the official schematic
([sch_papers3_V1.0.pdf](https://m5stack-doc.oss-cn-shenzhen.aliyuncs.com/517/sch_papers3_V1.0.pdf),
linked from [docs.m5stack.com/en/core/PaperS3](https://docs.m5stack.com/en/core/PaperS3)):

- Complete active-component list is `PMS150G-U06`, `ESP32_S3R8`, `SY8089AAAC`, `LGS4056H`,
  `BM8563EMA`, `XM25QH128CHIQ`, `MT9700` ×2, `MT3608`, `ME6203A33M3G`, `BMI270`. **No SHT30, no
  temperature IC.**
- Passives run R1–R55, C1–C56, D1–D16, L1–L10. **No `RT`/`TH`/`NTC` designator — no thermistor is
  fitted.**
- **The one net named `TEMP` is a dead end** — charger `LGS4056H` pin 1, the battery-NTC input, is
  wired straight to GND.
- EPD rail generation is open-loop: MT3608 boost → `EPD_VPOS`, inverting charge pump → `EPD_VNEG`,
  zener clamps for `VGH`/`VGL`, resistive VCOM. No sense node.
- The panel connector has no temperature pin and no I²C. It is a raw panel — unlike M5Paper v1's
  IT8951.

**M5Paper v1 had an SHT30; PaperS3 removed it along with the IT8951 controller.** M5Unified reflects
this: no SHT30 driver exists for any board, the RTC is a BM8563 (no temperature register, unlike a
DS3231), and power is plain ADC with no PMIC die temperature
([`Power_Class.cpp:262-269`](https://github.com/m5stack/M5Unified/blob/4fb444784c85791e0b0207701392b42be234b2e7/src/utility/Power_Class.cpp#L262-L269)).

This is a departure from the reference design in both directions: **§13 of the ED047TC1 datasheet's
own block diagram shows a Temperature Sensor block wired to the EPD controller**, and the reference
PMIC for this class of panel (TI TPS65185) exists partly to read a 10k NTC and expose `TMST_VALUE`.
The PaperS3 has neither.

Two die sensors exist and both are unsuitable. The BMI270's is exposed as `M5.Imu.getTemp()`
([`IMU_Class.cpp:518-528`](https://github.com/m5stack/M5Unified/blob/4fb444784c85791e0b0207701392b42be234b2e7/src/utility/IMU_Class.cpp#L518-L528))
but Bosch requires the gyro in normal mode for validity and publishes no absolute accuracy. The
ESP32-S3's on-die sensor is disclaimed by Espressif themselves —
[ESP-IDF v6.0 docs](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/peripherals/temp_sensor.html):
"it's **not recommended to use it for ambient temperature measurement**."

### 5.3 What temperature actually does

ED047TC1 §6-1: operating **0 to +50 °C**, storage −25 to +70 °C. E Ink's own
[FAQ](https://www.eink.com/faqs.html) gives the same 0–50 °C for most displays.

**No public source gives a GC16-time-versus-temperature curve** — E Ink and Epson both publish a
single 25 °C column, and the waveform specification itself is NDA-only. But epdiy's conversion of a
genuine **ED047TC2 vendor waveform blob** exposes the real 14-band temperature range table, and it
is the closest thing to hard data available for a sibling of our panel:

| Band | 15–18 | 18–21 | 21–24 | **24–27** | 27–30 | 30–33 | 33–38 |
|---|---|---|---|---|---|---|---|
| DU phases | 25 | 22 | 22 | **22** | 18 | 17 | 15 |
| GC16 phases | 46 | 43 | 40 | **38** | 38 | 44 | **57** |

Two things fall out, and the second is counter-intuitive:

1. **Cold costs ~21 %** — GC16 is 46 frames at 15–18 °C against 38 at 24–27 °C (541 ms vs 447 ms at
   85 Hz). Modest.
2. **Hot costs more than cold.** GC16 at 33–38 °C is **57 frames**, 50 % above room temperature, and
   the trend reverses above 30 °C. Anyone assuming "warmer is faster" is wrong. DU, by contrast, is
   monotonic (25 frames cold → 15 hot).

Compensation everywhere it exists is **table selection, never frame-count scaling** — confirmed
across the i.MX EPDC, Allwinner's BSP, the DRM EPD helper and SSD1677. So update duration changes in
discrete steps at band boundaries, and a stack with one table (ours) simply has one duration.

The failure mode below range is worse than slowness. Good Display's GDEP133UT3 spec, carrying an E
Ink-sourced waveform table, is explicit: "**GC16 and INIT are only valid from 0 to 50 C. If used
outside the range, the display will update and the image quality will be very poor.**" SSD1677 will
refuse the update outright if no band matches. Crystalfontz states plainly that "**panel life is not
guaranteed when worked in temperatures below 0 degrees or above 50 degrees**". And Pervasive Displays
puts "**Operating outside the acceptable temperature range may damage the panel**" under a *Danger*
admonition — with the notable detail that **their fast-update window (0…+50 °C) is narrower than
their global-update window (−15…+60 °C)**, i.e. the fast modes are the temperature-fragile ones.

**Conclusion: do not build temperature compensation.** No sensor, no vendor curve, no second table
to select, and the fallback die sensors read high by an unquantified load-dependent offset. The pet
lives indoors; 0–50 °C covers any realistic desk. Record 0 °C as a hard *validity* boundary rather
than a performance one, and if #12 ever adds an external I²C sensor on the HC1.25-4PLT port
(`GPIO1`/`GPIO2`), the correct response is to *gate the fast and greyscale modes off* below ~5 °C,
not to retime them.

---

## 6. Panel longevity, and what software can actually damage

### 6.1 The "1,000,000 updates" figure is real but belongs to other panels

**No lifetime or endurance specification exists in the ED047TC1 datasheet, its ES103TC1 sibling, or
E Ink's waveform specification.** ED047TC1's reliability section is environmental soak and thermal
cycling only. The 1 M figure is genuine — Crystalfontz CFAP400300B1-0420 gives "1,000,000 times or
5 years", Good Display GDEQ0426T82 the same — but those are **small integrated-COG modules with
3.5–8 s update times**, a completely different population from an 85 Hz externally-driven Carta
panel. Anyone quoting 1 M for ED047TC1 has no primary source.

### 6.2 Sustained DC is the documented software-caused failure

Waveshare
[precautions](https://www.waveshare.com/wiki/Template:E-paper-precautions-color) — first-party and
unambiguous:

> "Note that the screen cannot be powered on for a long time. When the screen is not refreshed,
> please set the screen to sleep mode or power off it. Otherwise, **the screen will remain in a high
> voltage state for a long time, which will damage the e-Paper and cannot be repaired!**"

Good Display attributes the same hazard to the IC rather than the film — keep the distinction, but
the operational rule is identical. They also name a *software timing* variant: "there shouldn't be
any delay between initialization and refresh because that will let the E-paper film stay on
high-voltage."

M5GFX does power the panel down between updates
([`Bus_EPD.cpp:88-90`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Bus_EPD.cpp#L88-L90)),
so we inherit correct behaviour — but **[U]** we should confirm the rails actually collapse in deep
sleep, which is #12's business anyway, and that the datasheet's ordered power-down (`Ted` ≥ 0.5 s,
VGL last) is honoured.

**No vendor states that aborting a waveform mid-sequence damages the panel.** That it leaves a
partial imbalanced impulse follows from the definitions in §4.2, but it is our inference, not a
sourced claim. It is still a good enough reason for the port to serialise updates per region.

### 6.3 Static images and the resting state

Contested, and the two positions are probably about different ink types. In E Ink's favour: an
unpowered bistable Carta pixel is under no bias, so the film mechanism for burn-in is weak. Against:
Waveshare warns "if the screen keeps the same picture for a long time, the screen will burn and it
is difficult to repair", and Good Display advises storing with a white pattern because "displaying a
white pattern during storage prevents inerasable ghosting". Notably, **every storage test in the
ED047TC1 reliability matrix is annotated "Test in white pattern"** — E Ink's own protocol treats
white as the safe resting state.

Our reading: the vendor warnings most plausibly concern pigment settling in multi-colour media plus
residual charge, not monochrome Carta held at zero power. The design keeps the pet on screen at rest
and that is fine. What we take from this is narrower and uncontested — **the 24-hour full-refresh
rule**, which both vendors give and which is the one count-like rule with damage framing behind it.

KOReader implements a nice concrete version of the same instinct for its sleep screen: two redraws
1000 ms apart, each flashing black then redrawing, with the setting described as "anti-ghosting
redraws". The 1 s spacing is a decent empirical hint at the relaxation time constant, and matches
the ±3 V → ±1 V decay measured in US8558783B2.

### 6.4 The M5GFX "excessive strain" bug, and what it changed

The fix is
[`e1276ee`](https://github.com/m5stack/M5GFX/commit/e1276ee2595be1bd48e16c4ee2259022eada6185)
("improve EPD control for PaperS3", 2025-09-12), between v0.2.11 and v0.2.12. The maintainer, in
[issue #152](https://github.com/m5stack/M5GFX/issues/152#issuecomment-3331096133):

> "v0.2.11 has a **fatal control issue that has already been found to place excessive strain on the
> EPD**. The effects of this load can remain for **several tens of minutes even after the power is
> turned off.**"

Decoding both LUT versions **[D]** shows exactly what was wrong:

| Mode | v0.2.11 col 0 (black) | net | pinned SHA | net |
|---|---|---|---|---|
| `fastest` | `111111` | **−6** | `21111` | **−3** |
| `fast` | `1211111111` | **−8** | `22111111` | **−4** |
| `text` | (17 rows) | **−13** | (12 rows) | **−1** |

> **The old `lut_fastest` black column was literally `111111` — six consecutive unipolar to-black
> frames with no opposing pulse at all.** That is textbook remanent kickback, and it matches the
> maintainer's description of "gradation shift in the opposite direction after release". Every
> current LUT opens with a short opposite-polarity balancing pulse; `lut_text`'s imbalance fell 13×.

This is reassuring about the current code and alarming about version pinning. **Pin the exact
version and re-decode the LUTs on any upgrade** — a well-meaning waveform "tweak" upstream is
indistinguishable, from our side, from a panel-damaging regression.

### 6.5 The damage reports remain unattributed

Only two issues in M5GFX concern panel damage — [#152](https://github.com/m5stack/M5GFX/issues/152)
and [#166](https://github.com/m5stack/M5GFX/issues/166). The maintainer's own triage:

> "There may be a flaw in the M5GFX's control, causing overload on the panel. / There may be a flaw
> in the PaperS3's EPD control circuit… / The EPD panel itself may be designed to be more
> susceptible to damage. At this point, all of these are possibilities, but I don't know for sure."

Three anecdotes, no attribution, on an EOL panel. It should shape the budget as a precaution and be
reported as nothing more.

### 6.6 Two current defects at the pinned SHA that the port must work around [S]

**(a) The dirty rect is truncated to a multiple of 4 pixels in X.**
[`Panel_EPD.cpp:933-934`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L933-L934)
does `w = new_data.w >> 1; w &= ~1u;` — the fix for the heap overrun in
[issue #181](https://github.com/m5stack/M5GFX/issues/181), which resolved the crash by *truncating*.
**[D]** A 6-px-wide update stages 4 px; **a 1- or 2-px-wide update stages nothing at all.** The port
must round every rect's width out to a multiple of 4.

**(b) The no-argument `display()` skips the PSRAM cache write-back.**
[`Panel_EPD.cpp:586`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L586)
computes the flush extent from its arguments, but `LGFXBase::display(void)` passes `(0,0,0,0)`, so
`h == 0` and **zero bytes are flushed** while a non-empty range is queued. The EPD task is pinned to
the *other* core, and the maintainer's own comment warns about exactly this hazard
([`Panel_EPD.cpp:294`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L294)).
**[U]** whether it manifests on the S3's shared data cache is unverified — but the mitigation is
free.

**Related trap:** `Panel_EPD` sets `_auto_display` true in its constructor
([`Panel_EPD.cpp:164`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L164)),
and `endWrite()` calls `display(0,0,0,0)` when it is set
([`Panel.hpp:88`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/Panel.hpp#L88)).
**Every un-nested drawing call therefore costs a tick and a queue slot, and hits defect (b).** The
port must `setAutoDisplay(false)` and always call `display(x, y, w, h)` explicitly.

---

## 7. The refresh policy

This is the deliverable. It is stated as rules the mode state machine (#11) and every game can be
written against, and it is deliberately conservative where hardware has not spoken.

### 7.1 Four refresh intents, and what they map to

The port takes *our* enum, never `epd_mode_t` — as #2's port sketch already required.

| Our intent | M5GFX mode | Scans | Colour | E Ink class | Use |
|---|---|---|---|---|---|
| **`Animate`** | `epd_fastest` | 8 | 1-bit | A2 | Pet motion, game objects. The only mode used in a loop. |
| **`Flip`** | `epd_fast` | 11 | 1-bit | A2 | Discrete state changes — a heart filling, a care-log line landing. Higher contrast than `Animate`. |
| **`Settle`** | `epd_text` | 37 | 16-grey | GL16 | Redrawing paper chrome onto white. Skips white pixels; uniformly white-ward. The default for chrome. |
| **`Clear`** | `epd_quality` | 37 | 16-grey | GC16 | The clearing refresh. Re-drives everything, flashes, discharges ghosting. Rare and deliberate. |

`Settle` and `Clear` cost the same time (§1.5). `Settle` is the gentler default; `Clear` is reserved
for actually removing accumulated error. Note both mono intents are A2-class — there is no DU-class
mode on this driver (§3.5), which is the first thing to author a custom LUT for if `Flip` ghosts.

### 7.2 The ten rules

1. **`CONFIG_FREERTOS_HZ=1000` in `sdkconfig.defaults`.** Roughly halves every refresh. If
   measurement puts the frame period below the panel's 11.76 ms floor, drop to `HZ=500` (2 ms
   quantum → 12 ms) rather than raising the pixel clock, which would take us out of spec.
2. **Budget accumulated driven area, not update count.** The Rockchip model: track
   `Σ(region area × scans)` and clear when it reaches the equivalent of ~20 full-screen passes.
   **[D]** For a pet window at ~5 % of screen that is **~400 `Animate` updates**, ~80 s of
   continuous 5 fps animation. A count-based rule borrowed from e-reader page turns would over-clear
   by roughly (screen area ÷ window area).
3. **Every animation run must begin and end on the same rest frame.** `Animate` is exactly balanced
   over a complete black↔white cycle and leaves ±3 frame-units if it stops mid-cycle. A run that
   returns to its rest pose is net-zero *by construction*. This is a constraint on the animation
   vocabulary (#13), not a driver setting, and it is the single cheapest thing we can do.
4. **Bracket every animation run with a white frame**, entering and leaving. This is E Ink's own
   published A2 ritual, shipped as `A2_IN`/`A2_OUT` waveform modes by Allwinner and as
   `prepare_prev_before_a2` by Rockchip. It costs one extra `Animate` update per run.
5. **Never issue a new update to a region before its previous waveform completes** — 7 frames for
   `Animate`. Aborted waveforms are imbalanced by construction. The port serialises per region and
   gates on `waitDisplay()`, never `displayBusy()`.
6. **`Clear` is event-triggered, never periodic.** Each `Clear` pass is itself DC-imbalanced (+2 on
   white, −5 on black, §3.2), so a timer-driven cleanup is a cumulative DC source. Trigger it on
   entering a session, on the pet waking, on the pet going to sleep, or at a minigame round
   boundary — moments that already justify a visual transition, so a 400–740 ms flash reads as
   punctuation rather than a glitch.
7. **At least one `Clear` per 24 hours.** The one count-like rule with damage framing behind it
   (Waveshare, Good Display, §6.3). It costs nothing to honour: demand-driven sleep (ADR-0002)
   guarantees a daily wake transition to hide it in.
8. **Keep the pet 1-bit and the chrome greyscale.** Saturated black and white self-correct on every
   drive; mid-greys integrate their error (§4.5). The art direction was chosen on taste and turns
   out to be the physically correct split.
9. **Prefer small regions — for wear, CPU and budget, not for speed.** Staging cost scales linearly
   with area, only staged pixels are driven, and rule 2's budget is denominated in area. #17 should
   size the pet window for composition and for cumulative stress, and must not size it hoping to buy
   framerate.
10. **Pin the M5GFX version exactly, and re-decode the LUTs on every upgrade.** §6.4 is the reason:
    a waveform "tweak" upstream is indistinguishable from a panel-damaging regression from our side.

### 7.3 What the port must own

Beyond #2's `present(region, mode)` / `waitIdle()` sketch, this note adds five requirements:

- **Round every region's X and width out to a multiple of 4** (defect 6.6a). Layout code should
  never know.
- **`setAutoDisplay(false)` at init; always call `display(x, y, w, h)` with an explicit rect**
  (defect 6.6b and the `endWrite` trap).
- **Set the EPD mode before drawing, not before presenting** (§3.2 trap).
- **Own the area budget and the per-region serialisation.** Core emits a repaint effect carrying an
  intent; the port decides whether that intent is affordable now and whether a `Clear` is owed. Core
  must not know about waveforms.
- **Own the white-frame bracketing** so games and idle animation cannot forget it.

### 7.4 Answers the downstream tickets can build on

**#10 — is 5 fps real?** Yes, and the spike's question changes. `Animate` sustains **~7 fps at the
stock tick rate and ~10–13 fps at `HZ=1000`** **[D]**, so the framerate is available. But every frame
clocks the whole panel regardless of region size, so a 5 fps game costs the same panel energy as
refreshing the full screen five times a second — and rule 2 gives a round a budget of roughly **80
seconds of continuous animation** before it owes a clearing flash. The real questions are now "is it
worth the power" (#12) and "does a round fit in 80 seconds", not "is it fast enough".

**#13 — smooth-and-rare versus slow-and-constant.** Nearly cost-neutral. **[D]** Two seconds of 5 fps
once a minute is 10 updates/min; one frame every ten seconds is 6 updates/min — only 1.7× apart, both
at 8 scans each. **Choose on aesthetics, not on cost.** The tiebreaker is rule 3: a short animation
that returns to a rest pose is self-balancing, whereas a slow drift through poses that never closes a
loop is not.

**#12 — power.** The unit to multiply is **full-panel scans**, not updates and not pixels: 8 per
`Animate`, 11 per `Flip`, 37 per `Settle` or `Clear`. Energy per scan is unmeasured **[U]** and is
the top item on the measurement plan.

**#17 — layout.** The pet window's size buys panel longevity, CPU and ghosting budget — not
framerate. Chrome and pet can genuinely refresh concurrently in different modes (§3.4), so the layout
is free to treat them as independent regions.

**#11 — state machine.** Permitted intents per state fall out directly: ambient gets rest-framed
`Animate` and occasional `Flip`; session gets everything; the transition *into* session is the
natural home for an owed `Clear`; sleep gets nothing but the daily `Clear` at the wake boundary.

---

## 8. The measurement plan

Everything above is model. This is the afternoon that turns it into fact, in priority order.

1. **Frame period.** Instrument the `for (y…)` scan loop with `esp_timer_get_time()`; run at
   `HZ=100` and `HZ=1000`. Confirms or refutes the 10.6 ms floor and the tick-quantisation model —
   the foundation everything else rests on.
2. **Blit cost in isolation** (§1.7). Time the blit separately from the bus writes. Tells us whether
   PSRAM bandwidth co-determines the frame time.
3. **Wall-clock per mode, full screen versus a 200×100 box.** `display()` → `waitDisplay()` for all
   four modes. Confirms the no-area-scaling claim at the API level, where the design actually lives.
4. **Staging cost versus area.** Validates the ~26× figure and therefore rule 9.
5. **Ghosting curve.** Run `Animate` toggles on a test region; photograph against a freshly-cleared
   reference at N = 10/50/100/400/1000 and compute ΔE00. **This is what replaces the borrowed
   400-update budget with our own number**, measured against the ΔE00 ≈ 2 threshold (§4.6).
6. **Rest-frame balance.** The same curve for runs that end on the rest frame versus runs that stop
   arbitrarily, and for runs with and without white-frame bracketing. Directly tests rules 3 and 4,
   which are the load-bearing claims of the whole policy.
7. **Current draw per mode**, for #12.
8. **Two cheap checks:** whether `BORDER` is driven or tied off (§4.7), and whether the
   `memory_height` truncation trick produces a clean short scan (§2.1).

---

## What remains uncertain

1. **No timing was measured.** Everything is arithmetic from source constants. The realistic frame
   period band is 10.6–16.2 ms and the difference matters: it is the difference between 10 fps and
   13 fps, and between being inside and outside the panel's 85 Hz rating.
2. **The ~400-update budget is borrowed.** It is Rockchip's `20 × screen_area` constant, tuned for a
   different panel and a waveform that may not be DC-balanced, divided by our window fraction. The
   *model* is well-founded and the *number* is a placeholder. Measurement 5 is the one that matters.
3. **DC sums are frame counts, not charge.** They assume identical frame duration and fixed ±V and
   ignore state-dependent physics. Good for relative comparison, not absolute prediction. No vendor
   publishes per-mode DC balance for any panel, so there is nothing to check them against.
4. **The `epd_fastest` LUT is undocumented and hand-derived.** M5GFX ships no rationale for its
   5 driving frames, and E Ink publishes nothing shorter than A2's ~10. Whether 5 frames genuinely
   completes a transition on this panel, or merely gets close enough to look right while leaving
   residue, is exactly what measurement 5 tests.
5. **Nothing was built against ESP-IDF v6.** Carried forward unchanged from #2 and still the first
   thing to do.
6. **The panel-damage reports remain unattributed** (§6.5), and the coincidence that epdiy would
   drive this panel 66 % above its rated refresh rate (§1.4) is a hypothesis, not a finding.
7. **`BORDER` handling is unknown.** The pin exists on the connector; whether M5GFX drives it was not
   established, and §4.7 is the one place where a specific update count is documented to cause
   permanent degradation.
8. **The 180-second inter-update interval** advised by Good Display and Crystalfontz is never framed
   by a first-party source as damage prevention — only as ghosting and lifespan guidance. We are
   deliberately ignoring it during sessions and honouring only the 24-hour rule. **[U]** If
   measurement 5 shows a worse curve than expected, this is the next lever.
9. **Temperature timings come from the ED047TC2 blob, not TC1.** The 46/38/57-phase figures are a
   sibling panel's vendor waveform as converted by epdiy, and the band-index mapping is inferred from
   a Makefile filter rather than stated. Direction is trustworthy; magnitudes are not. (Related trap
   if anyone ever switches to that waveform: epdiy's ED047TC2 header declares `num_temp_ranges = 7`
   while shipping 14 intervals, so it selects the 33–38 °C table at room temperature.)
