# Display driver: M5GFX under ESP-IDF, epdiy, or a bare panel driver?

Research for [issue #2](https://github.com/r4stered/papergotchi/issues/2). Everything the pet
looks like renders through this choice, so it gates the refresh strategy (#9), the layout
(#17), the simulator's display port (#20) and the art pipeline.

Source was read at two pinned commits, and every file:line citation below is a permalink at
those commits:

- **M5GFX** [`729297d`](https://github.com/m5stack/M5GFX/tree/729297d6e3d657ddc1ec5189bac2f2ea68828085) (2026-07-22, v0.2.26)
- **epdiy** [`7c30780`](https://github.com/vroland/epdiy/tree/7c3078092479d9eabcf1dadcd928b5f3e446284e) (2026-08-01, v2.1.3)

---

## What the hardware actually is

| | |
|---|---|
| MCU | ESP32-S3R8, Xtensa LX7 dual-core @ 240 MHz |
| PSRAM | 8 MB — **must be configured Octal (OPI)** |
| Flash | 16 MB |
| Panel | `EPD_ED047TC1`, 4.7", 960 × 540, 16-level greyscale |
| Touch | GT911 (2-point) |
| Status | **EOL** — "This product is EOL (End of Life)." |

Specs and the OPI-PSRAM requirement: [M5Stack PaperS3 docs](https://docs.m5stack.com/en/core/PaperS3).
EOL wording: [M5Stack store listing](https://shop.m5stack.com/products/m5papers3-esp32s3-development-kit).

Electrically this is a raw parallel EPD — the SoC drives the panel's own row/column drivers
(SPV, CKV, SPH, OE, LE, CL + an 8-bit data bus) and generates the waveform itself. There is
no display controller doing the work for us, which is why the waveform question is a real
question and not a register write. The exact pin map is in M5GFX at
[`M5GFX.cpp:2101-2145`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/M5GFX.cpp#L2101-L2145):
data `GPIO 6,14,7,12,9,11,8,10`, `PWR 46`, `SPV 17`, `CKV 18`, `SPH 13`, `OE 45`, `LE 15`,
`CL 16`, bus 8-bit @ 16 MHz, plus `PWROFF_PULSE` on `GPIO 44`
([`M5GFX.cpp:2091-2093`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/M5GFX.cpp#L2091-L2093)).

---

## Candidate A — M5GFX as an ESP-IDF component

### Does it support M5PaperS3 as a pure ESP-IDF component?

Yes, and the Arduino-first worry is out of date.

- The board is a first-class entry: `board_M5PaperS3 = 19` in
  [`boards.hpp:30`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/boards.hpp#L30),
  runtime-autodetected by probing GT911 on I2C at GPIO 41/42
  ([`M5GFX.cpp:2035-2090`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/M5GFX.cpp#L2035-L2090)).
- The transport is **`esp_lcd`'s i80 bus** — `esp_lcd_new_i80_bus` / `esp_lcd_new_panel_io_i80` /
  `esp_lcd_panel_io_tx_color`, i.e. bare ESP-IDF
  ([`Bus_EPD.cpp:124-157`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Bus_EPD.cpp#L124-L157)).
  No Arduino core anywhere in the path.
- The component `CMakeLists.txt` has an explicit **ESP-IDF v6** branch
  ([`CMakeLists.txt:21-22`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/CMakeLists.txt#L21-L22),
  added 2026-06-25 in commit `38c7fcd`), and the `arduino-esp32` requirement is commented out
  by default ([`CMakeLists.txt:34-35`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/CMakeLists.txt#L34-L35)).
- Published to the ESP Component Registry as
  [`m5stack/m5gfx` v0.2.26](https://components.espressif.com/components/m5stack/m5gfx), MIT,
  "supports all targets", ~5.6k downloads.
- There is an ESP-IDF build-test project in-tree
  ([`examples/Test/build_test/`](https://github.com/m5stack/M5GFX/tree/729297d6e3d657ddc1ec5189bac2f2ea68828085/examples/Test/build_test))
  and a PlatformIO `framework = espidf` env
  ([`platformio.ini`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/examples/PlatformIO_SDL/platformio.ini)).

Two build-system caveats, both checked rather than assumed:

1. M5GFX registers via the legacy `register_component()` macro
   ([`CMakeLists.txt:40`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/CMakeLists.txt#L40)).
   That macro **still exists on ESP-IDF v6.0** — it is defined under "Deprecated functions" in
   [`tools/cmake/component.cmake`](https://github.com/espressif/esp-idf/blob/release/v6.0/tools/cmake/component.cmake)
   and forwards to `idf_component_register`. It works; it is on a deprecation path.
2. The PSRAM guard tests the pre-5.0 symbol `CONFIG_ESP32S3_SPIRAM_SUPPORT`
   ([`M5GFX.cpp:2096-2099`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/M5GFX.cpp#L2096-L2099)),
   and if it is undefined the panel is **not** initialised. On ESP-IDF v6.0 that symbol is still
   emitted as a deprecated alias for `CONFIG_SPIRAM` via
   [`components/esp_hw_support/sdkconfig.rename.esp32s3`](https://github.com/espressif/esp-idf/blob/release/v6.0/components/esp_hw_support/sdkconfig.rename.esp32s3)
   (`CONFIG_ESP32S3_SPIRAM_SUPPORT → CONFIG_SPIRAM`). So it works today. It is a silent
   failure mode if Espressif ever drops the alias.

### It no longer depends on epdiy — and that is the headline

M5GFX's own doc states it plainly:

> "The following explanation is for v0.2.6 and earlier. **EPDiy is no longer used from v0.2.7.**"
> — [`docs/M5PaperS3.md`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/docs/M5PaperS3.md)

The git history confirms the arc: `7ec94c5 add support M5PaperS3 (with EPDiy library)`, later
replaced by the native `Bus_EPD` + `Panel_EPD` pair. The epdiy shim
([`Panel_EPDiy.hpp`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/panel/Panel_EPDiy.hpp))
survives but is compiled only `#if __has_include(<epdiy.h>)` and is not used for this board.

An upstream that has already migrated *off* epdiy for this exact panel is the single strongest
data point in the whole investigation.

### Waveform and update-mode control

Four modes, selected per draw via `setEpdMode()`
([`LGFXBase.hpp:1431`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/LGFXBase.hpp#L1431),
enum at [`enum.hpp:44-50`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/misc/enum.hpp#L44-L50)).
Counting the LUT tables at
[`Panel_EPD.cpp:82-155`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L82-L155):

| Mode | Frames | Greys driven | Notes |
|---|---|---|---|
| `epd_quality` | 15 | all 16 | full greyscale, flashing; preceded by a 2-frame eraser |
| `epd_text` | 12 | all 16 | greyscale, skips already-white pixels |
| `epd_fast` | 8 | index 0 and 15 only | **effectively mono** — every mid-grey column is `3` = no-op |
| `epd_fastest` | 5 | index 0 and 15 only | **effectively mono** |

So we get both halves of what issue #2 asks for: fast mono partial updates *and* 16-grey full
refreshes, from one API, with per-pixel LUT state so different screen regions can be mid-update
in *different* modes simultaneously.

Custom LUTs are a supported extension point, not a fork: `Panel_EPD::config_detail_t` exposes
`lut_quality` / `lut_text` / `lut_fast` / `lut_fastest` pointers and step counts
([`Panel_EPD.hpp:41-57`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.hpp#L41-L57)),
defaulting to the built-ins only if null
([`Panel_EPD.cpp:180-196`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L180-L196)).

**Gap: no temperature compensation.** Grepping `Panel_EPD.cpp` and `Bus_EPD.cpp` for
`temperature` returns nothing. The LUTs are fixed. E-ink switching speed is strongly
temperature-dependent, and this is a device that sits on a desk through winter and summer.

### Sub-regions and whether time scales with area

Region granularity is good: `display(x, y, w, h)` accumulates a dirty bounding box and queues
it, with `x` snapped to even pixels because the buffer is 4 bits per pixel
([`Panel_EPD.cpp:553-597`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L553-L597)).
Unchanged pixels inside the region are skipped — the staging loop only writes a pixel's LUT
state when the target differs from what is already queued
([`Panel_EPD.cpp:932-1024`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L932-L1024)).

**But the panel scan does not shrink.** The driver task clocks *every* row of the panel on
*every* waveform frame, regardless of the dirty rectangle:

```c
for (uint_fast16_t y = 0; y < mh; y++) {          // mh == full panel height
  blit_dmabuf(...); bus->writeScanLine(...);
}
```
— [`Panel_EPD.cpp:1048-1060`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L1048-L1060)

The library maintainer describes the same behaviour from the user side in
[M5GFX issue #166](https://github.com/m5stack/M5GFX/issues/166):

> "The `epd_fast` and `epd_fastest` modes prioritize speed and do not aim for a complete clear
> image. **Areas with no gradation changes are left unprocessed.**"

So the cost model is: **update time ≈ (frames in the mode) × (one full-panel frame), and is
essentially independent of region area.** Region size buys you less panel stress, less ghosting
spread and less CPU — not less wall-clock time. That is a direct input to #9: to make a refresh
cheap, pick a shorter mode, not a smaller box.

Derived floor for one frame — *arithmetic from source, not measured*: `write_len = 960/4 + 8 =
248` bytes per line at `pclk = 16 MHz` gives ~15.5 µs/line × 540 lines ≈ **8.4 ms/frame**. That
would put `epd_fastest` at ~42 ms and `epd_quality` at ~140 ms. **However**, `task_update` calls
`vTaskDelay(1)` once per frame
([`Panel_EPD.cpp:1063`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L1063)),
so at the default `CONFIG_FREERTOS_HZ=100` the frame period is floored at 10 ms and
`epd_fastest` becomes ~90 ms (~11 fps). Raising the tick rate to 1000 Hz should mostly remove
that. **Unverified on hardware** — measure before believing either number, and treat it as a
tunable for the 5 fps question in #10.

### Memory

Three allocations at
[`Panel_EPD.cpp:228-240`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L228-L240):
the 4 bpp framebuffer (960 × 540 / 2 = 259 200 B) and the per-pixel LUT-progress buffer
(960 × 540 / 2 × 2 × 2 = 1 036 800 B) — **~1.3 MB of the 8 MB PSRAM**, plus a ~1 MB DMA LUT in
internal RAM (`lut_total_step × 256 × 2` bytes, `MALLOC_CAP_DMA`). The driver runs its own
pinned FreeRTOS task
([`Panel_EPD.cpp:293`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L293)),
so `display()` is asynchronous and drawing overlaps refresh.

### Simulator

This is where M5GFX runs away with it. LovyanGFX ships an SDL platform, and `M5GFX::autodetect`
on a host build constructs a `Panel_sdl`
([`M5GFX.cpp:3256`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/M5GFX.cpp#L3256))
with an explicit M5PaperS3 case:

```c
case board_M5Paper:
case board_M5PaperS3:
case board_M5PaperDIY:
  w = 960;
  h = 540;
  pnl_cfg.offset_rotation = 3;
  p->setColorDepth(lgfx::color_depth_t::grayscale_8bit);
```
— [`M5GFX.cpp:3320-3328`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/M5GFX.cpp#L3320-L3328)

Selected at compile time with `-DM5GFX_BOARD=board_M5PaperS3`, exactly as the shipped example
does for `board_M5Paper`
([`platformio.ini`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/examples/PlatformIO_SDL/platformio.ini)),
and `library.json` declares `"platforms": ["espressif32", "native"]`. **The same drawing code
compiles for the desk and the desktop, upstream, with a correctly-sized greyscale window.**

The honest limit: `Panel_sdl` has no e-ink emulation. `_epd_mode` stays 0 on a non-EPD panel and
`setEpdMode` is a no-op guarded by `if (_epd_mode && epd_mode)`
([`Panel.hpp:98`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/Panel.hpp#L98)).
The simulator gives us geometry, greys and input — not ghosting, not flash, not refresh latency.
Issue #20 still has to build that fidelity layer itself; M5GFX just means it does not also have
to build the drawing API.

### Licence and maintenance

MIT ([`LICENSE`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/LICENSE)),
with vendored LovyanGFX files under FreeBSD/BSD-2 headers. No copyleft obligation.

Last push 2026-08-01, 23 open issues, not archived (GitHub API). v0.2.26 released 2026-07-22.
Commits in the last fortnight include `Add M5PaperDIY board detection and display support` and
`Sync fixes from LovyanGFX develop` — the EPD path is being actively worked on, not merely kept
compiling.

Concrete evidence that the **ESP-IDF** path specifically is maintained:
[issue #160](https://github.com/m5stack/M5GFX/issues/160), "Not draw correctly when using
ESP-IDF on PaperS3, some weird stripes on screen". A contributor bisected it to an ESP-IDF
commit; the maintainer shipped the fix (`beeb74a fix: disable data pins open drain mode`,
2025-11-01) and closed it 2025-11-08. The workaround and its rationale are still in the source
as a comment ([`Bus_EPD.cpp:134-139`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Bus_EPD.cpp#L134-L139)).

### Ecosystem bonus

`M5Unified` (MIT, depends on `m5stack/m5gfx >= 0.2.26`) covers the rest of the board for the
same board enum — power/battery, touch, IMU, RTC pin tables all carry `board_M5PaperS3` cases
(e.g. [`M5Unified.cpp:96`](https://github.com/m5stack/M5Unified/blob/master/src/M5Unified.cpp#L96),
[`Power_Class.cpp:262`](https://github.com/m5stack/M5Unified/blob/master/src/utility/Power_Class.cpp#L262)).
That is relevant to issues #6 (input inventory) and #12 (power budget), not just this one.

---

## Candidate B — epdiy directly

### Does it support this panel and board?

**Panel: yes. Board: no.**

`ED047TC1` is a supported display — 960 × 540, 8-bit bus, 20 MHz
([`displays.c:48-55`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/displays.c#L48-L55)),
as is `ED047TC2` at the same geometry.

But there is **no M5PaperS3 board definition**. `grep -ri "papers3\|m5paper"` across the epdiy
tree returns nothing. The shipped boards are epdiy's own v2–v7 revisions plus two Lilygo boards
([`src/board/`](https://github.com/vroland/epdiy/tree/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/board)).
We would have to write and maintain an `EpdBoardDefinition` out of tree —
`init`/`deinit`/`set_ctrl`/`poweron`/`poweroff`/`set_vcom`/`get_temperature`
([`epd_board.h:26-80`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/epd_board.h#L26-L80)).
The nearest in-tree template, `lilygo_board_s3.c`, is 271 lines, though much of that is PCA9555
+ TPS65185 PMIC handling the PaperS3 does not have (its rails appear to be a plain GPIO enable
on `GPIO 46` plus `PWROFF_PULSE` on `GPIO 44` — inferred from the M5GFX pin map, **not**
verified against a schematic).

This is not hypothetical work, though — third-party PaperS3 board configs exist:
[`clackups/draftling` `components/display/epd_board_papers3.c`](https://github.com/clackups/draftling/blob/main/components/display/epd_board_papers3.c)
(~7 KB, repo active 2026-07),
[`bullno1/instaink`](https://github.com/bullno1/instaink/blob/main/components/device/papers3/src/epd.c),
[`vivalaakam/ai-papers3`](https://github.com/vivalaakam/ai-papers3). M5Stack's own docs still
say "Requires EPDIY library version 2.0.0 or higher"
([M5Stack PaperS3 docs](https://docs.m5stack.com/en/core/PaperS3)) — the official docs are one
integration behind M5GFX's own.

### Waveform and update-mode control

Richer in principle. `enum EpdDrawMode`
([`epdiy.h:50-88`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/epdiy.h#L50-L88))
lists `MODE_DU`, `MODE_GC16`, `MODE_GL16`, `MODE_A2`, `MODE_DU4`, `MODE_GL4`, plus epdiy's own
`MODE_EPDIY_WHITE_TO_GL16` / `BLACK_TO_GL16` / `MODE_EPDIY_MONOCHROME`.

The catch is written into the header: `MODE_A2`, `MODE_GC16_FAST`, `MODE_GL16_FAST`, `MODE_DU4`
and `MODE_GL4` are all annotated **"Not available with default epdiy waveforms."** For our panel
the built-in waveform actually offers five mode types
([`epdiy_ED047TC1.h:28`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/waveforms/epdiy_ED047TC1.h#L28)):

| Mode | Type | Phases | Temp ranges |
|---|---|---|---|
| `MODE_DU` (mono) | 1 | **5** | 1 |
| `MODE_GC16` | 2 | 30 | 1 |
| `MODE_GL16` | 5 | 30 | 1 |
| `MODE_EPDIY_WHITE_TO_GL16` | 16 | 15 | 1 |
| `MODE_EPDIY_BLACK_TO_GL16` | 17 | 15 | 1 |
| `MODE_EPDIY_MONOCHROME` | — | **1** | n/a |

`MODE_EPDIY_MONOCHROME` bypasses the waveform entirely — `frame_count = 1`
([`render.c:124-126`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/render.c#L124-L126))
against a 1 bpp buffer. That is genuinely faster than anything M5GFX offers, and is the strongest
argument for epdiy if the games in #10/#16 turn out to need it.

**Temperature compensation is a real API but is not actually wired up.** epdiy takes a
temperature argument on every draw and selects a phase table by range
(`epd_draw_base` at
[`epdiy.h:531-540`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/epdiy.h#L531-L540),
`epd_ambient_temperature()` at
[`epdiy.h:245`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/epdiy.h#L245)).
Its own doc comment then undercuts it:

> "`@param temperature`: The temperature of the display in °C. **Currently, this is unused by the
> default waveforms** at can be set to room temperature, e.g. 20-25°C."
> — [`epdiy.h:517-519`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/epdiy.h#L517-L519)

Consistent with that, the `ED047TC1` waveform declares `num_temp_ranges = 1` — one table, no
compensation. So epdiy's advantage here is a hook we could populate, not a working feature. The
Lilygo-contributed `ED047TC2` waveform *does* have 7 ranges (DU 15–22 phases, GC16 38–57 phases
depending on temperature,
[`epdiy_ED047TC2.h:51-52`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/waveforms/epdiy_ED047TC2.h#L51-L52)),
at the cost of 272 KB of flash versus 37 KB. Whether TC2's waveform is correct for the PaperS3's
TC1 panel is **unverified**.

### Sub-regions and whether time scales with area

Same answer as M5GFX, from different code. epdiy exposes `epd_hl_update_area(state, mode, temp,
area)` and computes a dirty rect with per-line and per-column masks
([`render.c:465-490`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/render.c#L465-L490)),
and undrawn lines are pushed as zero-filled — no voltage applied
([`render_lcd.c:210-226`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/output_lcd/render_lcd.c#L210-L226)).
But the frame still clocks the whole panel: `lines_total = rounded_display_height()`
([`render.c:170`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/render.c#L170))
and `lcd.display_lines` is fixed at init from the display height, never from the area
([`lcd_driver.c:444`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/output_lcd/lcd_driver.c#L444)).
Horizontal cropping is not supported at all in the S3 path — `render_lcd.c` asserts
`area.width == ctx->display_width && area.x == 0`
([`render_lcd.c:194`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/output_lcd/render_lcd.c#L194)).

Derived floor, again *arithmetic not measurement*: `lcd_res_h = 960/4 = 240` bytes at 20 MHz
gives ~13 µs/line × 544 lines ≈ **7.1 ms/frame**, with no per-frame `vTaskDelay` in the LCD path.
So `MODE_DU` ≈ 36 ms, `MODE_GC16` ≈ 210 ms, `MODE_EPDIY_MONOCHROME` ≈ 7 ms.

One structural limitation: `render_context` is a single file-scope global
([`render.c:45`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/render.c#L45),
populated per call at
[`render.c:156-183`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/render.c#L156-L183))
and `epd_draw_base` blocks until the update completes. **One update at a time, one mode at a
time.** M5GFX's per-pixel LUT-progress model lets the pet animate in `epd_fastest` while the
paper chrome is still settling in `epd_quality`. For our layout — a small animated **silhouette**
inside static **paper chrome** — that difference is not academic.

### Simulator

Poor. epdiy is a C library against ESP-IDF and the Xtensa toolchain; `epd_board.h` unconditionally
includes `<xtensa/core-macros.h>`
([`epd_board.h:11`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/src/epd_board.h#L11))
and the registry manifest declares targets `esp32, esp32s3` only
([`idf_component.yml`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/idf_component.yml)).
There is no host backend. epdiy also has no drawing API worth speaking of beyond framebuffer
blits and a font renderer, so we would be writing the whole 2D layer *and* its host double
ourselves.

### Licence and maintenance

**LGPL-3.0-or-later** (`license: LGPL-3.0-or-later` in
[`idf_component.yml`](https://github.com/vroland/epdiy/blob/7c3078092479d9eabcf1dadcd928b5f3e446284e/idf_component.yml);
README: "Firmware and remaining examples are licensed under the terms of the GNU Lesser GPL
version 3"). For a statically-linked ESP-IDF firmware image, LGPL §4 obligations mean shipping
whatever is needed for a recipient to relink against a modified epdiy. For a personal desk toy
that is an annoyance rather than a blocker, but it is a real difference against M5GFX's MIT, and
it does propagate into whatever we publish.

Maintenance is good: last push 2026-08-01, 9 open issues, v2.1.3 on the
[ESP Component Registry](https://components.espressif.com/components/vroland/epdiy) (uploaded
2026-08-01, ~202 total downloads). ESP-IDF v6.0 support landed 2026-06-13
(`8781397 IDF-6.0 support ported from #463`), and
[issue #450 "Make it possible to compile in ESP-IDF version 6.1"](https://github.com/vroland/epdiy/issues/450)
is **still open** — worth knowing if we plan to track IDF 6.1.

---

## Candidate C — a bare panel driver we write ourselves

Recorded for completeness; the evidence against is the size of what we would be reimplementing.

To match M5GFX's current behaviour we would need: `esp_lcd` i80 bus setup with the S3's
8-bit-mode dummy-byte quirk, CKV/SPV/SPH/OE/LE sequencing with sub-microsecond timing
([`Bus_EPD.cpp:48-109`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Bus_EPD.cpp#L48-L109)),
a per-pixel LUT-progress engine, a double-buffered DMA scanline pipeline, and — the part that is
easy to underestimate — a hand-written Xtensa assembly inner loop for the 2-pixel-per-byte LUT
blit, with a C fallback behind `#if defined(__XTENSA__)` / `#else`
([`Panel_EPD.cpp:599-812`](https://github.com/m5stack/M5GFX/blob/729297d6e3d657ddc1ec5189bac2f2ea68828085/src/lgfx/v1/platforms/esp32/Panel_EPD.cpp#L599-L812)).
That is ~1 250 lines of driver before we have drawn a single **heart**.

And the waveform data is the real wall. E Ink does not publish waveforms for ED047TC1; both
libraries carry hand-derived tables (M5GFX's LUT comments are in Japanese and describe an
empirically-tuned 16-column × N-row grid). Getting these wrong is not a cosmetic bug — see the
durability finding below.

The only honest argument for this option would be that neither library can express something we
need. Nothing in the evidence suggests that.

---

## Aside: a fourth candidate the issue did not list

[`bitbank2/FastEPD`](https://github.com/bitbank2/FastEPD) — Apache-2.0, 205 stars, last push
2026-05-30 — has `BB_PANEL_M5PAPERS3` as an **in-tree** panel constant, 1/2/4 bpp modes,
`fullUpdate()`/`partialUpdate()`, and a `BB_PANEL_VIRTUAL` for host use
([`src/FastEPD.h`](https://github.com/bitbank2/FastEPD/blob/main/src/FastEPD.h)).
Not evaluated in depth. Flagging it so nobody rediscovers it as a surprise in three weeks. It
does not obviously beat M5GFX on any axis we care about, and it has a much smaller drawing API.

---

## Head to head

| | M5GFX | epdiy | Bare driver |
|---|---|---|---|
| M5PaperS3 board in tree | **Yes**, autodetected | No — write your own `EpdBoardDefinition` | n/a |
| Pure ESP-IDF (no Arduino) | **Yes**, `esp_lcd` i80 | **Yes** | Yes |
| ESP-IDF v6 | v6.0 ✓ (explicit branch) | v6.0 ✓, 6.1 open issue | Our problem |
| Fast mono partial | `epd_fast` 8f / `epd_fastest` 5f | `MODE_DU` 5f, mono 1f | — |
| 16-grey full | `epd_quality` 15f | `MODE_GC16` 30f | — |
| Custom waveforms | Supported via `config_detail_t` | Supported via `EpdWaveform` | Mandatory |
| Temperature compensation | **None** | API present but "unused by the default waveforms"; 1 range for ED047TC1 | — |
| Concurrent regions in different modes | **Yes** (per-pixel LUT state) | No (single global context, blocking) | — |
| Time scales with region area | No | No | No |
| Desktop simulator | **Yes**, `Panel_sdl` + PaperS3 case | No host backend | n/a |
| Drawing API | Full LovyanGFX 2D | Framebuffer + font | Ours |
| Licence | **MIT** | LGPL-3.0-or-later | n/a |
| Rest of board (power/IMU/RTC/touch) | M5Unified, same enum | Nothing | Nothing |

---

## Recommendation

**Build on M5GFX as an ESP-IDF managed component (`m5stack/m5gfx`), pinned to an exact version,
behind our own narrow display port.**

Five things decide it, in order:

1. **Upstream migrated off epdiy for this exact panel.** M5GFX's own doc says EPDiy is no longer
   used from v0.2.7. Choosing epdiy means walking back a path the people who own this board
   already walked forward, and maintaining a board definition they deleted.
2. **The simulator comes for free.** `-DM5GFX_BOARD=board_M5PaperS3` gives a 960 × 540 greyscale
   SDL window driven by the same drawing calls. epdiy would leave issue #20 building both a
   drawing API and its host double from scratch.
3. **Both modes we need are in one API**, mono-fast and 16-grey, switchable per draw, with
   per-pixel LUT state so **ambient** and **session** regions can be refreshing in different
   modes at once.
4. **MIT beats LGPL** for firmware we may want to publish.
5. **M5Unified extends the same board enum** to power, touch, IMU and RTC, which four other open
   issues need anyway.

### The trade-offs, stated honestly

- **No temperature compensation.** M5GFX's LUTs are fixed. A cold room will make refreshes
  sluggish and possibly muddy, and we have no software lever. This costs less than it first
  looks, because epdiy's temperature argument is documented as "unused by the default waveforms"
  and its ED047TC1 table has one range — so the alternative does not actually solve it either.
  If greyscale quality drifts seasonally, the fix in either library is authoring custom LUTs,
  and M5GFX's `config_detail_t` makes that a config field rather than a fork.
- **epdiy's `MODE_EPDIY_MONOCHROME` is one frame; M5GFX's floor is five.** If a game needs
  genuinely high frame rates, epdiy has a gear M5GFX does not. Resolve this in the #10 spike
  before committing the art pipeline.
- **We inherit a large dependency for a small need.** M5GFX is a whole 2D graphics library; we
  want a pet, four **hearts**, some **paper chrome** and a **care log**. The mitigation is the
  port, not avoidance.
- **The M5GFX EPD driver is young and being actively changed.** Issue #152's thread shows
  v0.2.11 shipped a control bug that "placed excessive strain on the EPD" and left staining for
  tens of minutes after power-off. Pin an exact version; upgrade deliberately, with a visual
  regression check.
- **Two legacy build-system dependencies** (`register_component()`, `CONFIG_ESP32S3_SPIRAM_SUPPORT`)
  work on IDF v6.0 today but are both on deprecation paths, and the second fails *silently* —
  no panel, no crash. Assert on `getBoard() == board_M5PaperS3` at startup.

### The finding that should worry us most, and it is not about the driver

The M5GFX maintainer, in
[issue #166](https://github.com/m5stack/M5GFX/issues/166) and
[issue #152](https://github.com/m5stack/M5GFX/issues/152):

> "I also have two Paper S3s that I purchased early on, and one of them already had one broken
> line when I first started it up, and **the broken areas have increased over time as I've used
> it.** … On the damaged unit I have, even the undamaged pixels have unstable gradations and tend
> to transition to gray with continued use."

> "I'm not sure why the PaperS3's EPD panel is damaged. Possibilities include… a flaw in the
> M5GFX's control… a flaw in the PaperS3's EPD control circuit… the EPD panel itself may be
> designed to be more susceptible to damage. At this point, all of these are possibilities, but I
> don't know for sure."

Another user in the same thread reported a line dying during normal use. **This is a
refresh-durability risk on an EOL panel, and it is a driver-independent hardware concern that
happens to argue for the design papergotchi already wants**: rare refreshes in **ambient**, the
full budget only in **session**, and the smallest region that does the job — not because small
regions are faster (they are not) but because fewer driven pixels is less cumulative stress.
Issue #9 should treat total driven-pixel-frames, not refresh count, as the quantity to budget.

### Shape of the port

The port should be narrow enough that swapping to epdiy or FastEPD is a week, not a rewrite —
which is the real insurance policy against everything above:

- `clear(region)`, `drawGrey4(region, buffer)`, `present(region, mode)` where `mode` is *our*
  enum (`Ambient`, `Reveal`, `Animate`, `Settle`), not `epd_mode_t`
- `busy()` / `waitIdle()` — M5GFX's update is asynchronous on its own task, so the port must
  expose that rather than pretend `present` blocks
- Region coordinates snapped to even X (4 bpp buffer) — bake that into the port so layout code
  never has to know

Keep LovyanGFX types out of `core/`. The host build then differs only in which display port is
linked, and #20's e-ink fidelity work (ghosting, flash, latency) lives in the host
implementation of that port rather than being bolted onto `Panel_sdl`.

---

## What remains uncertain

Marked plainly, because a flagged unknown is worth more here than a confident guess.

1. **Nothing here was run on hardware.** Every timing figure is arithmetic derived from source
   constants. The `vTaskDelay(1)` per frame in particular could make `epd_fastest` anywhere from
   ~42 ms to ~90 ms depending on `CONFIG_FREERTOS_HZ`. Measure before #10 or #13 commit to a
   frame rate.
2. **Nothing was compiled against ESP-IDF v6.** The v6 evidence is a CMake branch, an alias in
   `sdkconfig.rename.esp32s3`, and `register_component()` existing on `release/v6.0`. Whether
   `m5stack/m5gfx` v0.2.26 actually builds clean on IDF 6.0.2 for `esp32s3` is unverified. Do
   this first; it is an afternoon and it invalidates everything if it fails.
3. **The panel identity is not fully pinned.** M5Stack's docs and store say `ED047TC1`. epdiy has
   both `ED047TC1` and `ED047TC2` at 960 × 540 with quite different waveforms. M5GFX's native
   driver names neither — it just ships LUTs. If we ever want epdiy's temperature ranges, which
   waveform is correct for this panel is an open question.
4. **No schematic was read.** The M5PaperS3 power rails are inferred from M5GFX's pin usage
   (`PWR` on GPIO 46, `PWROFF_PULSE` on GPIO 44) and the absence of PMIC code. I did not locate an
   official schematic PDF. Anyone writing an epdiy board definition needs this; the third-party
   configs listed above are a shortcut, not a substitute.
5. **The panel-damage reports are anecdotal.** Three data points from GitHub issues, with the
   maintainer explicitly unable to attribute cause. Whether this is a driver bug since fixed, a
   board flaw, or panel-lot variance is unknown. It should shape the refresh budget as a
   precaution, not be reported as an established defect.
6. **The 16-grey claim in `epd_quality` was not measured.** The LUT drives all 16 columns, and
   the panel is specified for 16 levels, but how many are *visually distinct* on this panel — and
   therefore how many greys the **paper chrome** can actually use — is an art-pipeline question
   for #17 that only hardware answers.
7. **FastEPD was not evaluated in depth.** Panel constant and mode enums were read; its waveform
   quality, update-time behaviour and `BB_PANEL_VIRTUAL` host story were not.
8. **The relationship between region area and panel stress is assumed, not verified.** The claim
   "fewer driven pixels is less cumulative stress" is physically plausible and matches the
   maintainer's remarks, but nothing in either codebase measures or documents it.
