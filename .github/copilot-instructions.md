# Copilot Instructions — CYD Power Dashboard (78D27C)

## Commands

```bash
python -m esphome config  cyd-78d27c.yaml   # validate (fast)
python -m esphome compile cyd-78d27c.yaml   # full compile check
python -m esphome run     cyd-78d27c.yaml   # build + flash (USB)
python -m esphome upload  cyd-78d27c.yaml   # OTA flash
python -m esphome logs    cyd-78d27c.yaml   # live serial log
```

Use `python -m esphome` — bare `esphome` may not be on PATH on Windows.

---

## Hardware

| Function | Pins |
|----------|------|
| TFT SPI (display) | CLK=GPIO14, MOSI=GPIO13, MISO=GPIO12, CS=GPIO15, DC=GPIO2 |
| Backlight (LEDC) | GPIO21 |
| RGB LED back (LEDC, common anode — active LOW) | R=GPIO4, G=GPIO16, B=GPIO17 |

---

## Architecture

Single YAML (`cyd-78d27c.yaml`) targeting an ESP32 with ILI9341 240×320 display rendered landscape via LVGL 270° rotation (effective 320×240).

### Pages

| Index | LVGL Page ID | Content |
|-------|-------------|---------|
| 0 | `clock_page` | Full-screen analog clock — 3 LVGL scales |
| 1 | `dc_page` | PV Power / Battery SOC / Battery Power+Time cycling |
| 2 | `ac_page` | Apparent Power / Ecoflow / A/C 1F↔3F cycling |
| — | `doorbell_page` | Overlay alert — not in rotation, HA-triggered |

Screens 1 & 2 use three vertically-stacked rows of 320×80 px each.

### Intervals

**1 s** — screen rotation + slot cycling:
- Decrements `g_screen_secs`; on expiry calls `lv_scr_load_anim`. Skipped entirely while `g_doorbell_active == true`.
- Decrements `g_slot_secs`; on expiry advances both the DC battery slot and AC A/C slot.

**250 ms** — animations + doorbell:
- Drives battery icon animation (states 0–3) and solar icon animation (states 0–2).
- Handles doorbell flash: picks random color from `DB_COLORS[]`, sets both the display background and RGB LED to the same color. Decrements `g_doorbell_ticks`; on zero dismisses the alert, turns off LED, restores backlight.
- **No demo-mode guard** — animations always run even in layout preview mode.

### Sensor → Display Flow

Each HA sensor `on_value` lambda:
1. Caches value into matching `g_*_val` global.
2. Checks `g_lvgl_ready` (first), `g_demo_mode` (second — where applicable), then updates display.

---

## Key Conventions

### Guard Order (critical)
```cpp
if (!id(g_lvgl_ready)) return;   // ALWAYS first
if (id(g_demo_mode)) return;     // second, only where applicable
```
Never reverse this order.

### PHT Time
Always use raw UTC epoch + 8 h offset — never `localtime()` or TZ env var:
```cpp
time_t pht = now.timestamp + (8 * 3600);
struct tm t;
gmtime_r(&pht, &t);
```

### Value + Unit Formatting
No space between value and unit:
```cpp
snprintf(buf, sizeof(buf), "%.2fkW", val);  // ✓
snprintf(buf, sizeof(buf), "%.2f kW", val); // ✗
```

### Font Glyph Sets
Adding a character not in a font's `glyphs` list causes a crash or garbled render.

| Font ID | Size | Glyph set |
|---------|------|-----------|
| `value_font` | 65 | `"0123456789.:-AMPWkV%"` |
| `label_font` | 12 | alphanumerics + `. % / + -` |
| `clock_num_font` | 30 | `"12369"` |
| `doorbell_font` | 100 | `"BDELOR"` (covers "DOOR" + "BELL") |

### LVGL Clock Hands
Use `lvgl.indicator.update` (ESPHome native). Raw `lv_meter_set_indicator_value()` is not in lambda scope → compile error.

Value mapping (all 0–60 scale):
- Hour = `(tm_hour % 12) * 5 + tm_min / 12`
- Minute = `tm_min` / Second = `tm_sec`
- Tail (counterweight) = `(hand_value + 30) % 60`

### LVGL Scale Rules
- All `scales:` items at the same indent level — never nest.
- Minimum `count: 2` (ESPHome rejects `count: 1`).
- Use `length:` not the deprecated `r_mod`.
- `styles:` on indicators accepts string style IDs only — no inline `part:/bg_color:` dicts.

### Demo / Layout Preview Mode
`switch.layout_preview` sets `g_demo_mode = true` and freezes all labels. The 250 ms animation interval has no demo guard — animations must keep running. Force `bstart = 3` in demo mode (at 100% SOC `bstart = 0` would stall the animation).

### Doorbell Overlay
- Triggered by `text_sensor` watching `${doorbell_entity}` — state changes to a timestamp on each press.
- Activation: saves backlight brightness → sets backlight to 100% → sets RGB LED to red → loads `doorbell_page`.
- 250 ms tick: picks random color from `DB_COLORS[]` (8 colors — all have at least one channel at 0xFF for maximum brightness), applies to both display background and RGB LED simultaneously.
- Dismiss (countdown reaches 0): turns off RGB LED (unconditional) → restores backlight → returns to current rotation screen.
- Subsequent presses while active are ignored.
- `doorbell_led_enabled` switch (HA, default ON) — disables RGB LED without affecting the display alert.

### DC Page Row 2 (intentional)
Battery icon container uses `height: 240` — the icon backdrop intentionally spans rows 2 and 3. Do not "fix" this to 80.

### Screen Enable Fallback
If all three screen-enable switches are OFF, `g_screen_idx` is forced to 0 (clock always shows as fallback).

---

## Globals Reference

| ID | Purpose |
|----|---------|
| `g_lvgl_ready` | Gate flag — set `true` in `on_boot -10`; all LVGL lambdas check first |
| `g_demo_mode` | Layout preview flag |
| `g_screen_idx` / `g_screen_secs` | Screen rotation state |
| `g_slot_secs` | Shared slot countdown (DC batt slot + AC A/C slot) |
| `g_batt_slot` / `g_batt_*_val` | Battery slot and value caches |
| `g_batt_anim_state` / `g_batt_anim_step` | Battery icon animation |
| `g_solar_val` / `g_solar_anim_state` / `g_solar_anim_step` | Solar icon animation |
| `g_ac_slot_idx` | AC slot: 0=A/C 1F (load2), 1=A/C 3F (load3) |
| `g_load1_val` / `g_load2_val` / `g_load3_val` | Load value caches |
| `g_home_val` | Apparent power cache |
| `g_doorbell_active` / `g_doorbell_ticks` / `g_doorbell_last_state` | Doorbell state |
| `g_saved_brightness` | Backlight level saved before doorbell, restored after |

---

## Substitutions (edit here, never inline)

All entity IDs, device name, and color/threshold values live in `substitutions:` at the top.

| Group | Substitutions | Defaults |
|-------|--------------|----------|
| Solar Power (W) | `solar_color_green/blue/yellow` | `3600` / `2000` / `900` |
| Home Apparent Power (VA) | `home_color_green/blue/yellow` | `1000` / `2000` / `3000` |
| Load 1 Ecoflow (W) | `load1_color_green/blue/yellow` | `140` / `200` / `400` |
| Load 2 A/C 1F (W) | `load2_color_green/blue/yellow` | `600` / `740` / `900` |
| Load 3 A/C 3F (W) | `load3_color_green/blue/yellow` | `80` / `900` / `1000` |
| Battery SOC (%) | `batt_color_green/blue/yellow` | `80` / `50` / `25` |
| Battery glyph (%) | `batt_thresh_full/5bar/4bar/3bar/2bar/1bar/alert` | `98/85/70/55/40/25/10` |
| Battery Power (W) | `batt_pwr_green/blue` | `3600` / `900` |
| Battery time — charging (h) | `batt_chg_green_h/blue_h/yellow_h` | `2` / `4` / `6` |
| Battery time — discharging (h) | `batt_dis_orange_h/yellow_h/blue_h` | `3` / `6` / `9` |

---

## Color Threshold Quick Reference

**Solar Power** (W): > 3600 🟢 / > 2000 🔵 / > 900 🟡 / ≤ 900 🟠

**Home Apparent Power** (VA): < 1000 🟢 / < 2000 🔵 / < 3000 🟡 / ≥ 3000 🟠

**AC Load Slots** (W):

| Load | 🟢 Green | 🔵 Blue | 🟡 Yellow | 🟠 Orange |
|------|---------|---------|----------|---------|
| A/C 1F (load2) | < 600 | < 740 | < 900 | ≥ 900 |
| A/C 3F (load3) | < 80 | < 900 | < 1000 | ≥ 1000 |
| Ecoflow (load1) | < 140 | < 200 | < 400 | ≥ 400 |

**Battery SOC** (%): ≥ 80 🟢 / ≥ 50 🔵 / ≥ 25 🟡 / < 25 🟠
`icon_battery` state=0 (static) uses `batt_color_*` substitutions — always matches `val_battery`.

**Battery Power** (W): ≥ 3600 🟢 / ≥ 900 🔵 / > 0 🟡 / = 0 🩶 Grey `0x888888` / < 0 🟠


## Commands

```bash
esphome run cyd-78d27c.yaml       # flash via USB (first time)
esphome upload cyd-78d27c.yaml    # OTA update
esphome config cyd-78d27c.yaml    # validate YAML without compiling
esphome compile cyd-78d27c.yaml   # full compile check without flashing
```

No unit tests. Validation = `esphome config` (fast) → `esphome compile` (full).

---

## Architecture

Single ESPHome YAML (`cyd-78d27c.yaml`, ~2100 lines) targeting an ESP32 with ILI9341 240×320 display rendered landscape via LVGL 270° rotation (effective 320×240).

### Pages

| Index | LVGL Page ID    | Content                                             |
|-------|-----------------|-----------------------------------------------------|
| 0     | `clock_page`    | Full-screen analog clock — 3 LVGL scales            |
| 1     | `dc_page`       | PV Power / Battery SOC / Battery Power+Time cycling |
| 2     | `ac_page`       | Apparent Power / Ecoflow / A/C 1F↔3F cycling       |
| —     | `doorbell_page` | Overlay alert, not in rotation, HA-button triggered |

Screens 1 & 2 use three vertically-stacked rows of 320×80 px each, with dividers at y = 0 / 80 / 160 / 239.

### Interval State Machines

Two intervals drive all runtime behaviour:

**1 s interval** — screen rotation + slot cycling:
- Decrements `g_screen_secs`; on expiry calls `lv_scr_load_anim`. Skipped entirely while `g_doorbell_active == true`.
- Decrements `g_slot_secs`; on expiry advances both the DC battery slot (row 3) and AC A/C slot (row 3).

**250 ms interval** — animations + doorbell:
- Drives battery icon ping-pong (states 0–3) and solar icon cycle (states 0–2).
- Handles doorbell flash color changes and countdown-to-dismiss.
- **No demo-mode guard** — animations must run even in layout preview mode.

### Sensor → Display Flow

Entity IDs are declared once in `substitutions:` at the top. Each HA sensor has an `on_value` lambda that:
1. Caches the value into the matching `g_*_val` global.
2. Calls the relevant display script (`update_batt_display`, `update_ac_slot_display`, etc.).

Scripts that touch LVGL labels check `g_demo_mode` first and return early if true. Sensor `on_value` callbacks use an `if:` condition wrapper:

```yaml
on_value:
  - if:
      condition:
        lambda: 'return !id(g_demo_mode);'
      then:
        - lvgl.label.update: ...
```

---

## Key Conventions

### PHT Time
Always compute Philippine Time (UTC+8) with raw epoch arithmetic — never use `localtime()` or rely on a TZ environment variable:

```cpp
time_t pht = now.timestamp + (8 * 3600);
struct tm t;
gmtime_r(&pht, &t);
```

### LVGL Clock Hands
Use `lvgl.indicator.update` (ESPHome native action). The raw C API `lv_meter_set_indicator_value()` is **not** in lambda scope and will cause a compile error.

Clock hand value mapping (all 0–60 scale):
- Hour  = `(tm_hour % 12) * 5 + tm_min / 12`
- Minute = `tm_min`
- Second = `tm_sec`
- Tail (counterweight, 180° opposite) = `(hand_value + 30) % 60`

### LVGL Scale Rules
- All `scales:` list items must be at the **same indent level** — never nest Scale 2 or 3 inside Scale 1's `indicators:`.
- Minimum tick count: `count: 2` (ESPHome rejects `count: 1`).
- `r_mod` is deprecated — use `length:` (absolute px from center).
- `styles:` on a scale indicator accepts only string style IDs. Inline `part:/bg_color:` dicts cause a YAML parse error. Use an overlay `obj` widget for custom styling instead.

### Value + Unit Formatting
No space between value and unit in any `snprintf` / `str_sprintf` format string:

```cpp
snprintf(buf, sizeof(buf), "%.2fkW", val);  // ✓  "2.50kW"
snprintf(buf, sizeof(buf), "%.2f kW", val); // ✗
```

### Font Glyph Sets
Fonts use exact glyph lists to minimise flash. When adding new display text:

| Font ID          | Size | Glyph set |
|------------------|------|-----------|
| `value_font`     | 65   | `"0123456789.:-AMPWkV%"` |
| `label_font`     | 12   | alphanumerics + `. % / + -` |
| `clock_num_font` | 30   | `"12369"` |
| `doorbell_font`  | 100  | `"BDELOR"` (covers "DOOR" + "BELL") |

Battery and solar icons use Unicode code-point strings (`"\ue1a4"` etc.). Always check the inline comment for the icon name before editing.

### Globals Naming
| Prefix | Covers |
|--------|--------|
| `g_screen_*` | Screen rotation state |
| `g_slot_secs` | Shared slot-cycle countdown |
| `g_batt_*` | Battery slot values and icon animation |
| `g_solar_*` | Solar icon animation |
| `g_ac_*` / `g_load*_val` | AC slot and load value caches |
| `g_home_val` | Apparent power cache (used by layout_preview restore) |
| `g_demo_mode` | Layout preview flag |
| `g_doorbell_*` | Doorbell overlay state |
| `g_lvgl_ready` | Gate flag — all LVGL lambdas check this before writing |

All globals use `restore_value: false` unless explicitly persisting a user preference across reboots.

### Demo / Layout Preview Mode
`switch.layout_preview` sets `g_demo_mode = true` and freezes all labels with max-width placeholders. Guard in scripts:

```cpp
if (id(g_demo_mode)) return;
```

The 250 ms animation interval has **no** demo guard — battery and solar animations must keep running in preview mode. In demo mode, force `bstart = 3` for the battery animation (at 100% SOC `bstart` would be 0 and the animation stalls).

### Doorbell Overlay
- Triggered by a `text_sensor` watching `input_button.doorbell` — the button's state changes to a timestamp on each press, firing `on_value`.
- While `g_doorbell_active` is true the 1 s screen rotation timer still ticks, but the `lv_scr_load_anim` call is skipped so the alert is not cut short.
- Backlight level is saved to `g_saved_brightness` on activation and restored on dismiss.
- Subsequent presses while already active are ignored.
- Press and dismiss events are logged with `ESP_LOGW`.

### DC Page Row 2 (intentional)
The battery icon container on DC page uses `height: 240` — the 150 px battery icon backdrop intentionally spans both rows 2 and 3 visually. Do not "fix" this to 80.

### Screen Enable Fallback
If all three screen-enable switches are OFF, `g_screen_idx` is forced to 0 (clock) — the clock is always shown as a fallback.

### Substitutions (change here, not inline)
All sensor entity IDs, the doorbell entity, the device name, and all color/threshold values live in `substitutions:` at the top of the YAML. Edit them there only.

Color threshold substitutions available:

| Group | Substitutions | Default values |
|-------|--------------|----------------|
| Solar Power (W) | `solar_color_green/blue/yellow` | `3600` / `2000` / `900` |
| Home Apparent Power (VA) | `home_color_green/blue/yellow` | `1000` / `2000` / `3000` |
| Load 1 Ecoflow (W) | `load1_color_green/blue/yellow` | `140` / `200` / `400` |
| Load 2 A/C 1F (W) | `load2_color_green/blue/yellow` | `600` / `740` / `900` |
| Load 3 A/C 3F (W) | `load3_color_green/blue/yellow` | `80` / `900` / `1000` |
| Battery SOC (%) | `batt_color_green/blue/yellow` | `80` / `50` / `25` |
| Battery glyph SOC (%) | `batt_thresh_full/5bar/4bar/3bar/2bar/1bar/alert` | `98/85/70/55/40/25/10` |
| Battery Power (W) | `batt_pwr_green/blue` | `3600` / `900` |
| Battery Time Estimate (h) | `batt_chg_green/blue/yellow_h` | `2` / `4` / `6` |
| Battery Time Estimate (h) | `batt_dis_orange/yellow/blue_h` | `3` / `6` / `9` |

### Guard order convention
All `on_value` lambdas must check guards in this exact order:
```cpp
if (!id(g_lvgl_ready)) return;   // ALWAYS first
if (id(g_demo_mode)) return;     // second (only where applicable)
```
Never check `g_demo_mode` before `g_lvgl_ready`.

---

## Color Threshold Quick Reference

All color thresholds are `substitutions:` — edit there, never inline.

**Solar Power** (W) — `solar_color_green/blue/yellow`
| Range | Color |
|-------|-------|
| > 3600 | 🟢 Green |
| > 2000 | 🔵 Blue |
| > 900  | 🟡 Yellow |
| ≤ 900  | 🟠 Orange (includes 0 W) |

**Home Apparent Power** (VA) — `home_color_green/blue/yellow`
| Range | Color |
|-------|-------|
| < 1000 | 🟢 Green |
| < 2000 | 🔵 Blue |
| < 3000 | 🟡 Yellow |
| ≥ 3000 | 🟠 Orange |

**AC Slot (A/C 1F ↔ A/C 3F)** (W) — per-load `load2/3_color_green/blue/yellow`

| Load | Green below | Blue below | Yellow below | Orange |
|------|-------------|------------|--------------|--------|
| A/C 1F (load2) | 600 W | 740 W | 900 W | ≥ 900 W |
| A/C 3F (load3) | 80 W  | 900 W | 1000 W | ≥ 1000 W |

**Ecoflow River 3 / Load 1** (W) — `load1_color_green/blue/yellow`
| Range | Color |
|-------|-------|
| < 140 | 🟢 Green |
| < 200 | 🔵 Blue |
| < 400 | 🟡 Yellow |
| ≥ 400 | 🟠 Orange |

**Battery SOC** (%) — `batt_color_green/blue/yellow`
| Range | Color |
|-------|-------|
| ≥ 80 | 🟢 Green |
| ≥ 50 | 🔵 Blue |
| ≥ 25 | 🟡 Yellow |
| < 25 | 🟠 Orange |

**Battery Power** (W, slot 0) — `batt_pwr_green` / `batt_pwr_blue`
| Range | Color |
|-------|-------|
| ≥ 3600 | 🟢 Green (fast charge) |
| ≥ 900  | 🔵 Blue (moderate charge) |
| > 0    | 🟡 Yellow (slow / trickle) |
| = 0    | 🩶 Grey `0x888888` (idle — matches cloud icon) |
| < 0    | 🟠 Orange (any discharge) |

**Battery icon state=0 (static)**: color computed from `batt_color_*` substitutions — always matches `val_battery`. Never use a glyph-index color array for state=0.
