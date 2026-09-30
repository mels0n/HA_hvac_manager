# ADR-0002: One flag represents window mode

Status: accepted (2026-09-30)

## Context

An earlier design represented "the windows are open" in three places: a boolean, a "Windows" value in the daily-plan select, and a text helper that saved the plan to restore afterwards. A restart restored the boolean but reset the others to their defaults, so the trackers disagreed. The scheduled close reminder used a time that was now in the past and never fired, the plan guard deadlocked so the thermostat could not re-apply, and closing restored the wrong plan.

## Decision

`input_boolean.hvac_windows_open` is the only representation of window mode. `input_select.hvac_daily_plan` holds only the HVAC strategy (Cool, Heat, Mixed) and is never overwritten by window mode. Opening the windows turns the thermostat off; closing clears the flag and re-applies the intact plan. There is nothing to save or restore.

Two guarantees keep the close reminder from being stranded. Whenever the flag turns on by any route, `script.hvac_window_arm_close` re-arms the close time if the stored one is in the past. And after a restart, if the flag is on, the reconciler re-arms it too.

## Alternatives considered

- **Keep the plan value and fix the restore.** It removes the symptom but keeps three facts that can drift apart.
- **Derive window mode from a window sensor.** Most homes have no open/closed sensors on every window, and the manual "I opened them" tap is the input this design is built around.

## Consequences

Divergence is structurally impossible: there is one fact and one entity. The cost is that "windows open" is a manual signal, so if you forget to tap Closed, the thermostat stays off until the close reminder or a manual change.
