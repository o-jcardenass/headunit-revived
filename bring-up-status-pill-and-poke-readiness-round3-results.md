# bring-up-status-pill-and-poke-readiness — round 3 results

**Candidate:** `fork/feat/bring-up-status-pill-and-poke-readiness` @ `fe8b91e2` (two commits past round
1/2's `ef66abbf`: `0f2a65ce5` "the WiFi Direct channel toasts move into the status pill", `fe8b91e2`
"a wake round with no answer drops the pill back to waiting" — the latter is a direct fix attempt for
round 1/2's P3 FAIL)      **Baseline:** none (unchanged from brief §1; no `main` behaviour to compare)
**APK md5:** `f35b083ca006ab55e4eba1b9ca77631b`
(`com.andrerinas.headunitrevived_3.4.0-beta1_debug.apk`, versionCode 106)
**Unit:** D-HU = UNISOC MT50 (`MT50_YT610E4GFPSL_U`, Android 14, single BT radio, adbd root), bonded
name "Navegadortz2" `11:46:03:10:33:59`. D-POCO = POCO X3 NFC (`M2007J20CG`, Android 11, not rooted)
`DC:B7:2E:5E:4E:59`, phone role for Part A. D-MOTO not used this round.
**Date:** 2026-09-10

## Setup notes

### Why this round exists

This is not a re-run of the full round 1/2 brief. It targets the two commits landed on the branch
since round 2 graded `ef66abbf`: `0f2a65ce5` (pill grows a third line, absorbing the 5GHz-band toast
that round 1/2 found covering it) and `fe8b91e2` (a `ConnectionStageTracker.retreat()` call meant to
fix P3's FAIL — the pill latching on `WAKING_PHONE` forever when the phone never answers). Both
commits only touch pill/UI display and the poke retry loop's stage reporting, not poke targeting,
credential handling, or auto-disconnect. **Scope decision:** re-ran R0, P1, P2, P3, P4, and P5
(everything Part A, the only section either commit can affect); did not re-run Part B (W1/W1b/W2/
W2b/W3) or Part C (D1/D2/D3) or P6. Part B/C exercise the exact same poke-loop and auto-disconnect
code paths already PASSed twice on `ef66abbf` and are graded on poke/session lines, never the pill,
so neither commit changes what they measure; D2/D3 were blocked last round by an unrelated D-HU
network condition this branch does not touch; P6 has no USB host on this rig regardless.

### Scripts used (`hur-wifi-test-scripts/`, a sibling dir, not this repo)
`build_hur.sh`, `run_unit_tests.sh` (R0), `set_hu_settings_host.py` (D-HU, rooted). No new script
needed.

### Deviations from protocol

1. **Capture hygiene: the previous run's `adb logcat` was left running twice this round** —
   `r2-p1-dhu.txt`'s capture was still attached when P3's launch fired, and `r1-p2-dhu.txt`'s capture
   was still attached when P4's launch fired. Each later run had its own dedicated, freshly-cleared
   (`logcat -c`) capture file that is unaffected (a second independent `adb logcat` client reading the
   same ring buffer does not lose or corrupt lines for either reader). The stale files' *tails* carry
   a second app process's lines after the graded run ended; where that changed a raw discard-rule
   count (P1 run 2's `createGroup SUCCESS` counted 2 in the raw file), the run was re-scoped to its
   own pid before the next process's first line and re-checked (see P1 run 2 below) — the actual run
   itself was never contaminated, only the bookkeeping. Both stray captures were killed once noticed;
   confirmed `ps aux | grep logcat` clean at the end of the round.
2. **`native-preferred-device-mac` came into the round as an empty string, not an absent key** (a
   leftover from round 2, which the template says can read differently from a deleted key). Deleted
   it with `del:` before R0 per brief §2, confirmed absent in the read-back.
3. D-POCO's airplane mode was driven with `settings put global airplane_mode_on` plus explicit
   `svc bluetooth disable`/`svc wifi disable` (and the enable pair to restore), verified with
   `dumpsys` before treating either radio as down — consistent with this rig's known
   airplane-mode-via-`am broadcast` limitation, applied here to the phone.
4. Both D-HU's `settings.xml` and D-POCO's radios were restored (D-HU: pre-round backup pushed back;
   D-POCO: both radios re-enabled and verified `enabled: true` / `Wi-Fi is enabled`) before ending the
   round.

## R0 — build and unit-test gate

**PASS**

- Build: `build_hur.sh` → `BUILD SUCCESSFUL`, apk md5 `f35b083ca006ab55e4eba1b9ca77631b` (different
  from round 1/2's `222016d5…`, as expected for a different SHA).
- Unit tests: `run_unit_tests.sh` → `BUILD SUCCESSFUL`; test-results XMLs sum to **1626 tests, 0
  failures, 0 errors, 0 skipped** (up from round 1/2's 1616 — `0f2a65ce5` and `fe8b91e2` together add
  10 new JVM tests, all passing).
- Install: `adb install -r` on D-HU; live `md5sum` of `pm path` base.apk matches the built apk.
- Identity: `ACTION_QUERY_STATE` → `"commit":"fe8b91e2ae8a"`.

---

## Part A — status pill, D-HU head unit, D-POCO phone

### P1 — Native AA cold bring-up to projection

**FAIL** (run 1 of 2 — the rank-monotonicity condition, "the most valuable thing Part A can find")

This is the round's central finding: **`fe8b91e2`'s fix for P3 introduces the exact P1 regression the
brief warns about, and it is intermittent, not deterministic.**

**Run 1 — FAIL.** Settings: `wifi-connection-mode=3`, `log-level=2`, `auto-start-bt-macs` empty,
`native-preferred-device-mac` deleted. Radio: D-POCO airplane-on + `svc bluetooth/wifi disable` at
launch (22:41:xx), confirmed off; D-POCO radios restored 22:42:24.862, confirmed on. Discard-rule
check: clean (`MATCH!`=0, `createGroup SUCCESS`=1, `Magic Garbage`=0; the `p2p-wlan0-7`→`8` bump
happened before the first `createGroup SUCCESS`, the documented stale-group-teardown pattern, not
contamination).

| # | time | value | rank | note |
|---|---|---|---|---|
| 1 | 22:42:01.968 | ARMED | 10 | first value |
| 2 | 22:42:02.030 | PREPARING_NETWORK | 20 | |
| 3 | 22:42:02.340 | WAITING_FOR_PHONE | 40 | |
| — | 22:42:02.354 | — | — | `startNativeAaQuietHost() requested` |
| 4 | 22:42:02.356 | PREPARING_NETWORK | 20 | rank decrease, 2ms after the requested line — the allowed exception |
| 5 | 22:42:02.643 | WAKING_PHONE | 45 | |
| 6 | **22:42:29.748** | **WAITING_FOR_PHONE** | **40** | **rank decrease, 27.39s after the requested line — NOT the allowed exception. This is `fe8b91e2`'s new `ConnectionStageTracker.retreat()` call firing at the end of a poke-loop iteration that found no answer, unconditionally, whether or not the bring-up ultimately succeeds.** |
| 7 | 22:42:35.210 | PHONE_ANSWERED | 50 | via D-POCO's own dial-back, not a successful poke (no `Successfully poked` line this run) |
| 8 | 22:42:35.672 | SENDING_CREDENTIALS | 55 | |
| 9 | 22:42:36.689 | PHONE_JOINING | 60 | |
| 10 | 22:42:37.363 | CONNECTING | 70 | |
| 11 | 22:42:37.523 | SECURING | 80 | |
| 12 | 22:42:37.665 | — | — | `SSL handshake complete` |
| 13 | 22:42:37.670 | STARTING_PROJECTION | 90 | last value |

Mechanism, confirmed against the source: `NativeAaHandshakeManager`'s poke loop calls
`ConnectionStageTracker.retreat(WAKING_PHONE, WAITING_FOR_PHONE)` unconditionally at the end of every
pass through `devicesToPoke`, right before `delay(POKE_RETRY_GAP_MS)` — regardless of whether that
pass is the last one before the phone answers. Two `Attempting active poke to device: POCO X3 NFC`
lines appear in this run (22:42:02.699, 22:42:17.289) before the phone's dial-back; the retreat fired
after the second one, ~12.5s before D-POCO reconnected. Any bring-up needing more than one poke round
— which is the ordinary case, not an edge case, whenever the phone's radios take more than
`POKE_RETRY_GAP_MS` (15s) to come up — now shows this decrease.

**Run 2 — the same condition PASSED.** Identical settings and protocol, re-run back to back.

| time | value | note |
|---|---|---|
| 22:44:46.476 | ARMED | |
| 22:44:46.536 | PREPARING_NETWORK | |
| 22:44:46.873 | WAITING_FOR_PHONE | |
| 22:44:46.896 | — | `startNativeAaQuietHost() requested` |
| 22:44:46.897 | PREPARING_NETWORK | allowed exception, 1ms after the requested line |
| 22:44:47.169 | WAKING_PHONE | |
| 22:44:47.210 | — | `Attempting active poke to device: POCO X3 NFC` (attempt 1) |
| 22:45:02.425 | — | `Attempting active poke to device: POCO X3 NFC` (attempt 2, 15.2s later — no `WAITING_FOR_PHONE` printed in between) |
| 22:45:19.325 | — | `Successfully poked POCO X3 NFC via HSP-AG. Holding 15000ms...` |
| 22:45:19.560 | PHONE_ANSWERED | |
| 22:45:19.560–22:45:21.983 | SENDING_CREDENTIALS → STARTING_PROJECTION | monotonic, no decrease |
| 22:45:21.978 | — | `SSL handshake complete` |

Discard-rule check (re-scoped to this run's own pid, since the capture file's tail was later
polluted by P3's launch — see Setup notes item 1): `MATCH!`=0, `createGroup SUCCESS`=1, `SSL`=1,
clean.

**Verdict basis:** two identical setups, one FAIL and one PASS on the exact condition the brief calls
out as the most valuable thing this run can find. The mechanism is understood and deterministic in
code (`retreat()` fires whenever an outer poke-loop pass ends without an answer while the pill is
still `WAKING_PHONE`); what varies between runs is only *how many* poke-loop passes complete before
the phone answers relative to when each pass's `pokeDevice()` call itself returns, which this rig
does not control precisely. Reporting both runs rather than picking one: **this is FAIL, and
intermittent** — a bring-up that takes just one poke-loop pass longer to succeed will show the
regression, and there is no guarantee a given real-world bring-up lands on the "run 2" side.

### P2 — the pill does not cover the home controls

**PASS** — bounds check still clears, but the pill is now substantially bigger.

- Setup: D-HU mode 3, D-POCO BT+WiFi OFF (no phone). `uiautomator dump` + `screencap` while the pill
  showed `WAKING_PHONE`.
- Screen: 1440×720 landscape, density 240 — unchanged from round 1/2.
- `auto_connect_pill` bounds: **`[544,591][896,696]`, 352×105px** — up from round 1/2's
  `[565,625][874,696]` (309×71px): **+43px wide (+14%), +34px tall (+48%)**. The height jump is the
  third line (`0f2a65ce5`'s WiFi Direct channel detail, replacing the separate toast); the width gain
  is the commit's stated "10% bigger" plus room for the longer channel string.
- Other clickable nodes, unchanged from round 1/2: `self_mode_button [84,220][312,448]`,
  `usb_button [432,220][660,448]`, `wifi_button [780,220][1008,448]`,
  `settings_button [1128,220][1356,448]`, `exit_button [1284,624][1416,696]`.
- Top-row buttons end y=448, pill now starts y=591: 143px gap (was 177px). `exit_button` starts
  x=1284 vs the pill's new end x=896: 388px gap (was 410px). **No intersection**, on either axis,
  despite the growth.

Evidence: `evidence/p2-ui-round3.xml`, `evidence/p2-screen-round3.png`.

### P3 — a failed attempt falls back to the resting line

**FAIL** (the brief's literal condition — "a later `status pill step: ARMED`" — is not met), but
**the underlying defect round 1/2 found is gone.** This is a real, measured change in behaviour, not
a null result.

- Setup: D-HU mode 3, D-POCO BT+WiFi OFF for the **entire run, 246s** (over double the brief's 120s
  floor, extended specifically to see whether the pill ever reaches `ARMED`).
- Discard-rule check: clean (`MATCH!`=0, `createGroup SUCCESS`=1, `Magic Garbage`=0).
- `status pill step:` count: **15** (≥3 ✓). `WAITING_FOR_PHONE` reached ✓ repeatedly. `hidden` count:
  **0** ✓.
- **`ARMED` appears exactly once — the first value at launch (22:48:13.236) — and never again in 246s.**
  Round 1/2's defect (the pill permanently latched on `WAKING_PHONE` with zero further changes for the
  whole run) is **not reproduced**: this candidate's pill visibly moves, cleanly and repeatedly,
  oscillating `WAKING_PHONE` (45) ↔ `WAITING_FOR_PHONE` (40) on a steady cadence for as long as the
  phone stays unreachable:

  | from | to | gap |
  |---|---|---|
  | WAKING_PHONE 22:48:13.882 | WAITING_FOR_PHONE 22:49:00.563 | 46.68s |
  | WAITING_FOR_PHONE 22:49:00.563 | WAKING_PHONE 22:49:15.586 | 15.02s |
  | WAKING_PHONE 22:49:15.586 | WAITING_FOR_PHONE 22:49:46.416 | 30.83s |
  | WAITING_FOR_PHONE 22:49:46.416 | WAKING_PHONE 22:50:01.440 | 15.02s |
  | WAKING_PHONE 22:50:01.440 | WAITING_FOR_PHONE 22:50:32.238 | 30.80s |
  | WAITING_FOR_PHONE 22:50:32.238 | WAKING_PHONE 22:50:47.259 | 15.02s |
  | WAKING_PHONE 22:50:47.259 | WAITING_FOR_PHONE 22:51:18.148 | 30.89s |
  | WAITING_FOR_PHONE 22:51:18.148 | WAKING_PHONE 22:51:33.165 | 15.02s |
  | WAKING_PHONE 22:51:33.165 | WAITING_FOR_PHONE 22:52:03.989 | 30.82s |
  | WAITING_FOR_PHONE 22:52:03.989 | WAKING_PHONE 22:52:19.011 | 15.02s |

  The 15.02s leg is `POKE_RETRY_GAP_MS`'s `delay()`; the ~30.8s leg is a poke-loop pass itself (two
  failed protocol attempts, HFP-AG then HSP-AG, against a phone with Bluetooth fully off).

- **Reading the brief strictly, this is a FAIL**: the pill never returns to `ARMED`, only to
  `WAITING_FOR_PHONE`, so it never reaches what the brief calls "the resting line." But the fix
  measurably does what its own commit message says — a wake round with no answer now visibly gives
  the pill back, rather than leaving it stuck. Whether "resting" should mean `ARMED` specifically or
  any return to a lower, non-`WAKING_PHONE` stage is a brief-language question, not a hardware
  question; the round can only report which one this build actually does. Full capture kept:
  `r1-p3-dhu.txt`.

### P4 — a different mode produces a different sequence

**PASS**

- Settings: `wifi-connection-mode=1`, otherwise as P1. `WifiLauncher: Initializing WiFi Mode: AUTO`
  confirmed (22:54:03.946).
- `status pill step:` values: ARMED → SEARCHING (count 2, ≥2 ✓), `NSD Registered: AAWireless`
  follows (22:54:04.988). `WAKING_PHONE`=0 ✓, `SENDING_CREDENTIALS`=0 ✓, `SEARCHING`=1 ✓ — unaffected
  by either of this round's two commits, as expected (mode 1 never touches the WiFi Direct / native
  poke path either commit changed).

### P5 — the pill renders two lines

No PASS/FAIL of its own, but the round's other confirmed fix. **The pill now renders three lines**,
captured incidentally in P2's screenshot (mid-`WAKING_PHONE`, `evidence/p2-screen-round3.png`): bold
"Android Auto is starting…", "Waking your phone", and a new third line "WiFi Direct on 5 GHz (5765
MHz)" — all legible, none truncated or overlapping. **This directly answers round 1/2's addendum
finding** (the 5GHz-band toast visually covering the pill on every bring-up): `0f2a65ce5` removes
that toast entirely and folds its content into the pill's own layout, so there is no longer a second
UI element competing for the same screen position. No further toast-covering-pill occurrence was
observed in any of this round's five runs.

### P6 — USB

Not re-run. Pre-registered INCONCLUSIVE on this rig (no USB host), unaffected by either commit.

### WB1 — the status pill must also appear when using the WiFi button (added mid-round)

**FAIL.** Not one of the brief's original runs — added mid-round on request, because every run above
only exercises the *automatic* app-launch bring-up. The manual path was never checked. **Confirmed
pre-existing on `ef66abbf` too** (the code in question is byte-identical across both SHAs; neither of
this round's two commits touches it), so this is not a regression from this branch, but it is a real
gap the brief should have covered and any future pill work should close.

- Setup: from a fresh Native AA session (formed automatically per P1's protocol), disconnected with
  `headunit://disconnect` — confirmed pill goes to `hidden` (23:00:13.143) and the group tears down.
  Then `adb shell input tap 894 334` (the `wifi_button` centre, `[780,220][1008,448]` per P2's dump).
- D-POCO was reachable this time (both radios on, already bonded and recently seen), so the button
  press resolves a single known driver automatically (`NativeDriverSelectionMode.DISABLED`, one
  candidate) and connects without showing a selector.

| ts | line |
|---|---|
| 23:00:41.740 | `HomeFragment.connectToNativeDevice: Connecting to Native-AA device: POCO X3 NFC` |
| 23:00:41.741 | `MainActivity.beginAutoConnect: Auto-connect: begin (Native-AA driver: POCO X3 NFC, mode=OVERLAY)` |
| 23:00:41.843 | `NativeAA: ACTIVELY LISTENING on Android Auto UUID...` |
| 23:00:42.701 | `WifiDirectManager: 5GHz createGroup SUCCESS!` |
| 23:00:43.734 | `NativeAA: Connection accepted from POCO X3 NFC` |
| 23:00:45.130 | `NativeAA: [RX] WifiStartResponse ip= status=SUCCESS(0)` |
| 23:00:50.349 | `AapSslContext.performHandshake: SSL handshake complete.` |

**Zero `status pill step:` lines appear anywhere in this 8.6s window** — not `ARMED`, not
`WAKING_PHONE`, not even a `hidden` transition line, despite the connection actually running its full
course (group create, poke skip because the car-kit link was live, handshake, SSL) end to end.

Root cause, read from source (`MainActivity.kt`, present identically on `ef66abbf` and `fe8b91e2`):
the WiFi button's single-driver path calls `beginAutoConnect(..., ConnectionUiMode.OVERLAY)`, and
`renderStagePill()` computes `overlayOwnsScreen = autoConnectInProgress && autoConnectMode ==
ConnectionUiMode.OVERLAY`, forcing `shown = null` — and therefore suppressing the `status pill step:`
log entirely — for the whole attempt. `showAutoConnectOverlay()` shows a full-screen splash instead,
but that splash's text is a single static string (`"Android Auto is starting…"` or a settings
override) set once at overlay-show time; it never subscribes to `ConnectionStageTracker.stage`, so
the user sees no `WAKING_PHONE` / `PHONE_ANSWERED` / `SENDING_CREDENTIALS` progression at all for a
WiFi-button-initiated connect, only a spinner. The pill only ever appears for the automatic
app-launch bring-up path (`ConnectionUiMode.PILL`, or `PILL_THEN_OVERLAY` before it hands off).

This is scope for a future brief, not this branch: neither `0f2a65ce5` nor `fe8b91e2` changed
`ConnectionUiMode`, `beginAutoConnect`, `renderStagePill`, or any call site that chooses OVERLAY over
PILL. Flagging here because the requirement — "the status pill must also appear when pressing the
WiFi button" — was raised for this results file specifically; it is not something round 1's or
round 2's brief asked either. Evidence: `evidence/wb1-dhu.txt.gz`.

---

## Part B / Part C — not re-run this round

See Setup notes for the scope decision. `W1/W1b/W2/W2b/W3` (poke targeting under a car-kit link) and
`D1/D2/D3` (auto-disconnect grace) exercise code neither `0f2a65ce5` nor `fe8b91e2` touches, and both
passed twice already on `ef66abbf` (round 1 and round 2 results). Re-running them would not exercise
either new commit.

## Report-back summary

1. **P1 is a regression from `fe8b91e2`**, and it is intermittent: 1 FAIL / 1 PASS across two
   identical setups on the same rank-monotonicity condition the brief flags as the most valuable
   thing Part A can find. Mechanism: the new `ConnectionStageTracker.retreat(WAKING_PHONE,
   WAITING_FOR_PHONE)` call fires unconditionally at the end of every poke-loop pass that found no
   answer, including passes inside an otherwise-successful bring-up.
2. **P3's original defect (permanent latch on `WAKING_PHONE`) is fixed** — confirmed over a 246s
   unanswered run, the pill now cycles cleanly between `WAKING_PHONE` and `WAITING_FOR_PHONE` on a
   ~15s/~31s cadence — but the brief's literal condition (a return to `ARMED`) is not met; the fix
   returns to `WAITING_FOR_PHONE`, not all the way to rest.
3. **P2 still clears with the pill's new size** (352×105 vs 309×71, +14%/+48%), by a comfortable
   margin (143px / 388px to the nearest control).
4. **P5's toast-over-pill finding from rounds 1/2 is resolved**: the toast is gone, its content is
   now the pill's own third line.
5. **New requirement, WB1: the status pill must also appear when pressing the WiFi button, and today
   it does not.** A WiFi-button-initiated connect (single known driver, mode 3) runs its full course —
   group create, handshake, SSL — with zero `status pill step:` lines; it shows a static, non-updating
   splash instead. Confirmed pre-existing on `ef66abbf` as well, so not a regression from either of
   this round's commits, but a real gap this branch's own subject matter should probably close before
   or alongside it.

## Anything the brief did not ask about

- **P1 and P3's fixes are in direct tension.** The mechanism that makes P3 pass partially
  (`retreat()` at the end of every failed poke-loop pass) is exactly what makes P1 intermittently
  fail. A fix that only retreats once some give-up threshold is crossed (rather than after every
  single pass) would likely resolve P1's regression without reintroducing P3's original permanent
  latch — but that is a design call for the coding session, not something this round should decide.
- **P3's own bar may be wrong for what `fe8b91e2` was built to do.** The brief was written before
  this fix existed and named `ARMED` specifically. If `WAITING_FOR_PHONE` is an acceptable "resting"
  state for an in-progress bring-up (arguably more honest to the user than `ARMED`, since the
  listeners are still open and a poke is still being retried), the brief's P3 condition should be
  updated rather than the fix pushed to actually reach `ARMED` mid-retry-loop, which would itself
  misrepresent that the group and listeners are still live.
- Round 1/2's FX Plus classifier gap and D2/D3 IPv6-only bind finding are untouched by this round
  (neither is exercised by Part A) and remain open follow-ups from round 2's results file.
- **WB1's gap is worth folding into whatever fixes P1's regression.** Both are about which UI element
  shows connection progress and when: P1 is "the pill moves when it should hold still", WB1 is "the
  pill never shows up at all for a manual connect". A future brief should probably grade the WiFi
  button (and the analogous Self Mode / USB buttons, which take the same `OVERLAY` path) explicitly,
  rather than only ever testing the automatic launch-time bring-up the way rounds 1-3 have.
