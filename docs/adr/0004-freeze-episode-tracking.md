# ADR-0004: Track thermostat freeze episodes with timestamps

Status: accepted (2026-09-30)

## Context

A cloud thermostat feed can freeze while its entity stays available. A template binary sensor turns on when the temperature sensor has not reported for 3 hours. The obvious next step, alerting when that sensor has been on for 24 hours, does not work: reloading the integration rewrites entity state and refreshes `last_reported` even when the feed is still dead, so the sensor flaps off after every reload attempt. A `for: "24:00:00"` trigger or an age check on the sensor can never reach 24 hours across reload attempts. A `for:` clock also resets on every Home Assistant restart.

## Decision

Stamp when an episode begins in `input_datetime.thermostat_stale_first_seen`. The watchdog runs every 30 minutes, reloads at most once every 6 hours (throttled by `input_datetime.thermostat_last_reload_attempt`), and treats a stale detection as a new episode when the previous attempt was more than 12 hours ago (attempts recur about every 6.5 hours during a persistent outage, so a longer gap means the feed recovered in between). It alerts only when the feed is still stale two minutes after a reload and the episode is more than 24 hours old, which repeats about every 6 hours until fixed. A stamp older than 30 days is treated as ancient and restarted.

The reload targets the climate entity, so Home Assistant resolves the config entry itself and re-adding the integration cannot leave a stale entry id behind.

## Alternatives considered

- **`for: "24:00:00"` on the stale sensor.** Defeated by the reload behavior above.
- **Alert on every stale detection.** Most freezes clear by themselves after one reload; alerting would be noise.
- **Skip the reload and only alert.** The reload fixes most freezes without human help.

## Consequences

Two extra datetime helpers and a few templates in the watchdog. In exchange the alert means what it says: the feed has been dead for a day and reloads are not fixing it. When the feed recovers, the plan is re-applied, because `script.hvac_apply_plan` refuses to run while data is stale and would otherwise have dropped any change requested during the episode. This can also fire on a feed that is still dead, which is harmless because applying only sends commands when the thermostat differs from the plan.
