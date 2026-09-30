# ADR-0001: Ship helpers as a package, logic as scripts and one-file automations

Status: accepted (2026-09-30)

## Context

The manager needs about twenty helpers, two template sensors, a handful of automations, and logic that several automations share (apply the plan, scan the forecast, send a tracked message). It also has to be installable by someone who has never seen it.

## Decision

Declare the helpers and template sensors in one package, `packages/hvac_manager.yaml`. Put shared logic in scripts, one per file, and keep each automation in its own file in the automation editor's YAML format. Scripts that other files call (`hvac_apply_plan`, `hvac_window_scan`, `hvac_message_send`) are the single home of their logic.

The helpers have no `initial:` value. An `initial:` is re-applied on every restart, so a reader's tuned setpoints would be wiped. Without it a helper restores its last value, at the cost that numbers start at their minimum on first load, which the README covers as a first-run step.

## Alternatives considered

- **One large automation.** Simple to paste, but a single queued automation with a dozen branches is hard to read, and the window logic needs a different mode (`parallel`) from the schedule logic (`queued`).
- **UI-created helpers only.** More editable, but twenty helpers created by hand is a long, error-prone install, and the template sensor with an `attributes:` block or a trigger cannot be created there anyway.
- **A blueprint.** Blueprints cover one automation or script, not the helpers and sensors that make up most of this setup.
- **A shared Jinja macro.** Considered and left out. No non-trivial expression is repeated in more than one place, so there is nothing to share.

## Consequences

Installation is three kinds of paste (package, scripts, automations) instead of one. In return each file has one job, shared logic has one definition, and a reader can adopt any subset.
