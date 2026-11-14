# Matchmaking password can't be set reliably: the text box loops and adds a space

Status: open, not yet investigated in code. Seen 2026-11-14 on a63a526 (fix/thd-bbhost-signaling),
but the text box (`src/runtime_services.c` ImeDialog, `gpu/shim/window.cpp` text input) is the
same on master.

## Observation (Mac 1, 16 GB, macOS 27.2)

Setting the game's matchmaking password (Network settings, up to 8 characters):

- The dialog is opened twice by the game (enter, confirm). Ours pre-fills each box with the
  game's current buffer (`initial` in `ime_init`).
- 17:31 and 17:49: password typed, confirm box closed with Esc -> password not set; the bell
  then found no partner.
- 18:05-18:09: the confirm box repeatedly came back as the 7-letter password **plus a trailing
  space** (8 characters = the field's maximum), within ~1 s of opening, while the player says
  they only pressed Enter. Once stored, the first box was pre-filled with the spaced version
  too. Entries never matched, and the box kept reopening; only Esc ended the loop.
- 18:14:52-56: player cleared both boxes with Backspace and retyped: four identical 7-letter
  entries (two matching pairs), yet the box still reopened after each pair; Esc ended it.
- The 14:47 session the same day: two entries, both typed, no extra box (worked).

## Suspects (unverified)

1. Trailing space: the pre-fill reads up to `max_length` UTF-16 units; if the game's buffer is
   space-padded or not terminated inside that limit, the pad comes along. Or a Space key event
   reaches the text box (Space bound to a game action).
2. Reopening: the Enter that confirms the box is consumed on KEY_DOWN only; the game may still
   see Enter (KEY_UP, or the bridged keyboard state polled in bb-probe) as "confirm" on the menu
   item and open the setting again.

## Next steps

Reproduce with `BB_DATA_DIR=$PWD/out/macdata` (never the real save): log the raw UTF-16 pre-fill
(length, last code unit) and every key event while the box is open; then trim/terminate the
pre-fill and swallow the confirming Enter until it is released.
