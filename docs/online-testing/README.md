# Online play testing (PR #78)

Shared notes from testing The Hunter's Dream online play on PR #78
(https://github.com/Wooloomooloo2/bloodborne_mac_port/pull/78), so the combined PR can carry a
test summary as the maintainer asked: Mac–Mac and Mac–Windows, sign-in, passwords, invasions,
disconnects/reconnects and long sessions.

Testers: byebyevlad (Mac 1), da-ya-pavl (Mac 2). Code under test: byebyevlad's fork, branch
`pr-78-rebased` (= #78 head). These notes live on branch `online-testing`.

## Layout

- `STATUS.md`: what has been tested, the results so far, and the open findings. Keep it current.
- `sessions/<date>-<tester>.md`: one report per tester per session (setup, timeline, results,
  log excerpts).
- `findings/<short-name>.md`: one file per bug or open question, with evidence and next steps.

## Rules

- **No raw logs in git.** The repository is public, and session logs contain IP addresses,
  online IDs of other players and server user IDs. Commit only short excerpts with IP addresses
  replaced by `<ip>`, user IDs by `<id>`, and other players' names by a role
  (`<windows-helper>`). Share full logs privately.
- No tokens or keychain contents anywhere.
- Times are UTC, as in the session logs (`~/Library/Logs/Bloodborne/game-<start>.log`).
- Every report names the build (About Bloodborne), Mac model, RAM and macOS version.

## Capturing more detail

- `BB_THD_TRACE=1` logs every datagram and signaling callback, not only the first ones. Needed for
  the guest rejoin bug and for invasions. From a checkout:
  `open -n --env BB_THD_TRACE=1 mac_port/Bloodborne.app`
- A freeze without a crash: before quitting, `sample bb-probe 10 -file ~/Desktop/bb-freeze.txt`.
