# ADR-0003: Track Telegram messages by recorded id

Status: accepted (2026-09-30)

## Context

The window advisor and the filter reminder put an inline button on a message and later need to update that message in every chat: remove the button, or replace the text with a confirmation. Home Assistant's Telegram integration can address an edit to "the last message" in a chat. That is the newest message, including one a person typed, and a bot cannot edit messages it did not send. The symptom was a tap in one chat updating that chat but not the other, because the other person had typed a command in between.

## Decision

`telegram_bot.send_message` returns the sent message ids. `script.hvac_message_send` records them as `chat:message,chat:message` in an `input_text` helper (the "store") and, before sending a new message, removes the buttons from the previously recorded ones so only the newest message carries a live button. `script.hvac_message_update` edits the recorded messages by id. Button handlers also edit the tapped message directly when its id is not in the store, which covers a message sent before tracking existed. A message sent without a store (a one-off alert or a digest without a button) is not recorded, so it cannot strip a live button from an earlier prompt.

## Alternatives considered

- **`message_id: last`.** Rejected for the reason above.
- **Delete and resend.** Loses the chat history and notifies again.
- **One helper for all messages.** Window and filter messages have different lifetimes, so each gets its own store.

## Consequences

Every interactive flow needs a store helper, and the 255-character limit on `input_text` bounds how many chats can be tracked (about a dozen). Typed commands such as `/opened` and `/closed` fire `telegram_command`, not `telegram_callback`, so each flow handles both.
