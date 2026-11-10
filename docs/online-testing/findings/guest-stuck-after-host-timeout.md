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

## Lead

`gpu/bbnet/thd/thd_np.cpp`, `sig_activate`: when the game activates a connection to a peer that is
not a room member, the client resolves the peer's address on the server and, if it finds one,
reports ESTABLISHED then ACTIVE (`announce_locked`) with no traffic from the peer. After the room
is gone the game activates a connection to the old host (conn 5 / conn 2 above), the address
resolves because the host is online again, and the game is told it has a live connection to the
host outside any session. Hypothesis: the game then ignores further summons from that host while
it believes this connection is up. On a PS4 the activation would only succeed with the peer
answering.

Possible fixes, to choose after the trace below:
- do not announce a connection to a peer we share no room with unless the peer answers (or the
  peer is the guest a host is summoning, the case the code comment describes);
- fire DEAD for such a connection after a timeout without traffic;
- clear signaling state for the old host when `room_gone_locked` runs (it already marks the old
  connections dead, but the later activation creates a new one).

## Next steps

1. Reproduce with `BB_THD_TRACE=1` on the guest (Mac 1) to see the signaling callbacks, the
   game's status reads and the P2P datagrams from the host after each invite.
2. Mac 2's log for the same minutes (host side: summon datagrams sent after each invite?).
3. Check whether Windows players see the same (the backend comes unchanged from the Windows
   port).
