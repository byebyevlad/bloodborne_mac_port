# Session 2026-11-09/10, byebyevlad (Mac 1)

- Mac: Mac15,12 (Apple M3), 16 GB, macOS 27.2 (26B123)
- Build: `0.5.14-39-g0e4a9f3` (checkout app, Command Line Tools 27.2)
- Settings: 1920x1080 windowed, upscaler off, 60 FPS cap, online on, pool The Hunter's Dream
- Partner: da-ya-pavl (Mac 2, another city, same build). A Windows player (`<windows-helper>`)
  joined as a second helper.
- Sessions: `game-2026-11-09_23-50-42` (23:50–01:52, 121 min) and `game-2026-11-10_01-52-43`
  (from 01:52). Times are UTC.

## Timeline

| Time | What | Result |
|---|---|---|
| 23:53 | Online start | Connected, STUN address registered, long polling |
| 00:5x–01:20 | Mac 1 rings the Sinister Resonant Bell, Mac 2 the Beckoning Bell | No match (see findings/invasions-no-match.md) |
| 01:24:58 | Password co-op, Mac 1 joins Mac 2 | Invite, punch in 4 probes, joined at 01:25:02. Windows helper connected at 01:25:11 |
| 01:31:35 | **Test 1:** Mac 1 Wi-Fi off ~40 s | Mac 1 leaves the room at once, requests fail while offline, back online by 01:32:17 without restart. Mac 2 unaffected |
| 01:36:25 | Rejoin after test 1 | Joined at 01:36:29, no restart |
| ~01:37:00 | **Test 2:** Mac 2 Wi-Fi off ~30 s | 01:37:28 server closes the room (`host_timeout`); Mac 1 shows "Session lost", back in own world, no freeze |
| 01:39–01:51 | Rejoin attempts | 11 invites from Mac 2's rooms (14501, then 14532 after Mac 2 restarted): **Mac 1 never joins** |
| 01:52:22 | **Test 3:** Mac 1 quits normally | Clean exit ("ended normally after 121 min 41 s") |
| 01:57:29 | Rejoin after restarting Mac 1 | Joined room 14532 at 01:57:33 on the first invite |
| 02:00:29 | **Repro of test 2:** Mac 2 drops | Same as test 2: `host_timeout`, "Session lost", new signaling activation to Mac 2 7 s later |
| 02:00–02:10 | Rejoin attempt | Mac 2 out of Insight: no Beckoning Bell, not verified |
| ~02:10 | Mac 1 rings the Sinister Resonant Bell alone | Nothing at all in the log |

## Memory (Mac 1, phys_footprint every 30 s)

| Time | bb-probe | bbgpu-host-macos27 | Swap used |
|---|---|---|---|
| 00:13 | 3.6 GB | 8.9 GB | 7.8 GB |
| 00:55 | 3.6 GB | 9.0 GB | 8.4 GB |
| 01:36 | 3.8 GB | 9.1 GB | 8.5 GB |
| 01:59 (after restart) | 3.3 GB | 6.9 GB | 5.6 GB |

FPS dropped to 22–24 at 1080p late in the first session (60 cap); after the restart it recovered.
Not network-related, but relevant to long sessions and to #144.

## Log excerpts

Join (01:24:58):

```
THD: np: guest_invite: room 14460 from da_ya_pavl (<ip>)
THD: net: punch da_ya_pavl <ip>: 4 probes, peer heard
THD: sceNpSignalingActivateConnection da_ya_pavl -> conn 1
THD: sceNpMatching2JoinRoom req 2 room 14460
THD: np: joined room 14460 as member 2 with 1 peers
```

Test 1, guest offline (01:31:35):

```
THD: sceNpMatching2LeaveRoom req 3
THD: np: session over (LeaveRoom): room 14460
THD: np: POST /mp/matching2/leave_room -> Could not establish connection
THD: sceNpSignalingActivateConnection da_ya_pavl -> conn 3 (not known and not resolved: Could not establish connection)
THD: np: POST /np/events/poll -> Failed to read connection        (01:32:17, last error)
```

Test 2 and the stuck guest: see findings/guest-stuck-after-host-timeout.md.
