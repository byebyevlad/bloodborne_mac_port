# Current bbhost players can't connect to the Mac client: missing signaling handshake

Status: fixed on branch fix/thd-bbhost-signaling (34a1ef2 + 0a4b21c); co-op and an invasion with a
bbhost v0.2.3x player worked on 2026-11-14.

## Observation (Mac 1 host, 2026-11-14, test build 0fa03c8, `BB_THD_TRACE=1`)

Mac 1 (level 60) rang the Beckoning Bell, a Bell-Ringing Woman appeared; a Windows player on
bbhost (level 74) rang the Sinister bell. No invasion. The host's log showed, from two relay
ports (two peers), 46-byte datagrams arriving through The Hunter's Dream relay about every
500 ms, logged as `(no p2p header)`, passed to the game's first socket unparsed, and never
answered:

- one peer 14:47:18-14:48:28 and again from 14:49:50 (100+ datagrams)
- a second peer from 14:49:49 (the bbhost invader, by timing)

The game got no room or invite event; the client sent nothing back.

## Cause

bbhost [#217](https://github.com/droogie/bbhost/pull/217) (merged 2026-11-10, in v0.2.30+)
moved NpSignaling to a peer-to-peer handshake over the P2P port, as shadPS4's
`signaling_handler.cpp`: datagrams starting with `kSignalMagic` `{0xfe,'b','b','s'}`
(`src/hle/net_p2p.h`), handled by `hle_np_signaling_control` (`src/hle/np_signaling.cpp`).

Layout: magic (4), kind (1), reason (1), sender online id (16), receiver online id (16),
nonce (8) = **46 bytes**, matching the datagrams above. Kinds: ActivationRequest 0x01,
ActivationAck 0x02, Close 0x03, Established 0x04, Offer 0x11, Accept 0x12, Check 0x13,
CheckAck 0x14. The activating side resends ActivationRequest + Offer every 500 ms and gives up
(DEAD, timeout) after 30 s.

`gpu/bbnet/thd` was copied from Supermedo's backend at bbhost 566184a, before #217: it has
no handler for `0xfe 'b' 'b' 's'`, so it never answers, and every bbhost v0.2.30+ peer times
out. This likely breaks co-op with updated bbhost players too, not only invasions. Mac-to-Mac
and older bbhost peers are unaffected (both sides on the old scheme), which fits co-op with
Windows players working on 2026-11-08 and Mac-to-Mac working on 2026-11-10.

## Next steps

1. Port the handshake from bbhost's `np_signaling.cpp` (transport states, the 8 message kinds,
   500 ms resend, 30 s timeout, the events 0x01/0x0a/0x0b/0x0c/0x00) into `thd_np.cpp`, and
   route `kSignalMagic` datagrams in `thd_net.cpp` before the catch-all to the game socket.
   Decide how it coexists with the old server-side resolve (old Mac clients).
2. Unit test with recorded 46-byte packets, then a live test against a bbhost v0.2.34 player
   (co-op both ways and an invasion).
3. This also covers part of the maintainer's "drop unsolicited datagrams" request: these
   packets must be consumed, not handed to the game.

## Live test of the responder (2026-11-14 ~15:03, branch fix/thd-bbhost-signaling 34a1ef2)

Mac 1 host (Beckoning Bell, Bell-Ringing Woman present), bbhost v0.2.3x invader (Sinister bell):

- 15:03:49 room created for the invasion; 15:03:52 the invader's ActivationRequest, Offer and
  Check were answered; 15:03:53 "handshake done", the invader's Established arrived. Game
  datagrams flowed both ways (a few of 15-62 bytes, then 9-byte keep-alives every 6 s).
- The invader never joined the room on the server (no `room_member_joined`); after 60 s the host
  game left the room (15:04:52). The connection stayed open (keep-alives until 15:06).
- 15:06:29 a co-op attempt with the same player: new room, the host game activated a second
  connection to him while the first was still open, sent its 15-byte summon datagrams; his last
  datagram arrived 15:06:30.9 and his game crashed (no crash details available).

Open: why the invader didn't join (likely another part of bbhost #217's matching2 changes on his
side), and whether the crash comes from the overlapping second session. Next: a clean co-op test
with both games restarted.

## Second fix and the working test (2026-11-14 ~15:20, 0a4b21c)

Cause of the stall: bbhost's game waits for MUTUAL_ACTIVATED (0x0c) before it goes on with a
summon, and bbhost sets the connection's "peer activated" mark only from the other side's
ActivationRequest (`hle_np_signaling_control`, kActivationRequest). We answered the handshake
but never sent one. 0a4b21c sends it once, when our game has its own connection to a peer that
spoke the handshake (never to Mac peers).

Live, same players, both games restarted:
- Co-op, Mac host: 15:20:07 handshake with our ActivationRequest answered (ActivationAck),
  15:20:09 the bbhost player joined the room as member 2; left normally at 15:21:34.
- Invasion, Mac invading the bbhost host: 15:22:04 handshake, 15:22:06 joined his room as
  member 2; left at 15:23:05. The player confirmed the invasion worked.

Not yet tested: a bbhost player invading the Mac host, rejoin after drops with bbhost peers,
long sessions.

## Host Wi-Fi drop with a bbhost helper (2026-11-14)

- 15:25:37 Mac host's Wi-Fi off (~30 s); 15:25:45 session over, 15:26:57 the server closed the room.
- 15:33:15 Mac host rang again: room created, nothing from the bbhost helper for 5+ minutes (no
  handshake, no join): the stuck-guest bug of guest-stuck-after-host-timeout.md, on bbhost's side
  (same `punch_if_new` logic upstream; our guest half runs only on Macs).
- The helper restarted his game: 15:40:17 room, 15:40:19 activation (resolved), 15:40:20
  handshake, 15:40:23 joined. The Mac host recovered without a restart.
