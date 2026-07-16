# CYD Power Dashboard

An ESPHome LVGL dashboard for the **Cheap Yellow Display (CYD)** ESP32 with ILI9341 touchscreen (240×320, displayed in landscape via 270° rotation). Designed for real-time solar/battery/AC power monitoring with Home Assistant.

---

## Screens

### Screen 0 — Analog Clock
Full-screen analog clock with:
- Hour, minute, and second hands with counterweight tails
- Second hand always rendered on top (dedicated LVGL scale)
- Bold hour-mark ticks at every 5-minute position
- Cardinal labels (12 / 3 / 6 / 9) in large font
- Red center pivot dot

### Screen 1 — DC Page
Three vertically-stacked rows (320×80 each):

| Row | Content |
|-----|---------|
| 1 | **PV Solar Power** — animated sun/moon/cloud icon + kW value |
| 2 | **Battery SOC** — large animated battery icon + % value |
| 3 | **Battery Power / Time Estimate** *(cycling)* — power in W/kW or estimated charge/discharge time |

### Screen 2 — AC Page
Three vertically-stacked rows (320×80 each):

| Row | Content |
|-----|---------|
| 1 | **Apparent Power** — home icon + VA/kVA value |
| 2 | **Ecoflow / Load 1** — fixed, always shown |
| 3 | **A/C 1F ↔ A/C 2F** *(cycling)* — skips loads below configurable watt threshold |

### Doorbell Overlay
Full-screen alert triggered by an HA `input_button` entity:
- **"DOOR" / "BELL"** in large white text
- Background flashes random colors every 250 ms
- Max brightness during alert; previous brightness restored on dismiss
- Screen rotation paused for the full alert duration
- First state delivery on boot or HA reconnect is silently ignored — only genuine new button presses trigger the alert

---

## Home Assistant Controls

All controls are exposed as HA entities under the device.

### Numbers
| Entity | Description | Default |
|--------|-------------|---------|
| `number.screen_transition_secs` | Seconds between screen rotations | 10 s |
| `number.screen_anim_ms` | Transition animation duration | 500 ms |
| `number.slot_cycle_secs` | Seconds between slot cycling (DC row 3 / AC row 3) | 2 s |
| `number.daily_restart_hour` | PHT hour for daily reboot | 3 (3 AM) |
| `number.display_brightness` | Backlight brightness | 50 % |
| `number.load2_threshold` | Min watts for A/C 1F slot to show | 3 W |
| `number.load3_threshold` | Min watts for A/C 2F slot to show | 10 W |
| `number.battery_capacity_ah` | Battery bank capacity | 300 Ah |
| `number.batt_time_threshold_hrs` | Max hours ahead to show time estimate | 18 h |
| `number.doorbell_duration_secs` | Doorbell alert display duration | 5 s |

### Switches
| Entity | Description | Default |
|--------|-------------|---------|
| `switch.batt_time_estimate` | Enable battery charge/discharge time estimate slot | ON |
| `switch.screen_clock_enable` | Include clock in rotation | ON |
| `switch.screen_dc_enable` | Include DC page in rotation | ON |
| `switch.screen_ac_enable` | Include AC page in rotation | ON |
| `switch.doorbell_enabled` | Enable doorbell overlay alert | ON |
| `switch.layout_preview` | Freeze display with max-width test values for alignment | OFF |
| `switch.batt_log_enabled` | Verbose battery logging to serial | OFF |
| `switch.solar_log_enabled` | Verbose solar logging to serial | OFF |

### Select
| Entity | Options |
|--------|---------|
| `select.transition_style` | Slide Left / Slide Right / Slide Up / Slide Down / Fade / **Random** |

### Text
| Entity | Description |
|--------|-------------|
| `text.load_1_label` | Display label for Ecoflow / Load 1 row |
| `text.load_2_label` | Display label for A/C slot 0 |
| `text.load_3_label` | Display label for A/C slot 1 |

---

## Sensor Entity IDs

Configured via `substitutions` at the top of the YAML:

```yaml
substitutions:
  solar_entity:                "sensor.srne_pv_power"
  load1_entity:                "sensor.ef_r30241_ac_input_power"
  load2_entity:                "sensor.a_c_1f_power_meter_power"
  load3_entity:                "sensor.a_c_3f_power_meter_power"
  battery_entity:              "sensor.battery_soc_mean"
  home_entity:                 "sensor.srne_load_l1_apparent_power"
  battery_power_entity:        "sensor.total_battery_power"
  batt_current_entity:         "sensor.total_battery_current"
  srne_charge_limit_entity:    "number.srne_batttery_charge_limit"
  srne_discharge_limit_entity: "number.srne_batttery_discharge_limit"
  doorbell_entity:             "input_button.doorbell"
```

---

## Layout Preview Mode

Toggle `switch.layout_preview` ON to freeze all slots with max-width placeholder values:

- All value labels show widest possible strings (`8.88kW`, `88.8%`, `8.88kVA`, `12:00AM`)
- Battery row shows slot 1 format: `"Charges to 88% at"` / `"12:00AM"`
- Icon animations run live: battery charges, solar cycles cloud → sun → moon/stars every 6 s

Toggle OFF to restore live sensor data.

---

## Hardware

| Component | Detail |
|-----------|--------|
| Board | ESP32 (esp32dev) |
| Display | ILI9341 240×320, SPI |
| Display rotation | 270° (landscape 320×240) |
| Backlight | LEDC PWM on GPIO21 |
| SPI pins | CLK=GPIO14, MOSI=GPIO13, MISO=GPIO12, CS=GPIO15, DC=GPIO2 |

---

## Flashing

```bash
# First flash (USB)
esphome run cyd-78d27c.yaml

# OTA update
esphome upload cyd-78d27c.yaml

# Validate config only
esphome config cyd-78d27c.yaml
```

---

## Notes

- All Philippine Time (PHT, UTC+8) calculations use `timestamp + (8×3600)` + `gmtime_r` — no TZ environment variable dependency.
- If all three screen-enable switches are turned OFF, the clock screen is always shown as a fallback.
- The doorbell alert ignores subsequent presses while already active; it dismisses automatically after the configured duration.
- On ESP32 boot or HA server restart, the `input_button` entity sends its last-pressed timestamp as the initial state sync. This is silently absorbed by tracking the last seen value in `g_doorbell_last_state` — the alert only fires when the value actually changes after the first delivery.

---

## Support

If this project has been useful to you, consider buying me a coffee ☕

### <img src="docs/logo-ln.png" width="20"> Lightning
<img src="docs/qr-lightning.png" width="200"><br>
`greatjogging67@walletofsatoshi.com`

### <img src="docs/logo-xrp.png" width="20"> XRP
<img src="docs/qr-xrp.png" width="200"><br>
`rpWJmMcPM4ynNfvhaZFYmPhBq5FYfDJBZu`<br>
Destination Tag: `2135058530`

### <img src="docs/logo-btc.png" width="20"> BTC
<img src="docs/qr-btc.png" width="200"><br>
`bc1q5tqqew0wlpkdz22crltreu5ngc9sdje9hzr4vv`
