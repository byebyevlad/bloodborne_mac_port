# Invasions never match

Status: observed, cause unknown.

## Observations (Mac 1, byebyevlad, 2026-11-08 to 2026-11-10)

- Ringing the Sinister Resonant Bell in several areas for minutes at a time never invaded anyone,
  while the Small Resonant Bell in the same spots found a co-op session within a minute every time.
- Hours of play with Beckoning Bells rung: never invaded.
- 2026-11-10, ~00:5x–01:20: Mac 1 (level 20) on the Sinister bell, Mac 2 ringing the Beckoning Bell
  in an area with a living boss: no match.
  - **Mac 2 details:** Level 15 (character `suchka`), Central Yharnam lantern, with both Cleric Beast
    and Father Gascoigne alive.
  - **Bell-Ringing Woman:** Did **not** appear in Mac 2's world. In Bloodborne, a solo host ringing
    the Beckoning Bell in early areas like Central Yharnam does not spawn a chime maiden until a
    cooperator arrives (unlike Nightmare areas where she spawns naturally, or when the host rings the
    Sinister bell themselves).
  - **Mac 2 THD log lines:** The log showed only regular background polling and STUN keepalives
    (`THD: np: the server holds polls (WaitMaxMs 25000): long polling with WaitMs 25000`). No room
    creation or matchmaking requests occurred, and no network errors were recorded.
- While the Sinister bell rings, the log shows nothing at all: no matchmaking calls and no failed
  requests. The invader search goes through the game's own server requests, which the client logs
  only when they fail.

## What the client does

- Co-op matching works through the game's sign server and `guest_invite` events
  (`gpu/bbnet/thd/thd_matching2.cpp`). Rooms are created with 5 slots ("the host, two
  cooperators, two invaders"), so invaders are anticipated.
- `sceNpMatching2SearchRoom` is answered "server not available" ("No room list: summons go through
  the game's own sign server"). If the game used room search for invasions, that would block them;
  not verified.

## Possible causes

1. Game rules: no Bell-Ringing Woman in the host's world, boss defeated, levels out of range.
2. The Hunter's Dream server: invasion matching off, or not offered to the `bbport-mac` client.
3. The client: a request the invader or the host needs is missing or answered wrongly.

## Next steps

1. Ask on The Hunter's Dream / the Windows port whether invasions work for Windows players now.
2. A controlled test with `BB_THD_TRACE=1` on both Macs: host at level 10+ in an area with a living
   boss, Beckoning Bell rung, waiting for the Bell-Ringing Woman's sound; invader on the Sinister
   bell. Note levels, area and times.
3. Log the game server requests during the Sinister bell (successful ones too) to see what the
   invader asks and what the server answers.
