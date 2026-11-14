# Matchmaking password can't be set reliably: the text box loops and adds a space

Status: fixed on branch fix/ime-keyboard (ca08b43, from master); checked in the game 2026-11-14
18:41 (one box, 4-letter password confirmed, no reopen). Seen on a63a526
(fix/thd-bbhost-signaling); the text box (`src/runtime_services.c` ImeDialog) is the same on master.

## Cause

While the box was open the keyboard still drove the game's pad (`src/runtime_pad.c`), and Enter
and Space are both bound to Cross. The Enter that confirmed the box was still down when it
closed: the menu saw Cross on the password item and opened the field again. Bloodborne asks
for the password **once**; the "confirm box" below was this reopening. The fix keeps the
keyboard out of the pad while a box is open and until every key is released, and trims spaces
around a confirmed entry (`src/runtime_text_input.h`, `tests/test_text_input.c`).

## Observation (Mac 1, 16 GB, macOS 27.2)

Setting the game's matchmaking password (Network settings, up to 8 characters):

- The box seemed to open twice (enter, confirm); it was the reopening above. Ours pre-fills the
  box with the game's current buffer (`initial` in `ime_init`).
- 17:31 and 17:49: password typed, confirm box closed with Esc -> password not set; the bell
  then found no partner.
- 18:05-18:09: the confirm box repeatedly came back as the 7-letter password **plus a trailing
  space** (8 characters = the field's maximum), within ~1 s of opening, while the player says
  they only pressed Enter. Once stored, the first box was pre-filled with the spaced version
  too. Entries never matched, and the box kept reopening; only Esc ended the loop.
- 18:14:52-56: player cleared both boxes with Backspace and retyped: four identical 7-letter
  entries (two matching pairs), yet the box still reopened after each pair; Esc ended it.
- The 14:47 session the same day: two entries, both typed, no extra box (worked).
