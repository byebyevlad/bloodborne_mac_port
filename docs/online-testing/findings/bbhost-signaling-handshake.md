# Current bbhost players can't connect to the Mac client: missing signaling handshake

Status: cause identified from the host's log and bbhost's source; fix not written.

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
