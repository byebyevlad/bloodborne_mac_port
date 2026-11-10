# Online testing status

Build under test: `0.5.14-39-g0e4a9f3` (PR #78 head on byebyevlad/pr-78-rebased), pool The Hunter's
Dream. Last updated 2026-11-10.

| Area (maintainer's list) | Result | Where |
|---|---|---|
| Sign-in (Discord → The Hunter's Dream, keychain) | Works on Mac 1 and Mac 2 | sessions/2026-11-10-byebyevlad.md, sessions/2026-11-10-da-ya-pavl.md |
| Messages, bloodstains, ghosts | Work (receive). Writing a message: not tested | 2026-11-08 notes below |
| Mac ↔ Windows co-op | Works as guest (2 sessions, Windows hosts, 2026-11-08); a Windows helper joined Mac ↔ Mac sessions on 2026-11-10 | sessions/2026-11-10-byebyevlad.md, sessions/2026-11-10-da-ya-pavl.md |
| Mac ↔ Mac co-op, two networks (different cities) | Works, direct hole punch (1–4 probes), no relay | sessions/2026-11-10-byebyevlad.md, sessions/2026-11-10-da-ya-pavl.md |
| Passwords (co-op) | Work (IME dialog) | sessions/2026-11-10-byebyevlad.md, sessions/2026-11-10-da-ya-pavl.md |
| Guest drops (Wi-Fi off 30 s) | Host fine (zero hitch/crash); guest sent home, back online by itself, rejoins without restart | sessions/2026-11-10-byebyevlad.md, sessions/2026-11-10-da-ya-pavl.md |
| Host drops (Wi-Fi off 30 s) | Host survives, but retains stale port and locks bells (`notify_multi_play_error`); guest gets "Session lost", **then cannot rejoin until restarting** | findings/guest-stuck-after-host-timeout.md |
| Normal quit (guest) | Clean exit, no hang | sessions/2026-11-10-byebyevlad.md |
| Normal quit (host) | Clean exit, archive flushed, no hang (Mac 2) | sessions/2026-11-10-da-ya-pavl.md |
| Force quit (both) | Not tested yet | |
| Invasions | **Never matched**, neither as invader nor as host, over two evenings | findings/invasions-no-match.md |
| Long session | 121 min online on Mac 1 (FPS 22–24); 84 min and 48 min on Mac 2 (locked 60 FPS, 0 MB/hr leak) | sessions/2026-11-10-byebyevlad.md, sessions/2026-11-10-da-ya-pavl.md |
| Hosting (Mac 2 as host) | Verified working across 4 rooms | sessions/2026-11-10-da-ya-pavl.md |
| Hosting (Mac 1 as host) | Not tested yet | |
| Packaged app | Not tested yet | |
| Mods/cheats online rule | Not tested yet | |

## 2026-11-08 notes (Mac 1 only)

M3 16 GB, macOS 27.2, build from byebyevlad/online-play-on-master (same online code). Messages,
bloodstains and ghosts appear. Two co-op sessions as a guest in Windows players' worlds (about 4 and
3 minutes), direct hole punch in 2 probes, both ended with a clean LeaveRoom.

## Open findings

1. **Rejoin fails after the host drops:** reproduced twice, client side (`gpu/bbnet/thd`).
   Mac 1's reading: the host reuses a stale, unresolved signaling connection, and the guest never
   punches again toward a host address it already knows. Mac 2's reading: a stale mapped port on the
   host, and `/penalty/notify_multi_play_error` keeping the bells locked until the title screen.
   Details and the evidence for each in findings/guest-stuck-after-host-timeout.md; the traced repro
   decides.
2. **Invasions never match:** cause unknown. On Mac 2 (level 15, Central Yharnam) no Bell-Ringing
   Woman appeared for the solo host. The Hunter's Dream supports invasions but matches only within
   the same client. Needs `BB_THD_TRACE=1` and a comparison with Windows players.
