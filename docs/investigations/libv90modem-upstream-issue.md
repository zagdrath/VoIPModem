# Draft pull request for georgica/libv90modem

The text for a pull request from `zagdrath/libv90modem`, branch
`fix/s-after-jd-timing` (one commit on 0.6.1: the fix, a unit test, the
recording as a fixture). Background and method: `v90-physical-modem.md`
in this folder. The fix has not been tried on a live call; say so when
posting.

---

**Title:** v34train: S after a silence is taken on a tilted line at any timing phase (P34_ERR_NO_S2 with a USR Courier behind an FXS gateway)

## What happens

Answering a US Robotics Courier V.Everything (analogue, on an FXS gateway,
G.711 mu-law over RTP), V.90 start-up fails on every call with
`V90M_END_FAILED`, about 14 s after the answer. V.34 with the same modem
and path connects and holds (31200/24000, SNR 32 dB).

Phase 2 completes, Ja is decoded, the caller goes silent for Sd and
TRN1d, and then answers Jd with S. `sres_symbol` never leaves
`SRES_WAIT`, so `P34_JD` keeps sending Jd. The caller sends S for 625 ms,
gives up, and sends Tone A. `P34_JD` runs to `T_S_AFTER_JD`:
`P34_ERR_NO_S2`.

Seen with 0.6.0 (`d729110`, 0.6.0-rc1); 0.6.1 replays the recording the same way. Host: `v90m_process_linear`,
mu-law, `V90M_MOD_V21 | V22 | V32BIS | V34 | V90`, `tx_level` 3000,
`echo_delay` -1, everything else as `v90m_cfg_default`.

## Timeline (seconds from the modem's start)

| Time | |
|---|---|
| 3.113 - 5.741 | Phase 2. Round trip 1462 samples; upstream 3200 symbols/s, carrier 3/5 (1920 Hz) |
| 5.856 | first S / S-bar; PP and TRN, receiver error -34 to -37 dB |
| 8.696 | Ja valid (DIL N 197, Lsp 66, Ltp 66). Sd, Sd-bar, TRN1d |
| 9.785 | `P34_JD`; echo delay measured 1452 |
| 11.850 | the caller's S reaches the receiver: 625 ms at -18.0 dBm0, lines at 1920, 3520 (-4 dB) and 320 Hz (-10 dB), as in its first S |
| about 12.5 | Tone A (2400 Hz, with 1800 Hz 7 dB below) |
| 14.033 | `P34_ERR_NO_S2` |

## Why S is not accepted

Printing `p->up.rx.zr / zi` and `p->sres` on every symbol while the S is
there: the symbols alternate between about 1.55 and about 0.45 of
`QRX_REF` in magnitude, both near the same angle, for the whole 625 ms.
The mean of each pair is about 1.0.

`SRES_WAIT` wants `SRES_ON + SRES_SKIP` (14) symbols in a row with
`e > half / 2`, that is |z| above 0.71 of `QRX_REF`, and sets `cnt` to 0
on any symbol below. Every second symbol is below, so `cnt` reaches 5
while the signal builds up and then goes 1, 0, 1, 0 to the end.
`SRES_TIME` has the same kind of test (`e < half / 4`), which the weak
symbol of each pair (0.45, squared 0.2) would also fail.

It depends on the sampling instant. The same recording with the caller's
S moved by whole samples (samples of silence removed or added one second
before it), unmodified library:

| S moved by | |
|---|---|
| -3, -2, -1 | `P34_DIL` at 11.867 s |
| 0 | `P34_ERR_NO_S2` at 14.033 s |
| +1, +2, +3 | `P34_DIL` at 11.867 s |

The caller is silent for about 3 s between Ja and this S, with the
receiver's timing held. Its symbol rate measured through the gateway on a
V.34 call is 3199.837 Hz (-51 ppm against our clock): about half a symbol
of drift in 3 s. That would put the S on the same instant on every call,
which fits "every call fails"; this part is inferred, from one recording.

## The change

`sres_symbol` (`modem/v34train.c`): a symbol counts by itself, as before,
or as a pair with the one before it when the two hold the power of S
between them; `SRES_TIME`'s "it went away again" likewise. `SRES_WAIT`
keeps the energy of the symbol before in `sr`, which it did not use; no
struct changes size. A run of symbols that passed before is counted
exactly as before.

Tests, each seen failing first:

- `test_sres`: S after a silence through a first-order high-pass at
  500 Hz, -50 ppm, the silence stepped over a whole symbol of drift in
  sixteenths. Before: resumed 12 of 16 (not from 0.375 to 0.56 of a
  symbol). After: 16 of 16, the 800 symbols after S-bar without error.
- `test/fixtures/usrc1.raw.xz`, this call. Before: Phase 3/4 state 4
  until 14.033 s. After: state 5 at 11.867 s.

Golden files: `test_sres` (the new line and count); `test_orig_v34`, one
figure (in "retrain begun by the caller" a weak symbol at count 13 no
longer starts the count over; SNR 31.3 to 31.4 dB). The seven other
fixtures' traces are unchanged.

Run on Windows with gcc 15.2 (mingw): the 85 programs of `TESTS` against
their golden files, and the fixtures. **Not run**: `make check` as a
whole (`v34/test/test_bench.c` needs `clock_gettime` there), `check-arm`,
the benches' hashes, the replay proof. Please run those.

**Not tried on a live call.** The recording ends where the caller gave
up, so whether the Courier connects once Jd' and the DIL are sent is not
known yet.

## Two smaller things seen on the way

1. **Tone A in Phase 3.** The comment in `p34d_tick` explains why Tone A
   is not watched before `P34_RI`. In `P34_JD` and `P34_DIL` the caller is
   silent or sending S or SCR; here it sent Tone A for 1.5 s unanswered
   before the time limit. Would following it there (a retrain rather than
   `P34_ERR_NO_S2`) be safe? `tick_v92` already answers a failed start-up
   with a retrain; `tick_v90` ends the call.
2. **`v90m_process_linear` and mu-law negative zero.** In the PCM phase
   the modem sent octet 0x7F 72 times in this call (in the second after
   Sd began). Its linear value is 0, and a G.711 encoder turns 0 into
   0xFF, so a host on the linear interface cannot send it. Probably
   harmless on an analogue far end; `docs/line-interface.md` could say
   that this is the one codeword a mu-law codec does not turn back.

## Recording

`test/fixtures/usrc1.raw.xz` (in this pull request): `v90m-replay` format, companded, mu-law,
14.2 s, made from a packet capture on the answering host. Per 10 ms, 80
octets heard then 80 sent. Sample 0 is the modem's start; the heard
stream is delayed by the receive path's 1091 samples, found by matching
the library's own output to what was sent (0 samples differ over the
whole call when the output of `v90m_process_linear` is encoded with the
host's G.711 table).

```
v90m-replay -r 3000 -d 0 -s 0 usrc1.raw
```

ended with "the modem stopped (1)" at 14.033 s before the change. It reports 1962 samples of
transmit differing from the recording: the host used
`v90m_process_linear` and its own mu-law encoder, which rounds values
between two levels differently outside the PCM phase (1890 samples), and
the 72 samples of 0x7F above.

Not in the recording: anything after the failure. What the Courier does
once Jd' and the DIL are sent is not known.
