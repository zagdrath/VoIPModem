# V.90 start-up failure with a physical modem

Investigated 2026-11-25 on branch `investigate/v90-physical-modem`.
Status: **cause found in the modem engine (libv90modem), fixed in our fork
of it, and confirmed live on 2026-11-25: the Courier connects at V.90
through the gateway. Still open: the upstream runs at only 7200 bit/s.**
See "The fix" and "Live result".

## Summary

- The failure is in libv90modem's V.90 Phase 3, not in VoIPModem's audio
  path and, as far as this call shows, not in the gateway.
- The Courier trains on our TRN1d and answers our Jd with the signal S, as
  V.90 expects. The engine's detector for that S never accepts it, so the
  engine keeps sending Jd. The Courier gives up after 625 ms of S and
  restarts Phase 2 (Tone A). The engine does not follow Tone A in Phase 3
  and ends at its time limit: `P34_ERR_NO_S2`, reported as
  `V90M_END_FAILED` ("start-up failed").
- The miss depends on where the engine's symbol sampling falls on the
  Courier's S. Moving the Courier's S by one sample (125 us) either way in
  the recording makes the same engine accept it within 17 ms and go on.
- VoIPModem's transmit path was exact for this call except for one
  codeword: mu-law "negative zero" (0x7F) goes out as "positive zero"
  (0xFF). 72 samples of the 66,335 sent in the PCM phase. Both decode to
  the same level, so this is not the cause; see question 1.
- The earlier reading, "the Courier does not acquire VoIPModem's training
  signal", is wrong. It does.

## Setup and evidence

VoIPModem 0.0.1 answering on Windows Server 2022, libv90modem 0.6.0
(submodule at `d729110`, 0.6.0-rc1), offering V21,V22,V32BIS,V34,V90
without error control. Caller: US Robotics Courier V.Everything on an FXS
gateway (Cisco SPA112 and AudioCodes MP-124E, same failure on both). RTP
G.711 mu-law, 20 ms, relayed through Asterisk 20. Gateway: echo canceller
off, fixed jitter buffer, modem/fax transparent, silence suppression off,
gains 0 dB.

Recordings (8 kHz 16-bit WAV decoded from packet captures taken on the
VoIPModem host, samples placed by RTP timestamp; not in the repository):

- `v90-failed_voipmodem-tx.wav`, `v90-failed_courier-via-gateway-tx.wav`,
  `v90-failed_stereo_L-voipmodem_R-courier.wav`
- `v34-ok_*.wav`: the same three for a working V.34 call (not analysed
  here beyond the measurements below).

Measured from the captures before this investigation: no packet loss or
sequence gaps either way; packet pacing against the RTP clock +7.5 ppm
(VoIPModem) and +28.7 ppm (gateway); V.34 symbol rate 3199.997 Hz from
VoIPModem and 3199.837 Hz (about -51 ppm) from the Courier through the
gateway; the Courier arrives at about -10.7 dBm0; the gateway applies
about 10 dB of loss toward the phone port.

Unexplained and not used: the Courier's "Timing Offset -9088 ppm" on the
V.34 call.

## Method

Everything below was done offline from the stereo recording of the failed
call, with the engine's sources built by gcc outside the repository. No
change was made to VoIPModem or to the submodule.

1. **An offline copy of `CallAudioBridge`'s path**: an answering modem
   configured as the bridge configures it, fed through
   `v90m_process_linear`, its output encoded with pjmedia's own
   `pjmedia_linear2ulaw_tab` (compiled from
   `pjmedia/src/pjmedia/alaw_ulaw_table.c`). A second, identical modem ran
   on the same input through `v90m_process_g711`: the codewords the engine
   intends.
2. **Alignment.** The engine is deterministic, so if the recording is
   what the engine heard, the copy must reproduce what was sent. The
   transmit stream starts on a frame boundary (answer tone at sample 1920
   of the recording, 1600 samples after the modem starts). The receive
   delay between the wire and the engine is not known in advance, so it
   was swept.
3. **`v90m-replay`** (the engine's own tool, built with its trace hooks)
   on a recording file made from the WAV with that alignment.

Result of the alignment: with a receive delay of **1091 samples (136.4
ms)** the offline copy reproduces **all 112,320 samples VoIPModem sent,
with no difference**, from the answer tone to the cleardown. That one
number holds for the whole call, so the delay was constant.

This tells us several things at once:

- The recording is exactly what the engine heard: no packet-loss
  concealment, no jitter-buffer slip or discard, no resampling, on the
  receive side during this call.
- Nothing between the engine and the wire changed a sample other than the
  G.711 encoder itself: no level adjustment, VAD, echo canceller, DC
  removal or dither.
- The failure can be replayed at will from the recording.

Times below are the engine's clock (seconds since the modem started).
What VoIPModem sent appears in the recording 0.040 s later; what the
recording shows from the Courier reached the engine 0.096 s later.

## Question 1: is the transmit path bit-exact for PCM modulation?

**Almost. One codeword of 256 is changed, and it did not matter here.**

- **Interface.** libv90modem has three line interfaces
  (`third_party/libv90modem/include/v90modem.h`, `docs/line-interface.md`):
  `v90m_process_g711` (codewords both ways; "the one V.90 / V.92 need"),
  `v90m_process_lin_rx` (linear in, codewords out) and
  `v90m_process_linear` (linear both ways; "a PCM modulation then sends
  the linear value of its codewords", which "only works if the codec turns
  it back into the same codeword"). VoIPModem uses `v90m_process_linear`
  (`src/Lib/Sip/CallAudioBridge.cpp`), because a pjsua2 audio port carries
  16-bit PCM and PJSIP does the G.711 encoding.
- **Is every linear value turned back into its codeword?** Checked for
  all 256 codewords of each law, `v90m_g711_decode` then pjmedia's encode
  table:
  - mu-law: 255 of 256 return unchanged. **0x7F (negative zero) returns
    as 0xFF (positive zero)**: both are linear 0, and a 16-bit sample
    cannot carry the sign of zero.
  - A-law: all 256 return unchanged.
  - Receive: pjmedia's decode tables equal the engine's for all 256
    codewords of both laws.
- **Is anything else altering samples?** By reading: PCMU/PCMA only,
  8 kHz throughout, VAD off, echo canceller off (`ecTailLen = 0`), PLC
  off, and the conference bridge copies samples unchanged when a port has
  one source and the default level (`pjmedia/src/pjmedia/conference.c`;
  this build selects the parallel bridge, `conf_thread.c`, which was not
  read line by line). By measurement, which covers whichever bridge is
  built: zero differences in 112,320 samples (Method).
- **In this call**: the engine's PCM output phase (Phase 3 on) ran for
  66,335 samples. The engine's intended codeword was 0x7F in 72 of them,
  all in the second after Sd began, and each went out as 0xFF. No other
  sample of that phase differed. (Outside the PCM phase 1,890 samples
  differ between the two interfaces. That is rounding: pjmedia's encoder
  and the engine's choose a different neighbour for a value between two
  levels. It does not concern PCM modulation.)

Does the 0x7F change matter? Both codewords are the same output level of
a mu-law decoder, and this call shows the Courier training on the signal
that contained them. It is still a deviation from "exact codewords", it
is specific to mu-law, and it would matter on a path where the far end
sees the codewords themselves (a digital far end, or anything that
checks them). Removing it means bypassing PJSIP's encoder for the
transmit direction, which is not a small change; see Next steps.

**The temporary debug tap was not built.** The wire capture already
equals an exact offline copy of the path for the whole call, which is the
comparison the tap would have made.

## Question 2: what does the engine do when the caller restarts Phase 2?

**In Phase 3 it does nothing, by design, and then fails.**

- Tone A (2400 Hz, the caller's retrain request) is watched only from
  Phase 4 on and in data mode. `modem/p34d.c` says why: "Not in Phase 3,
  where the caller's own signals are not all known to stay clear of 2400
  Hz." So the Courier's Tone A at 12.5 s goes unheard.
- The state the engine is in, `P34_JD` (send Jd until S is heard), has one
  exit other than S: its time limit, 5100 ms plus a round trip from the
  start of TRN1d (`T_S_AFTER_JD`), then `fail(p, P34_ERR_NO_S2)`.
- `tick_v90` (`modem/ans_v90.c`) turns a failed Phase 3/4 into
  `ans_fail(a, ans_retrain_or(a, ANS_END_FAILED))`. Before the first
  connection that is `V90M_END_FAILED`. There is no retrain and no
  fallback to V.34 from there. (The V.92 scheme, `tick_v92`, does answer a
  failed start-up with a retrain; the V.90 scheme does not.)
- **The API offers nothing VoIPModem is leaving off.** `v90m_retrain` is
  for data mode only (`V90M_ESTATE` otherwise), and `v90m_cfg` has no
  setting for retrain or fallback during start-up. A caller's INFO1a can
  select V.34 inside a V.90 Phase 2, but that is the caller's choice.
- **The ROADMAP's Tone B note is a different case.** There, our side
  began a retrain (70 ms of silence, Tone B, Phase 2 again:
  `ans_v90_retrain`) on a V.92 call that was already past Phase 2 with
  BinModem. That machinery exists and runs in data mode and in a V.92
  start-up. It is not reached from a V.90 Phase 3.

The restart is a consequence, not the cause: the Courier restarts because
the engine did not answer its S (question 3).

## Question 3: does the engine receive the Courier's training correctly?

**Yes, up to the S that answers Jd. That S is where it stops.**

`v90m-replay` can be driven from these WAVs. It wants, per 10 ms, 80
codewords heard then 80 sent, and either 200 ms of silence before the
answer tone or `-s SAMPLE`. The recording file was made from the stereo
WAV with the alignment above (heard = the Courier's channel delayed 771
samples against a transmit channel advanced 320, so that sample 0 is the
modem's start) and run as `v90m-replay -r 3000 -d 0 -s 0`. Its report
agrees with the live log: the modem stops with `V90M_END_FAILED` at
14.033 s.

What the engine made of the call:

| Engine time | What happened |
|---|---|
| 0.200 - 3.038 s | ANSam; CM heard at 2.165 s, JM sent |
| 3.113 - 5.741 s | Phase 2 complete: INFO0, probing, L1/L2, INFO1d, INFO1a. Round trip 1462 samples; upstream 3200 symbols/s, carrier 1920 Hz |
| 5.741 s | Phase 3 begins; the Courier's S / S-bar found at 5.856 s |
| 5.856 - 8.696 s | Equaliser trained on PP and TRN: error -34 to -37 dB |
| 8.696 s | Ja decoded: a valid DIL descriptor (N 197). We send Sd, Sd-bar, TRN1d. The Courier stops sending within 0.4 s, as it should |
| 8.8 - 9.785 s | TRN1d; our echo found 1452 samples away and cancelled |
| 9.785 s | We send Jd and wait for S |
| 11.850 s | **The Courier's S arrives** (625 ms, -18.0 dBm0; lines at 1920, 3520 and 320 Hz, the same three as its first S at 5.7 s). **Not accepted** |
| 12.5 s | The Courier's Phase 2 tone again (2400 Hz, with 1800 Hz 7 dB below it). Not watched in Phase 3 |
| 14.033 s | `P34_ERR_NO_S2`; `V90M_END_FAILED` |

So the Courier got through Phase 2, sent a decodable Ja, heard Sd, kept
silent through TRN1d and answered Jd with S: it acquired our downstream
training signal through the gateway.

**Why the S is not accepted.** After a silence the engine re-acquires S
with `sres_symbol` (`modem/v34train.c`). Its first state, `SRES_WAIT`,
counts symbols whose energy is above a threshold (|z| above 0.71 of the
reference) and needs 14 **in a row**; one symbol below it sets the count
back to zero. During the Courier's S the engine's symbols alternate
between about 1.55 and about 0.45 of the reference, every other one below
the threshold, for the whole 625 ms. The count reaches 5 while the signal
builds up and never passes 1 after that. The later state `SRES_TIME` has
the same kind of test (a symbol under 0.5 of the reference means "it went
away again").

S is two points sent alternately, so a strong and a weak sample in turn
is what S looks like when the sampling instant is off and the channel is
not flat; the mean of each pair is right (about 1.0 of the reference).

**It depends on the sampling instant.** The same recording with the
Courier's S moved by whole samples (one sample is 0.4 of a symbol),
unmodified engine:

| S moved by | Result |
|---|---|
| -3, -2, -1 samples | S accepted at 11.867 s; engine goes on to Jd' and the DIL |
| 0 (as recorded) | never accepted; `P34_ERR_NO_S2` at 14.033 s |
| +1, +2, +3 samples | S accepted at 11.867 s; engine goes on to Jd' and the DIL |

**A prototype change** (outside the repository, to test the diagnosis
only): judging the energy over a pair of symbols in `SRES_WAIT` and
`SRES_TIME` instead of symbol by symbol. With it the engine accepts the S
as recorded, and at -2 to +2 samples. Not run against the engine's own
test suite.

**Why every call, and why not with BinModem** (inference, not measured):
between the end of Ja and this S the Courier is silent for about 3 s, and
the engine's symbol timing is frozen meanwhile. At the -51 ppm measured
for the Courier through the gateway, 3 s is about 150 us, half a symbol
at 3200 symbols/s. If the Courier's symbol clock runs on through its
silence, the S lands on the same bad instant on every call with this
modem and gateway. A software modem on the same clock has no such drift.

The recording ends there: what the Courier does after a Jd' and a DIL
cannot be seen from it.

## Question 4: what cannot be decided from code or from this recording

1. **Whether the Courier completes V.90 once its S is answered.** The
   DIL, Phase 4 and data mode have never run against it. Only a live call
   with a changed engine shows that.
2. **Whether the gateway passes the PCM downstream well enough for data.**
   Sd, TRN1d and Jd got through well enough for the Courier to follow
   them, but they use few levels. A V.90 data constellation needs every
   codeword to come out of the gateway's D/A as its own level, at the
   network's sample clock. Open here: whether the 10 dB of loss toward the
   phone port is analogue or applied to the samples (a digital pad, which
   V.90 measures with the DIL but which costs rate); whether the gateway
   ever adds or drops a sample to follow our clock (the 21 ppm between the
   two pacing figures is one sample every 6 s); its codec and filter.
   What would settle it: the rate and the Courier's diagnostics after a
   connected V.90 call; or a recording of the gateway's analogue output
   while VoIPModem sends a known codeword sequence.
3. **Why the Courier sends S for 625 ms and then retrains**, rather than
   S, S-bar and on. It looks like "S until Jd' is seen, then give up".
   Only the answered case shows what it does next.
4. **Whether 0x7F sent as 0xFF matters to a real far end.** Not for this
   call's failure. For data mode, a constellation that uses level 0 is
   unlikely but not excluded; a codeword-exact transmit path would remove
   the question.
5. **The half-symbol explanation** for "every call": needs more than one
   recording, or the Courier on a different gateway clock.

## Not checked

- The working V.34 call's recordings were not run through the offline
  copy.
- `conf_thread.c` (the bridge this build uses) was not read line by line;
  the measurement stands in for it.
- A-law on the wire: the table check passes, no call was analysed.
- VoIPModem calling the physical modem.

## The fix

In our fork of libv90modem (`zagdrath/libv90modem`, branch
`fix/s-after-jd-timing`, one commit on top of 0.6.1); the submodule points
there. Nothing in `third_party/libv90modem` is patched in-tree.

- **The change** (`modem/v34train.c`, `sres_symbol`): a symbol of S counts
  by itself, as before, or as a pair with the one before it when the two
  together hold the power of S. The "it went away again" test in the next
  state is done the same way. A run of symbols that passed before is
  counted exactly as before.
- **A unit test that fails without it** (`test/test_sres.c`): S after a
  silence on a simulated line with a first-order high-pass at 500 Hz and a
  -50 ppm clock, the silence stepped over a whole symbol of drift in
  sixteenths. Before: found at 12 of 16 timing phases (missed from 0.375
  to 0.56 of a symbol). After: 16 of 16, and the 800 symbols after S-bar
  are received without error. This also reproduces the mechanism without
  the recording.
- **The recording as a fixture** (`test/fixtures/usrc1.raw.xz` in the
  fork): before, the trace stays in Jd until 14.033 s; after, Jd' and the
  DIL at 11.867 s.
- **Nothing else moved**: the fork's seven other real-modem recordings
  replay with identical traces. Of its 85 test programs' outputs one
  figure changed, explained in the commit (a V.34 retrain takes its S
  sooner; SNR 31.3 to 31.4 dB).
- **Not run**: the fork's ARM suite, its bench hashes and its maintainer's
  private recordings; `make check` as a whole does not build with this
  machine's gcc (one benchmark program needs `clock_gettime`).
- **VoIPModem**: the submodule moves from 0.6.0-rc1 to 0.6.1 plus the
  fix. `libv90modem.vcxproj` gains `api/v90modem_diag.c`, new in 0.6.1.
  The "modem ended" log line now carries the engine's own cause, e.g.
  `start-up failed (P34_ERR_NO_S2, furthest phase3, V.90)`. Built here
  with the v142 toolset, x64 Debug: 160 tests pass. Not built here:
  Win32, the XP build, Linux.

## Next steps

1. **Live tests** (below). Until test 2 passes, keep lines limited to
   V.34 for callers behind a gateway.
2. **Send the fix upstream**: `libv90modem-upstream-issue.md` in this
   folder is the text for the pull request.
3. **VoIPModem, for decision, not done**: (a) call-audio recording as a
   switchable diagnostic in the format `v90m-replay` reads, so the next
   failure needs no packet capture and no alignment; (b) a codeword-exact
   transmit path, which removes the 0x7F deviation.

## Live result (2026-11-25)

Build `5352fcf` (x64 Release, engine 0.6.1 plus the fix), the Courier
calling through the AudioCodes MP-124E, V.42 on. Not yet repeated
through the SPA112.

| | |
|---|---|
| Manager, one call of over two minutes | V.90, 49333 down / 7200 up, LAPM, no compression; one rate renegotiation after 80 s; data both ways |
| Courier `ATI6` after a call | 50666/7200, LAPM 128/15, 423 blocks sent, 6 resent, 5 BLERs, 1 retrain granted |
| Courier `ATI11` | V.90/V.34+, 8000/3200 symbols/s, upstream carrier 1920 Hz, receive level -25.1 dBm, SNR 24.1 dB, near echo loss 21.1 dB, round trip 176 ms, V.90 status `uu,5,12Y,12.5,0,0N,0,41.4,64.2` |
| A captured call of 37 s | V.90, 45333 down / 7200 up at 17.0 s after the modem started |

So the start-up fault is fixed on the live line: the engine takes the
Courier's S 2.1 s into Jd, sends Jd' and the DIL, and Phases 3 and 4
complete.

### Why the upstream is 7200 bit/s

The captured call was analysed like the failed one. With a receive delay
of 1001 samples the offline copy of the bridge reproduces all 136,160
samples sent up to the connection, so the engine's trace of it is exact.

- At the start of Phase 4 (the Courier's CPt) the engine's upstream
  receiver is clean: error -33 dB, "upstream SNR 28.0 dB".
- Within a second of our TRN2d starting (our downstream at full power)
  the error rises to -17 dB and stays there, in data mode too.
- The engine sets the upstream ceiling from that figure at the end of
  TRN2d (`p34d.c`, `(snr - 9 dB) / 3 dB` bits per symbol): 7200 bit/s.

The -17 dB is the echo of our own downstream:

| Part of the call | We send | We hear | Echo of what we send |
|---|---|---|---|
| ANSam, before the Courier's CM | -14 dBm0, 2100 Hz | -67.8 dBm0 | none: 2100 Hz below -88 dBm0 |
| TRN1d and Jd, the Courier silent | -12.4 dBm0 | -67.7 dBm0 | none: more than 74 dB down |
| Data mode, both sending (18 s) | -12.3 dBm0 | -17.0 dBm0 | **-35.0 dBm0**, 179.5 ms late, 18 dB under the Courier's signal |

(Cross-correlation of the two streams over lags up to 500 ms; in data
mode the result is 11 dB above what the method can resolve.)

Two things follow.

1. **The gateway returns our echo only while the caller is sending.** A
   two-wire port cannot have 74 dB of return loss by itself: something in
   the gateway shuts the return path, or cancels it, while the phone side
   is quiet (an echo suppressor, the non-linear part of an echo canceller,
   or a noise gate), whatever its echo-canceller setting says.
2. **The engine trains its echo canceller exactly then.** V.90 has the
   digital modem learn its echo during TRN1d, with the caller silent
   (`p34d.c`, `P34_SD`). It finds nothing to learn (the trace shows the
   canceller taking 0.0 dB off, and its delay search settling on 1404
   samples where the echo is at 1436), and it does not adapt later. So in
   Phase 4 and in data mode the echo is not cancelled at all, and 18 dB is
   the upstream's signal-to-echo ratio.

V.34 on the same line reached 24000 bit/s upstream because there the
Courier arrives 7 dB stronger (-10.7 against -17 dBm0), our signal is
2 dB weaker, and the V.34 engine has a second canceller for the far echo.

Also seen: in this call the engine's intended codeword was mu-law negative
zero (0x7F, sent as 0xFF) 10,488 times, nearly all in the DIL, which the
Courier asked to include level 0. The Courier completed the DIL and chose
a constellation, so this did not stop it; whether it cost anything is not
known.

### What to try next

1. **Gateway**: find and turn off whatever gates the return path. These
   calls were on the MP-124E: look at its non-linear processor (NLP)
   setting, which is apart from the echo canceller's on/off, and at what
   it does to the echo canceller once it has switched a call to modem
   transport. On the SPA112 the echo suppressor ("Echo Supp Enable") is a
   setting of its own, apart from "Echo Canc Enable" and "Echo Canc Adapt
   Enable"; all three off. The setting names were not checked against
   either unit. Check with a capture: during our answer tone, before the
   Courier sends anything, 2100 Hz should now be visible in what we
   receive (somewhere near -35 dBm0 instead of nothing). If it is, the
   engine's canceller has something to train on, and the upstream rate of
   the next V.90 call shows whether that was all.
2. **If the gateway cannot be made to pass the echo**: the engine would
   have to learn the echo with the caller sending (in Phase 4 or data
   mode). That is a change to libv90modem, not a setting.
3. Not a fix, to bound the problem: `max_tx_code` in `v90m_cfg` lowers
   the downstream power the caller may ask for, and with it the echo;
   VoIPModem does not expose it.

### After turning the gateway's echo canceller off (2026-11-25)

One setting changed on the MP-124E: Echo Canceller set to Disable
(`EnableEchoCanceller`). `ECNLPMode` was not changed. Next call, captured:

| | Echo canceller on | Echo canceller off |
|---|---|---|
| Echo of our answer tone, caller silent | none (below -88 dBm0) | -26 dBm0 |
| Echo during TRN1d and Jd | none | -24.9 dBm0, 173.5 ms late, 12.5 dB under what we send |
| Engine's canceller | takes off 0.0 dB | takes off 35 dB |
| Upstream receiver error through TRN2d | rises to -17 dB | stays at -36 dB |
| Connection | V.90 45333 / 7200 | **V.90 49333 / 26400** |
| Courier: SNR, blocks resent | 24.1 dB, 6 of 423 | 26.1 dB, 0 of 727 |

So the 7200 bit/s was the gateway's echo canceller, as read above: with
it off the echo is there all the time (and strong: 12.5 dB of return
loss), the engine learns it during TRN1d and cancels it, and the upstream
gets the rate the line supports. The offline copy again reproduces every
sample sent up to the connection (135,680, receive delay 1007).

That call ended after 17 s of data mode with both directions still at
full level in the last packets: it was hung up, not lost by the modems.
The Courier logged "Loss of Carrier" and one link timeout; which side
ended it is not known from the capture.

For gateways: **the echo canceller of an FXS gateway must be off for
V.90 calls**, not only for the usual reason, but because it hides the echo
from the digital modem while that modem trains.

## Live tests

Test 1 is for the unchanged build (0.0.1 as deployed); the others need a
build from this branch.

1. **Is it the same instant every call?** Unchanged build, three or more
   V.90 attempts, packet capture each. Decode as before. Confirms the
   half-symbol explanation if every attempt shows the Courier's S of about
   625 ms starting about 3.0 s after its Ja ends, followed by the 2400 Hz
   tone, and "start-up failed" about 14 s after answer. Rules it out if
   some attempts get past Jd (longer call, different failure, or CONNECT).
2. **Does the engine change fix start-up?** Build against the changed
   engine, default offer limited to V.90 (`Modulations=V21,V22,V32BIS,V34,V90`),
   Courier through the SPA112, capture. Success: `libv90modem: CONNECTED
   V.90 ...` in the log. On a failure the log now names the cause
   (`modem ended -- start-up failed (<error>, furthest <phase>, ...)`):
   anything other than `P34_ERR_NO_S2` means the call got further. Record the rates, the SNR, and the Courier's
   `ATI6` and `ATI11` after the call. If it still fails: the capture shows
   the new stopping point (DIL, CPt, MP), and that goes to the engine's
   maintainer.
3. **Does the gateway carry V.90 data?** On a connected call from test 2:
   downstream rate and retrains over five minutes, and a file transfer
   with error control on. A rate far below 40000 bit/s or repeated
   retrains points at the gateway's D/A path (question 4.2), not at the
   engine.
4. **Second gateway.** Repeat test 2 through the MP-124E. Same result on
   both rules out one gateway's firmware.
5. **Regression.** V.34 with the Courier, and V.90 with BinModem, on the
   changed engine: both must still connect as in `ROADMAP.md`.
