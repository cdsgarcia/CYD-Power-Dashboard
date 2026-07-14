# Copilot Instructions — CYD Power Dashboard

## Commands

```bash
esphome run cyd-78d27c.yaml       # flash via USB (first time)
esphome upload cyd-78d27c.yaml    # OTA update
esphome config cyd-78d27c.yaml    # validate config (no compile)
esphome compile cyd-78d27c.yaml   # full compile check without flashing
```

There are no unit tests. Validation is `esphome config` + compile.

---

## Architecture

Single ESPHome YAML (`cyd-78d27c.yaml`) targeting an ESP32 (ILI9341 240×320, displayed landscape via LVGL 270° rotation → effective 320×240).

### Screens

| Index | Page ID      | Content                                         |
|-------|-------------|--------------------------------------------------|
| 0     | `clock_page` | Full-screen analog clock (3 LVGL scales)        |
| 1     | `dc_page`    | PV Power / Battery SOC / Battery Power+Time     |
| 2     | `ac_page`    | Apparent Power / Ecoflow / A/C 1F↔2F cycling   |
| —     | `doorbell_page` | Overlay alert — not in rotation, triggered by HA `input_button` |

Screen 1 and 2 use three vertically-stacked rows of 320×80 each with dividers at y=0/80/160/239.

### State Machines

- **Screen cycling** — 1 s interval; `g_screen_secs` countdown; calls `lv_scr_load_anim` with HA-selectable style. Skipped entirely while `g_doorbell_active == true`.
- **Slot cycling** — same 1 s interval; `g_slot_secs` shared counter advances both the DC battery slot (row 3) and AC A/C slot (row 3).
- **Animation tick** — 250 ms interval; drives battery icon ping-pong (states 0–3), solar icon cycle (states 0–2), and doorbell flash/countdown.

### Globals Naming Convention

| Prefix | Purpose |
|--------|---------|
| `g_screen_*` | Screen rotation state |
| `g_slot_secs` | Shared slot-cycle countdown |
| `g_batt_*` | Battery slot values and animation |
| `g_solar_*` | Solar icon animation |
| `g_ac_*` / `g_load*_val` | AC slot and load caches |
| `g_home_val` | Apparent power cache |
| `g_demo_mode` | Layout preview flag |
| `g_doorbell_*` | Doorbell overlay state |
| `g_lvgl_ready` | Gated flag; all LVGL lambdas check this first |

All globals use `restore_value: false` unless explicitly storing user preference across reboots.

### Sensor → Display Flow

Sensor entity IDs are wired via `substitutions` at the top of the YAML. Each HA sensor has an `on_value` lambda that:
1. Caches the reading into the corresponding `g_*_val` global.
2. Calls the relevant display script (`update_batt_display`, `update_ac_slot_display`, etc.).

Sensor callbacks that touch LVGL labels are wrapped with a demo-mode guard:
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
Always compute Philippine Time (UTC+8) with raw epoch math — never use `localtime()` or a TZ environment variable:
```cpp
time_t pht = now.timestamp + (8 * 3600);
struct tm t;
gmtime_r(&pht, &t);
```

### LVGL Clock Hands
Use `lvgl.indicator.update` (ESPHome native action). The raw C API `lv_meter_set_indicator_value()` is **not** available inside ESPHome lambdas.

Clock hand value mapping (all on a 0–60 scale):
- Hour  = `(tm_hour % 12) * 5 + tm_min / 12`
- Minute = `tm_min`
- Second = `tm_sec`
- Tail (counterweight) = `(hand_value + 30) % 60`

### LVGL Scale Rules
- All `scales:` list items must be at the **same indent level** — never nest Scale 2/3 inside Scale 1's `indicators:`.
- Minimum tick count: `count: 2` (ESPHome rejects `count: 1`).
- `r_mod` is deprecated; use `length:` (absolute px from center).
- `styles:` under a scale indicator accepts only string style IDs — inline `part:/bg_color:` dicts cause a parse error. Use an overlay `obj` widget instead.

### Value + Unit Formatting
No spaces between value and unit in any `snprintf`/`str_sprintf` format string:
```c
snprintf(buf, sizeof(buf), "%.2fkW", val);   // ✓ "2.50kW"
snprintf(buf, sizeof(buf), "%.2f kW", val);  // ✗
```

### Font Glyph Sets
Fonts are declared with exact glyph lists to minimise flash size. When adding new display text:
- Check `value_font` glyphs: `"0123456789.:-AMPWkV%"` — add characters here if needed.
- Check `label_font` (12 px Roboto): covers alphanumerics + `. % / + -`.
- `doorbell_font` glyphs: `"BDELOR"` (covers "DOOR" and "BELL" only).
- Battery/solar icons use Unicode code-point strings (`"\ue1a4"` etc.) — always reference the comment next to the glyph for the icon name.

### Demo / Layout Preview Mode
`switch.layout_preview` sets `g_demo_mode = true` and freezes all labels with max-width placeholders. Guard pattern in scripts:
```cpp
if (id(g_demo_mode)) return;
```
The 250 ms animation interval has **no** demo guard — animations must keep running in preview mode.

In demo mode, force `bstart = 3` for the battery animation (real 100% SOC makes `bstart = 0` and the animation stalls).

### Doorbell Overlay
- Triggered via `text_sensor` watching `input_button.doorbell` — the button's state changes to a timestamp on each press, firing `on_value`.
- While `g_doorbell_active` is true: screen rotation timer still ticks but the `lv_scr_load_anim` call is skipped.
- Brightness is saved to `g_saved_brightness` on activation and restored on dismiss.
- Subsequent presses while already active are ignored.
- Use `ESP_LOGW` for press and dismiss log entries.

### DC Page Row 2 Layout (intentional)
`height: 240` on the battery icon container is intentional — the 150 px battery icon backdrop spans both rows 2 and 3 visually. Do not "fix" this to 80.

### Screen Enable Fallback
If all three screen-enable switches are OFF, `g_screen_idx` is forced to 0 (clock) as a fallback — the clock is always shown.

---

## HA Entity Controls (configurable at runtime)

Sensor entity substitutions, all number/switch/select/text controls, and the doorbell entity ID are defined in `substitutions:` at the top of the YAML — change them there, not inline.

Key defaults:
- Screen rotation: 10 s (`screen_transition_secs`)
- Slot cycle: 2 s (`slot_cycle_secs`)
- Daily restart: 3 AM PHT (`daily_restart_hour`)
- Doorbell duration: 5 s (`doorbell_duration_secs`)
