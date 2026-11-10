# Guest cannot rejoin after the host times out

Status: reproduced twice on Mac 1 (2026-11-10), cause not confirmed, no fix yet.

## Symptom

1. Mac 1 is a guest (Small Resonant Bell) in Mac 2's world.
2. Mac 2 loses its network for ~30 s. The server closes the room with `host_timeout`; Mac 1 shows
   "Session lost" and returns to its own world (correct so far).
3. Both ring again. The server sends Mac 1 `guest_invite` about once a minute, but Mac 1 never
   activates signaling or calls JoinRoom. Mac 2 sees "Could not discover the chime of a Small
   Resonant Bell".
4. Restarting **Mac 2** does not help (new room 14532, still ignored).
5. Restarting **Mac 1** fixes it: the next invite from the same room 14532 joins in 4 s.

When the **guest** loses its network instead (test 1), rejoining works without a restart.

## Evidence (Mac 1)

```
01:37:28.331 THD: np: event #... room_closed
01:37:28.345 THD: np: room 14487 gone (room closed: host_timeout): the game hears 0x1104
01:37:28.377 THD: sceNpMatching2LeaveRoom req 5
01:37:28.377 THD: np: session over (LeaveRoom): room 0
01:37:28.443 THD: np: ROOM_EVENT_CB event 0x1104 room 14487
01:37:37.469 THD: sceNpSignalingActivateConnection da_ya_pavl -> conn 5 (resolved <ip>)
01:37:42.050 THD: net: punch da_ya_pavl <ip>: 9 probes, peer heard
01:39:21.836 THD: np: guest_invite: room 14501 from da_ya_pavl (<ip>)
01:39:26.902 THD: net: punch da_ya_pavl <ip>: 20 probes, peer heard
01:40:16.082 THD: np: guest_invite: room 14501 from da_ya_pavl (<ip>)
   ... invites at 01:41:40, 01:42:35, 01:43:30, 01:44:25, 01:45:21, 01:46:21, 01:47:16,
       then room 14532 at 01:49:17, 01:50:11, 01:51:06: no ActivateConnection, no JoinRoom
```

Reproduction at 02:00:29: the same sequence (`host_timeout`, `LeaveRoom ... room 0`, and
`sceNpSignalingActivateConnection da_ya_pavl -> conn 2 (resolved <ip>)` 7 s later).

In test 1 the same post-session activation **failed** (`not known and not resolved`, Mac 1 was
offline), no connection was created, and the later rejoin worked.

## Evidence (Mac 2, Host side)

Between 01:37:00 and 01:52:

```text
01:37:11.329 ERROR GPU [Lib.Http] cpp-httplib failed for POST .../penalty/notify_multi_play_error
01:37:12.500 INFO  THD: sceNpSignalingActivateConnection byebyevlad -> conn 4
01:37:13.583 WARN  THD: np: POST /mp/matching2/heartbeat -> Could not establish connection
01:37:17.922 WARN  THD: stun: cannot resolve thehuntersdream.com
01:37:20.904 INFO  THD: sceNpMatching2LeaveRoom req 7
01:37:20.904 INFO  THD: np: session over (LeaveRoom): room 14487
01:37:20.906 WARN  THD: np: POST /mp/matching2/leave_room -> Could not establish connection
01:37:24.508 WARN  THD: np: POST /np/signaling/resolve -> Could not establish connection
01:37:32.929 WARN  THD: stun: cannot resolve thehuntersdream.com
01:37:41.777 WARN  THD: np: POST /np/events/poll -> Failed to read connection
01:37:43.323 INFO  THD: np: event #<id> room_closed
01:39:20.447 INFO  THD: sceNpMatching2CreateJoinRoom req 8 maxSlot 5
01:39:20.463 INFO  THD: sceNetSocket extnetwork2_udpp2p_ipv4_3658 domain=2 type=6 -> 7 (udp p2p)
01:39:20.463 INFO  THD: sceNetBind 7 vport 30 -> udp 3658 (p2p)
01:39:20.918 INFO  THD: np: room 14501 created, we are member 1
```

- **Did public address change after Wi-Fi returned?**
  No: `our address is now` was **never logged** after Wi-Fi restored. In `gpu/bbnet/thd/thd_session.cpp` (`MappedRefresh`), when `P2pStun` failed during the outage at 01:37:32, it recorded `g_mapped.at_ms = NowMs()` and retained the pre-outage address/port, arming a 120-second cooldown that suppressed any new STUN check when Wi-Fi returned. Consequently, Room 14501 was published to the server with Mac 2's pre-outage mapped port.
- **Did Mac 2 keep sending summons after each invite?**
  **No.** Mac 2 sent zero direct summons for room 14501. The host only listens on port 3658 after room creation and waits for incoming guest punch probes / server notifications (`room_member_joined`). Because the guest was punching toward Mac 2's stale pre-outage port, Mac 2 never received the guest's probes, never received `room_member_joined`, and never initiated direct summon datagrams.

After Mac 2 restarted (01:47:50):

```text
01:48:06.114 INFO  THD: stun: thehuntersdream.com:3478 sees our P2P port 3658 as <ip>:3658
01:48:21.106 INFO  THD: stun: our address is now <ip>:3658 (was :0); telling the server
01:49:15.844 INFO  THD: sceNpMatching2CreateJoinRoom req 2 maxSlot 5
01:49:16.250 INFO  THD: np: room 14532 created, we are member 1
01:49:17.825 INFO  THD: sceNpSignalingActivateConnection byebyevlad -> conn 1 (resolved <ip>:3658)
01:49:18.060 INFO  THD: net: send 3 -> <ip>:3658 vport 40->40: 15 bytes -> 19
01:49:37.994 INFO  THD: net: punch byebyevlad <ip>:3658: 40 probes, nothing heard
01:50:11.941 INFO  THD: sceNpSignalingActivateConnection byebyevlad -> conn 2
01:50:12.170 INFO  THD: net: send 3 -> <ip>:3658 vport 40->40: 15 bytes -> 19
```

Mac 2 refreshed STUN cleanly, opened Room 14532, and sent signaling and 40 UDP probes directly to Mac 1's IP, but Mac 1's socket did not answer (`40 probes, nothing heard`).

Reproduction at 02:00:11–02:05:

```text
02:00:11.668 ERROR GPU [Lib.Http] cpp-httplib failed for POST .../penalty/notify_multi_play_error
02:00:11.668 WARN  THD: stun: cannot resolve thehuntersdream.com
02:00:12.425 WARN  THD: np: POST /mp/matching2/heartbeat -> Could not establish connection
02:00:21.311 INFO  THD: sceNpMatching2LeaveRoom req 3
02:00:21.311 INFO  THD: np: session over (LeaveRoom): room 14532
02:00:42.674 INFO  THD: sceNpSignalingActivateConnection byebyevlad -> conn 5 (resolved <ip>:3658)
02:00:43.178 INFO  THD: net: punch byebyevlad <ip>:3658: 1 probes, peer heard
02:00:46.512 INFO  THD: np: event #<id> room_closed
```

Following this second drop, Mac 2's Beckoning Bell and Silencing Blank became grayed out/locked in inventory due to the unhandled `notify_multi_play_error` penalty latch. Returning to the Title Screen and choosing Continue cleared the lock.

### On the "stale mapped port" explanation

Mac 2's notes suggest that `MappedRefresh`'s 120 s cooldown kept a stale pre-outage port. Two things
argue against it as the cause: `KeepaliveMain` (`thd_session.cpp`) calls `MappedRefresh(true)` every
15 s, which bypasses the cooldown, and Mac 2's address and port were the same before and after
(port 3658 both times, no `our address is now` after the outage because nothing changed). The guest
also heard Mac 2's probes at 01:39:26. The two client bugs below fit both logs; the trace run should
settle it.

## Mechanism (two client bugs, `gpu/bbnet/thd`)

**A. Host: a stale, unresolved connection is reused (01:39–01:47).** During its outage the host's
game activated connections to the guest. The last one (conn 6) was created after the session
ended and could not be resolved, so it has no member and is never announced. `sig_activate`
(`thd_np.cpp`) is idempotent per peer: later activations for byebyevlad return that same live but
unannounced connection without logging. The host's game never gets ESTABLISHED for the guest, so
it never summons. Room 14501 waited eight minutes with no activation in the log.

**B. Guest: no new hole punch for a known address (01:49–01:52).** The guest's client punches only
when a peer's address changes (`PunchIfNew`, `thd_session.cpp`). After the session ended, the
guest kept the host's address in its peer table (the post-session activation at 01:37:37 resolved
it, and the first invite at 01:39 seeded it again). Every later invite carried the same address,
so the guest never punched again, and its router's mapping toward the host expired. The restarted
host then punched 40 times and heard nothing. Restarting the guest cleared its peer table, so the
next invite punched (4 probes) and the join went through.

Test 1 worked because the guest's post-session activation failed while it was offline: nothing
stale was left on either side.

## Fix candidates

- Guest: punch on every `guest_invite` (or when the last punch toward that peer is older than
  ~20 s), not only when the address changes.
- Both: when a room ends, drop the peer entries and connections that belonged to it, including
  ones activated after the room ended, so the next session starts clean.
- Host: in `sig_activate`, do not reuse a connection that never resolved; resolve again (or mark
  the old one dead and create a new one).

## Next steps

1. Reproduce with `BB_THD_TRACE=1` on both Macs to confirm (signaling events, datagrams).
2. Implement the fix on a branch from `pr-78-rebased`, repeat test 2 three times, then open a PR
   into `pr-78-rebased`.
3. Tell the Windows port: the backend comes from it unchanged.
