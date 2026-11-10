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
