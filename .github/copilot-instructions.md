# CYD-78D27C — Repo-Specific Instructions

> Generic conventions, hardware, color thresholds, build commands, and animation details
> are in the CYD skill (`~/.copilot/skills/CYD/SKILL.md`). This file covers only
> 78D27C-specific architecture, pages, clock/scale rules, demo mode, and variable references.

---

## Pages

| Index | LVGL Page ID | Content |
|-------|-------------|---------|
| 0 | `clock_page` | Full-screen analog clock — 3 LVGL scales |
| 1 | `dc_page` | PV Power / Battery SOC / Battery Power+Time cycling |
| 2 | `ac_page` | Apparent Power / Ecoflow / A/C 1F↔3F cycling |
| — | `doorbell_page` | Full-screen alert overlay — not in rotation, HA-triggered |

Screens 1 & 2: three vertically-stacked rows of 320×80 px each.

---

## Interval State Machines

**1 s** — screen rotation + slot cycling:
- Decrements `g_screen_secs`; on expiry calls `lv_scr_load_anim`. Skipped entirely while `g_doorbell_active == true`.
- Decrements `g_slot_secs`; on expiry advances both the DC battery slot and AC A/C slot.

**250 ms** — animations + doorbell:
- Drives battery icon animation (states 0–3) and solar icon animation (states 0–2).
- Handles doorbell flash: random color from `DB_COLORS[]` → display background + RGB LED. Decrements `g_doorbell_ticks`; on zero dismisses alert, turns off LED, restores backlight.
- **No demo-mode guard** — animations always run even in layout preview mode.

---

## Sensor → Display Flow

Each HA sensor `on_value` lambda:
1. Caches value into matching `g_*_val` global.
2. Checks `g_lvgl_ready` **first**, then `g_demo_mode` (second, where applicable), then updates display.

Guard order is critical — never reverse it.

---

## LVGL Clock Hands

Use `lvgl.indicator.update` (ESPHome native action). Raw `lv_meter_set_indicator_value()` is not in lambda scope → compile error.

Value mapping (all 0–60 scale):
- Hour = `(tm_hour % 12) * 5 + tm_min / 12`
- Minute = `tm_min` / Second = `tm_sec`
- Tail (counterweight, 180° opposite) = `(hand_value + 30) % 60`

---

## LVGL Scale Rules

- All `scales:` items at the **same indent level** — never nest Scale 2 inside Scale 1.
- Minimum `count: 2` (ESPHome rejects `count: 1`).
- Use `length:` not the deprecated `r_mod`.
- `styles:` on indicators accepts string style IDs only — no inline `part:/bg_color:` dicts.

---

## Value + Unit Formatting

No space between value and unit:
```cpp
snprintf(buf, sizeof(buf), "%.2fkW", val);  // ✓
snprintf(buf, sizeof(buf), "%.2f kW", val); // ✗
```

---

## Font Glyph Sets

| Font ID | Size | Glyph set |
|---------|------|-----------|
| `value_font` | 65 | `"0123456789.:-AMPWkV%"` |
| `label_font` | 12 | alphanumerics + `. % / + -` |
| `clock_num_font` | 30 | `"12369"` |
| `doorbell_font` | 100 | `"BDELOR"` (covers "DOOR" + "BELL") |

---

## Demo / Layout Preview Mode

`switch.layout_preview` sets `g_demo_mode = true` — freezes all labels with max-width placeholders.

Guard in scripts:
```cpp
if (!id(g_lvgl_ready)) return;   // ALWAYS first
if (id(g_demo_mode)) return;     // second
```

250ms animation interval has **no** demo guard — animations must run in preview mode.
Force `bstart = 3` in demo mode for battery animation (at 100% SOC `bstart=0` would stall animation).

---

## Doorbell Overlay (78D27C-specific — full screen + LED)

Unlike E713B0 (LED only), 78D27C shows a full LVGL `doorbell_page` overlay:
- Activation: saves backlight to `g_saved_brightness` → sets 100% brightness → sets LED red → loads `doorbell_page` with `LV_SCR_LOAD_ANIM_NONE`.
- 250ms tick: picks random color from `DB_COLORS[]` → sets both `doorbell_page` background AND RGB LED to same color simultaneously.
- Dismiss: RGB LED off → restore backlight → return to current rotation screen (`pages[g_screen_idx]`).
- 1s rotation timer still ticks during alert but `lv_scr_load_anim` is skipped.
- Subsequent presses while active are ignored.

`doorbell_font` glyphs: `"BDELOR"` — covers "DOOR" + "BELL" exactly.

---

## DC Page Row 2 (intentional)

Battery icon container uses `height: 240` — the icon backdrop intentionally spans rows 2 and 3 visually. Do **not** fix this to 80.

---

## Screen Enable Fallback

If all three screen-enable switches are OFF, `g_screen_idx` is forced to 0 (clock always shows as fallback).

---

## Globals Reference

| ID | Purpose |
|----|---------|
| `g_lvgl_ready` | LVGL gate flag — set in `on_boot -10` |
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

## Daily Restart HA Controls

| Entity | ID | Range / Type | Default |
|--------|----|-------------|---------|
| Daily Restart Hour | `daily_restart_hour` | 0–23 hr | 3 |
| Daily Restart Enabled | `daily_restart_enabled` | switch | **OFF** |

Guard: `if (!id(daily_restart_enabled).state) return;` — first line of the `on_time` lambda. Fires top of every hour; checks PHT hour via `gmtime_r`.

---

## Substitutions (color thresholds — edit here, not inline)

| Group | Substitutions | Defaults |
|-------|--------------|----------|
| Solar Power (W) | `solar_color_green/blue/yellow` | `3600` / `2000` / `900` |
| Home Apparent Power (VA) | `home_color_green/blue/yellow` | `1000` / `2000` / `3000` |
| Load 1 Ecoflow (W) | `load1_color_green/blue/yellow` | `140` / `200` / `400` |
| Load 2 A/C 1F (W) | `load2_color_green/blue/yellow` | `600` / `740` / `900` |
| Load 3 A/C 3F (W) | `load3_color_green/blue/yellow` | `80` / `900` / `1000` |
| Battery SOC (%) | `batt_color_green/blue/yellow` | `80` / `50` / `25` |
| Battery glyph (%) | `batt_thresh_full/5bar/4bar/3bar/2bar/1bar/alert` | see substitutions block |
| Battery Power (W) | `batt_pwr_green/blue` | `3600` / `900` |
| Battery time charging (h) | `batt_chg_green_h/blue_h/yellow_h` | `2` / `4` / `6` |
| Battery time discharging (h) | `batt_dis_orange_h/yellow_h/blue_h` | `3` / `6` / `9` |
