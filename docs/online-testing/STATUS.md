# Online testing status

Build under test: `0.5.14-39-g0e4a9f3` (PR #78 head on byebyevlad/pr-78-rebased), pool The Hunter's
Dream. Last updated 2026-11-10.

| Area (maintainer's list) | Result | Where |
|---|---|---|
| Sign-in (Discord → The Hunter's Dream, keychain) | Works on Mac 1 and Mac 2 | sessions/2026-11-10-byebyevlad.md |
| Messages, bloodstains, ghosts | Work (receive). Writing a message: not tested | 2026-11-08 notes below |
| Mac ↔ Windows co-op | Works as guest (2 sessions, Windows hosts, 2026-11-08); a Windows helper joined Mac ↔ Mac sessions on 2026-11-10 | sessions/2026-11-10-byebyevlad.md |
| Mac ↔ Mac co-op, two networks (different cities) | Works, direct hole punch (2–4 probes), no relay | sessions/2026-11-10-byebyevlad.md |
| Passwords (co-op) | Work | sessions/2026-11-10-byebyevlad.md |
| Guest drops (Wi-Fi off 30 s) | Host fine; guest sent home, back online by itself, rejoins without restart | sessions/2026-11-10-byebyevlad.md |
| Host drops (Wi-Fi off 30 s) | Guest gets "Session lost" cleanly, **then cannot rejoin until it restarts** | findings/guest-stuck-after-host-timeout.md |
| Normal quit (guest) | Clean exit, no hang | sessions/2026-11-10-byebyevlad.md |
| Normal quit (host), force quit (both) | Not tested yet | |
| Invasions | **Never matched**, neither as invader nor as host, over two evenings | findings/invasions-no-match.md |
| Long session | 121 min online on Mac 1, no crash; FPS fell to 22–24 as memory grew | sessions/2026-11-10-byebyevlad.md |
| Hosting (Mac 1 as host) | Not tested yet (Mac 2 hosted) | |
| Packaged app | Not tested yet | |
| Mods/cheats online rule | Not tested yet | |

## 2026-11-08 notes (Mac 1 only)

M3 16 GB, macOS 27.2, build from byebyevlad/online-play-on-master (same online code). Messages,
bloodstains and ghosts appear. Two co-op sessions as a guest in Windows players' worlds (about 4 and
3 minutes), direct hole punch in 2 probes, both ended with a clean LeaveRoom.

## Open findings

1. **Guest stuck after the host times out:** reproduced twice. Client side (`gpu/bbnet/thd`).
   Lead: after the room closes the game re-activates signaling to the old host, and the client
   reports that connection established at once.
2. **Invasions never match:** cause unknown (game rules, The Hunter's Dream server, or client).
   Needs `BB_THD_TRACE=1` and a comparison with Windows players.
