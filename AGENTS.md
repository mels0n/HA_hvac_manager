# AGENTS.md

Contributor guide for this repository.

## What it is

A how-to for a thermostat manager in Home Assistant: daily plan, setpoints, humidity-compensated cooling, an open-window advisor, a thermostat freeze watchdog, and a filter reminder. The deliverables are configuration files a reader copies into their own instance. There is no build step and no application code.

## Layout

- `packages/`: Home Assistant package (helpers and template sensors)
- `scripts/`: one script per file, in the format the script editor's YAML mode accepts
- `automations/`: one automation per file, in the format the automation editor's YAML mode accepts
- `docs/adr/`: architecture decision records

## Checks

- YAML must parse: `python -c "import yaml,sys; [yaml.safe_load(open(f,encoding='utf-8')) for f in sys.argv[1:]]" packages/*.yaml scripts/*.yaml automations/*.yaml`
- Test every template change in Developer tools > Template against a real instance before committing. Templates that read a forecast can be tested by substituting a hand-built list for the `weather.get_forecasts` response.
- No em dashes anywhere in the README, this file, the ADRs or comments.

## Conventions

- Keep examples generic: no real entity names from a specific house, no IP addresses, no chat ids, no tokens. The mapped entities are `climate.thermostat`, `weather.home`, `sensor.thermostat_temperature` and `sensor.thermostat_humidity`.
- Prefer native triggers and conditions over templates where one exists. A template is fine for comparing an attribute to a computed value, or for building message text.
- Choose the automation mode deliberately: `queued` for sequential work, `parallel` where a long wait must not block other triggers, `single` for idempotent resyncs.
- Explain non-obvious steps with a `note:` or a comment next to the step.
- Address Telegram edits by recorded message id, never by "the last message".
- Leave presence, away and vacation logic out. This repository is the generic core; add those conditions in your own copy.
