# Home Assistant HVAC Manager

![Today's Plan](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fcr_lights.melson.us%2Fcr_light_stats.php&query=%24%5B%27sensor.public_hvac_plan%27%5D&label=Today%27s%20Plan&color=blue)
![Humidity Trim](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fcr_lights.melson.us%2Fcr_light_stats.php&query=%24%5B%27sensor.public_hvac_humidity_trim%27%5D&label=Humidity%20Trim&color=teal)
![License](https://img.shields.io/badge/license-MIT-blue)

*The badges show what the system is doing in my house right now: today's plan, and how far the cooling setpoint is trimmed for indoor humidity.*

## What this is

A how-to for a thermostat manager in Home Assistant. Every morning it decides whether today is a Cool, Heat or Mixed day from the forecast, sets day and night setpoints, lowers the cooling setpoint when the house is humid, suggests opening the windows when the weather is genuinely comfortable, and watches for a frozen thermostat feed. It talks to you over Telegram, with inline buttons and typed commands.

This is a cleaned-up, generic version of the setup running in my home, not a copy of my live configuration. Entity names are generic and anything specific to my house has been removed. Temperatures are in Fahrenheit and the thresholds are starting values I picked, not tuned ones, so expect to adjust them.

**How it works**

- **Daily plan.** At 05:00 `script.hvac_compute_plan` reads the daily forecast. The season (Summer, Winter, Shoulder) comes from the next three days. Today's plan comes from today alone: a big swing (afternoon 76 or above, morning 52 or below) is Mixed, 78 or above is Cool, 60 or below is Heat, anything else is Mixed. `script.hvac_apply_plan` then puts the thermostat in the matching mode. It only sends a command when the thermostat differs from the plan, because cloud thermostat APIs are rate limited.
- **Setpoints.** Separate day and night values for cooling and heating, switched at 07:30 and 21:30. A change made at the wall or in the vendor's app turns on a manual hold, so the manager stops overriding it until the next schedule boundary. The 07:30 and 21:30 triggers tell the apply script which setpoints to use (`force_day`) rather than waiting on the awake-hours sensor, which flips at the same instant.
- **Humidity-compensated cooling.** 74F at 61% humidity feels warmer than 74F at 45%. The cooling setpoint is lowered by 1F for every 8% of indoor humidity above 45%, capped at 5F, with hysteresis so the trim does not flap around a band edge.
- **Daily digest.** At 07:35 a message with the plan, outdoor and indoor conditions, the setpoint, and the open-window outlook.
- **Window advisor.** It scans the next 20 hours of forecast for a run of at least N comfortable hours (default 4). If one has started it prompts you with an "Opened them" button. If one starts later it reminds you 15 minutes before. Once the windows are marked open the thermostat is turned off, and you get a close reminder 15 minutes before the stretch ends, plus alerts for rain, heat or cold. Tap "Closed them" or type `/closed` and the plan is restored. Typed `/opened` and `/closed` work like the buttons.
- **Thermostat freeze watchdog.** Cloud thermostat feeds can freeze while the entity still looks available. A "stale" sensor notices, the watchdog reloads the integration, and it only alerts you once the freeze has lasted more than 24 hours. When the feed recovers, the plan is re-applied.
- **Restart reconciler.** A restart can leave the season and plan stale, so after startup they are recomputed and re-applied.
- **Filter reminder.** Heating and cooling runtime is counted in 15-minute units, with a reminder at 350 hours that repeats every two days until you tap "Replaced" or type `/filterdone`.

| File | Purpose |
|---|---|
| `packages/hvac_manager.yaml` | Helpers, the awake-hours and stale-data sensors, and the two public display sensors |
| `scripts/hvac_compute_plan.yaml` | Season and daily plan from the forecast |
| `scripts/hvac_apply_plan.yaml` | Puts the thermostat in line with the plan |
| `scripts/hvac_restart_recalc.yaml` | Recompute and re-apply after a restart |
| `scripts/hvac_window_scan.yaml` | Finds the next comfortable stretch of weather |
| `scripts/hvac_window_arm_close.yaml` | Arms the close reminder |
| `scripts/hvac_window_close_restore.yaml` | Marks windows closed and restores the plan |
| `scripts/hvac_message_send.yaml` | Sends a Telegram message and records its ids |
| `scripts/hvac_message_update.yaml` | Edits the recorded messages |
| `scripts/hvac_filter_remind.yaml` | Sends the filter reminder |
| `scripts/hvac_filter_done.yaml` | Resets the counter and closes out reminders |
| `automations/hvac_daily_plan.yaml` | 05:00 plan |
| `automations/hvac_setpoints.yaml` | 07:30 and 21:30 setpoints |
| `automations/hvac_manual_hold.yaml` | Honors changes made at the thermostat |
| `automations/hvac_humidity_compensation.yaml` | Humidity trim with hysteresis |
| `automations/hvac_daily_digest.yaml` | 07:35 digest |
| `automations/hvac_window_suggest.yaml` | Window prompts and pre-window reminder |
| `automations/hvac_window_respond.yaml` | Opened and Closed buttons and commands |
| `automations/hvac_window_close_alerts.yaml` | Close, rain, heat and cold alerts |
| `automations/hvac_thermostat_watchdog.yaml` | Freeze watchdog and re-apply on recovery |
| `automations/hvac_restart_reconciler.yaml` | Startup repair |
| `automations/hvac_filter_runtime.yaml` | Runtime counter and reminders |
| `automations/hvac_filter_done.yaml` | Replaced button and `/filterdone` |

## Map your entities

The files use these generic ids. Find and replace each one with yours across `packages/`, `scripts/` and `automations/` before installing.

| Generic id | What it is | Notes |
|---|---|---|
| `climate.thermostat` | Your thermostat | Must support `heat`, `cool`, `heat_cool` and `off`, with `temperature`, `target_temp_high` and `target_temp_low`, and report `hvac_action` |
| `weather.home` | A weather entity | Needs daily forecasts with `temperature` (`templow` is used when present; days without it are skipped), and hourly forecasts with `temperature`, and ideally `humidity`, `wind_speed` and `precipitation` (`precipitation_probability` is used only if the feed provides it) |
| `sensor.thermostat_temperature` | Indoor temperature in F | Whole-house average or the thermostat's own reading |
| `sensor.thermostat_humidity` | Indoor relative humidity in % | The humidity bands assume whole-percent readings |
| Telegram | The built-in `telegram_bot` integration | Not `notify.*`. Buttons, edits and typed commands need the bot integration itself |

Everything else is created by the package. Its ids are `input_boolean.hvac_windows_open`, `input_boolean.hvac_manual_hold`, `input_select.hvac_season`, `input_select.hvac_daily_plan`, `input_number.hvac_cool_day`, `hvac_cool_night`, `hvac_heat_day`, `hvac_heat_night`, `hvac_humidity_offset` and `hvac_window_min_hours`, `input_datetime.hvac_window_close_by`, `hvac_window_open_by`, `thermostat_last_reload_attempt`, `thermostat_stale_first_seen` and `hvac_last_filter_reminder`, `input_text.hvac_chat_ids`, `hvac_window_messages` and `hvac_filter_reminder_messages`, `counter.hvac_filter_runtime_quarters`, `binary_sensor.hvac_awake_hours`, `binary_sensor.thermostat_data_stale`, `sensor.public_hvac_plan` and `sensor.public_hvac_humidity_trim`.

## Install

**Requirements:** Home Assistant 2026.6 or later (the automations use `note:` fields to explain non-obvious steps; on an older release, delete those lines), a thermostat integration, a weather integration, and the `telegram_bot` integration configured with your bot and allowed chats. No custom components.

1. **Telegram.** Set up the Telegram bot integration and add your chat ids to its allowed list. Buttons and typed commands only work for allowed chats.
2. **Package.** Enable packages if you have not already, by adding this to `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
   Copy `packages/hvac_manager.yaml` to `/config/packages/` and restart Home Assistant.
3. **First-run values.** Helpers deliberately have no `initial:` value, so numbers start at their minimum. In Settings > Devices & services > Helpers, set the four setpoints, `HVAC Window Min Hours` (4 is a reasonable start), and `HVAC Chat Ids` (comma-separated, for example `111111111,222222222`; a group id is negative). Confirm `binary_sensor.thermostat_data_stale` is off and `binary_sensor.hvac_awake_hours` matches the time of day.
4. **Scripts.** For each file in `scripts/`: Settings > Automations & scenes > Scripts > Add script > three-dot menu > Edit in YAML, paste the file, save. The script id (the entity id after `script.`) must match the file name, so use the file name, minus `.yaml`, when Home Assistant asks for or generates an id. Install `hvac_message_send`, `hvac_message_update`, `hvac_window_scan` and `hvac_compute_plan` first; the others call them.
5. **Automations.** For each file in `automations/`: Settings > Automations & scenes > Create automation > three-dot menu > Edit in YAML, paste the file, save.
6. **Try it.** Run `script.hvac_compute_plan` from Developer tools > Actions and check that `input_select.hvac_daily_plan` changes. Run `script.hvac_message_send` with a test message to confirm Telegram works.

**Tuning.** These are starting values you can change without touching the design.

| What | Where |
|---|---|
| Season and plan thresholds (78/62, 76/52, 78, 60) | `scripts/hvac_compute_plan.yaml` |
| Comfort band for windows (62F floor, 76F ceiling, dew point under 67F, wind under 20 mph) | `scripts/hvac_window_scan.yaml` |
| Minimum comfortable hours for a window | `input_number.hvac_window_min_hours` |
| Humidity trim (45%, 8% per degree, 5F cap) | `automations/hvac_humidity_compensation.yaml` |
| Too hot or too cold with windows open (77F and 66F) | `automations/hvac_window_close_alerts.yaml` |
| Filter reminder at 1400 quarter-hours (350 h) | `automations/hvac_filter_runtime.yaml` |

**Continuous fan.** Nothing here touches the fan mode. Some thermostat integrations refuse to turn the fan on while the system is off, which is exactly the state windows-open mode uses. If you want the fan always on, use the thermostat vendor's own schedule.

**Dashboard.** A quick view of what the manager is doing:

```yaml
type: entities
title: HVAC Manager
entities:
  - entity: input_select.hvac_season
  - entity: input_select.hvac_daily_plan
  - entity: input_number.hvac_humidity_offset
  - entity: input_boolean.hvac_windows_open
  - entity: input_boolean.hvac_manual_hold
  - entity: binary_sensor.thermostat_data_stale
  - entity: counter.hvac_filter_runtime_quarters
```

## Publish live values (optional)

The package defines two display sensors meant for public badges:

- `sensor.public_hvac_plan` is `Cool`, `Heat` or `Mixed`, straight from the daily plan.
- `sensor.public_hvac_humidity_trim` is `None` when no trim is active, or a value like `-2°F` when cooling is being trimmed for humidity.

Neither says anything about whether windows are open or anyone is home. To publish them, use the read-only proxy and badge setup described in the "Publish live values (optional)" section of [HA_circadian_lights](https://github.com/mels0n/HA_circadian_lights), and add these two ids to the proxy's allow-list of entities:

```
sensor.public_hvac_plan
sensor.public_hvac_humidity_trim
```

Then point a shields.io [dynamic JSON badge](https://shields.io/badges/dynamic-json-badge) at the proxy with a query like `$['sensor.public_hvac_plan']`.

## Secrets

The Telegram bot token is entered in the integration's setup screen and lives in Home Assistant's own storage. Chat ids go in `input_text.hvac_chat_ids`. Do not paste either into the YAML in this repository or commit them. Nothing else here needs credentials, and there is no `secrets.yaml`.

## License

MIT. See [LICENSE](LICENSE).
