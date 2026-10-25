# Modulation testing

## Source evidence

Evidence source: repeat98/octamad commit `b8deefc88b2c3e5f3c6158e364eb741df1924e1d`. Read [the original record](README.md).

The source records 29 reference gates and 13-setting identity comparisons. Image 88 overran with four Modulation instances beside reverb; the following cycle optimization has not been retested on hardware.

## Commands and results

Declared native gates:

`tools/verify/verify_modulation.py`

This SDK import did not rerun these gates. CPU-heavy checks remain paused. Historical results are source records, not new qualification of the SDK or a combined configuration. Record exact commands, tested source commit, workload, modes, track count, duration and results here whenever the module changes.

## Hardware and limitations

Hardware status: historical. Priced words per sample: PHSR 298, COMB 329, LINE 354. These are estimates, not a percentage of DSP capacity.

Do not claim a passed hardware test without a model, image version, conditions and actual result. Packaging parity is separate from audio quality and hardware safety.

## 0.1.1-experimental licence update

Copyright notices, full component terms and SPDX metadata were corrected.
No DSP implementation changed and no firmware, emulator, audio or hardware tests
were rerun. Existing evidence remains attached to the source revision above;
this documentation version does not create new firmware qualification.

## 0.1.2-experimental: octabam 8250be0's DSP source

Source: `modulation.asm` and `modulation_ref.py` from
[sambanks/octabam](https://github.com/sambanks/octabam) at
`8250be0c23ca0a0a953cb7e5721e3bdb98dadfe5`, unchanged
(`sdk/imports/modulation-8250be0.json`). Modwerk's `manifest.py` is kept:
effect id 0x0b, FX1 only, stock CHORUS kept; its parameters, counts,
defaults and claims equal upstream's. Upstream's two changes since 0.1.1:
the delay lines run as modulo-1024 rings (`m5 = $3ff`, octabam 7e6afd08,
5 Oct 2026) and the one `rep #$18` became `do #$18,>norep_md_01`
(86a9a611, 12 Oct 2026). `npm run module:doctor -- modulation`: no `rep`,
no label prefixing another.

Everything below was run on a computer (macOS, Apple silicon) on 24 October
2026 unless it says otherwise.

### Bit identity with 0.1.1 (here)

In a scratch copy of octabam's own tree at `8250be0` (outside Git; the
modules whose git submodules the copy lacked were left out of it, none of
them Modulation or its harness), `dsp_host` and `dsp_asm` built by
`make setup` from the same `tools/patches/dsp56300.patch` and
`dsp_host.cpp` as Modwerk's SDK:

- `tools/verify/verify_ident.py modulation ref` on 0.1.1's
  `modulation.asm` with its single `rep #$18` rewritten as the same
  `do #$18,>norep_md_01` upstream uses (octabam's build now refuses `rep`;
  the effects guide records the rewrite as bit-identical), then
  `verify_ident.py modulation check` on 0.1.2's source:
  **IDENTICAL: 13/13 settings bit-identical** (defaults, every mode hot and
  cool, all zeros, all max). The 13 reference renders are 12 distinct
  outputs, so the matrix exercised the wet paths.
- `tools/verify/verify_modulation.py` on both: **ALL PASS**, 26 checks each
  (FX1 MIX 0 exact passthrough, FX2 exact dry and guard clean, every mode
  against the float reference at two settings, the LOFI settings, JUNO's
  sweep window and 0.5 Hz rate, FLNG's through-zero null, PHSR's unity
  magnitude, COMB's tuning at DLY 32, 64 and 96, every knob at both ends).

Not here: the same gates in Modwerk's trimmed SDK. Its `tools/remix/audition.py`
builds a scratch remix with octabam's SEND module, which the SDK does not
carry; with SEND copied in, the FX1 instance rendered its dry path for 0.1.1
and 0.1.2 alike (21 of 26 checks failing on the published 0.1.1 too), so
that harness, not the module, is the difference. Not investigated further.

### Cycles (here)

`REMIX=stations-fx2 python3 -B tools/build/cycle_count.py` in the same tree,
on each source. A static count of the per-sample loop: one-word
instructions as one cycle, two-word as two; memory-contention stalls not
modelled, so the chip spends more.

| loop | 0.1.1 (rep as do) | 0.1.2 |
|---|---|---|
| LINE (JUNO, DIM, FLNG) | 354 | 330 |
| COMB | 329 | 291 |
| PHSR | 298 | 298 |
| `mo_tap` / `mo_itap` per call | 27 / 19 words | 21 / 13 words |

These equal octabam's own figures for the change (its README, "5 Oct 2026,
the cycles pass"). The worst loop, 330, is the USB package's declared
`cycles` (`--cycles-kind modeled`).

### Packages (here)

- `bash scripts/build-modules-isolated.sh` in `modwerk-source-tools`
  (`sha256:f4bded99…`), then `npm run modules:import -- … --development`:
  only Modulation's rows changed. Its relocatable DSP package is 1,559 words
  (was 1,575); every other module's package is unchanged.
- USB runtime package: `src/engine/assets/octatrack-module-packages.json`
  gains Modulation (ABI 6, `slots fx1`, `cycles 330 modeled`, `state 112`,
  `buffer 2048`); `public/octatrack-modules/modulation/0.1.2-experimental.mwrm`,
  7,386 bytes, SHA-256 `21bdc13b58a2827f4e2da39113154eb0ca459306e386979ba200c6be5ef946e9`,
  no stock-code sites, copies or fills. `npm run octatrack:module-packages --
  --check --stock <the contributor's 1.40C>`: Modulation `identical`, most
  non-zero bytes in a run shared with the stock OS **0** (fails at 12); every
  other package checked before the run stopped is identical. The run stops
  at a standalone recipe ("does not name the exact source/image inputs") on
  a clean checkout of `main` too, so that stop predates this change.
- State: the r7 block is `$00..$6f` (the header's map: persistent and
  per-block slots to `$47`, the constant stream `$48..$6f`), 112 words.
  Buffer: the two 1,024-word lines inside the stock FX1 block of 3,072.
- Ring alignment over USB: the base gives an FX1 module the stock FX1 block
  (`FX1_BUFFER 3072u` in `sdk/machines/octatrack/elekloader/dsp.c`) and the
  module reads its base from the stock allocator, so the lines start at
  0x1000, 0x1c00, 0x2800 or 0x3400 (+0x400 for R): 1,024-aligned, which the
  modulo rings need. From reading the source; the hardware run below is the
  measurement.
- The frontend MCP's `site_selection` with `usb: true` prepares the package
  and an exact Undo package (`56b52b76…`) without device writes.

### Screens (here)

`scripts/capture-module-ui.py --mki` with `media/capture.json`'s plan on a
private native image of MODULATION alone with stock FX2 kept
(`export-composition-proofs.py --select modulation:true`, image
`0fb6585f…`, never committed), Modwerk's prepared `ot_emu` (`bb17b9ee…`,
`scripts/prepare-octatrack-emulator.py --build`, six gates passed). Five
PNGs; the location capture repeated byte for byte across two runs.

### Over the emulated USB (here)

The beta base (identity `56c44acc…`, the site's `CURRENT_BASE`, rebuilt
from the contributor's 1.40C by `npm run octatrack:base -- --stock … --beta
--out …`, MAIN `bdae012d…`, never committed) on Modwerk's adapted emulator
(`scripts/octatrack-emulator/build.py`, `ot_emu` `d940339e…`; its
`test-idle-range` regression passes; the boot gate creates all ten tasks),
the contributor's own stock-effects test project staged with
`tools/emu/ot_emu/stage_card.py`, `--frame --dsp`:

- `scripts/verify-octatrack-dsp-loader.mjs` `pick:modulation` with the
  0.1.2 package: **module installed** over the emulated USB, **picked on FX1
  of T1 and of T5** (one track per DSP core, FX1 row 11). Live effect ids
  FX1 T1..T8 `11 0 0 0 11 0 0 0`; transport errors 0, refused picks 0,
  completed picks 2; resident words per core `1559, 1559`, the package's
  size. The script's `check` asserts the FX2 slots (`ids[8]`, `ids[12]`),
  written for FX2 effects; with those two indices read as FX1 (`ids[0]`,
  `ids[4]`) in a scratch copy, the rest of its check passes unchanged:
  **each core holds Modulation's code word for word inside its arena and
  dispatches to it**.
- Not run: `remove`, `missing`, `beside` and the other scenarios. They
  pick and poke FX2 slots, so they do not exercise an FX1-only effect as
  written. Removal, Undo and the missing-module fallback are left to the
  hardware run.
- Emulator evidence only: executed instructions, no audio, hardware timing,
  cache behaviour or real USB.

Sound quality (aliasing, clipping, DC, idle behaviour): not tested; `npm run fx:audit` was not run for 0.1.2. 0.1.2's output is bit-identical to 0.1.1's, so it neither improves nor worsens them.

Not run on a computer: `fx:audit`, `verify_knob_clicks.py`, a stress
render or `perf:audit` (Modulation predates the performance record, listed
before 7 October 2026), and `module:verify`, which skips paused modules.

### Card image (here)

The private native image above (MODULATION alone, stock FX2 kept, MAIN
`0fb6585f9bb6dc1ae4075b5a37e42565c7a46e5516d697b5007816864b5c326c`) packed
into an update file with the same steps `export-composition-proofs.py` runs
for its proofs (`tools/build/bin_decode.py` on the contributor's 1.40C `.bin`,
`scripts/native-container-oracle.c` with octabam's `elektron-firmware-tool`
sources, `tools/build/make_bin.py`), version `MW_MOD012`, `.bin`
`2539846f87b01703cce4a6340b203e40f2a1603d0b66b1d26be7124bf519457b`, never
committed. Its section 3 decompresses to the MAIN image byte for byte; the
bootloader copy the OS re-flashes from (0x400de1e0-0x400e21e0) equals stock;
9,116 bytes of the MAIN image differ from stock. On Modwerk's adapted
emulator with `--frame --dsp` it boots (all ten tasks) and loads the
contributor's test project (LOAD PROJECT handled in 14.2 s emulated).

### Hardware: reported (24 October 2026)

Octatrack **MKI**, OS 1.40C, tester **nathangroth** (the contributor), the
card image above flashed with OS UPGRADE. Their report, in their words where
quoted:

- "testing for a few minutes and most things are working"; Modulation
  selected from FX1.
- "I put an instance on all 8 tracks and did not hear any audible crackles or
  dropouts."
- "The settings also persisted after a reboot."
- Scene morphing: a first try appeared to jump MIX from 100% to 0 at the
  crossfader's middle; on a retest it "seems to be working fine now" for
  Modulation and for stock CHORUS with the same scene setup, and the first
  observation could not be reproduced ("perhaps I just had a strange
  combination of scene locks").
- Mode names: after changing T1 to COMB, selecting T2 (in JUNO) on the same
  FX1 page kept COMB's knob names until page 2 was opened and closed. This is
  the documented limit of octabam's `ModeView` names (one descriptor shared
  by every track running the effect; renamed when MODE is drawn;
  `sdk/octabam/docs/remixer/MODULES.md`), unchanged by this release.

Not reported or not tested on hardware: each mode by ear, knob extremes,
parameter locks, LFOs, Part changes, a measured audio quality run, an MKII.

### Over USB on hardware: did not load

The same MKI on the beta base (`56c44acc…`), connected to macOS through a USB
2.0 hub: IDENTIFY, DIAG, status and the frontend MCP's `device_review` worked;
every upload stopped at its first multi-packet frame ("chunk was not
confirmed … The device kept refusing the submission"; also with 1,000- and
200-byte chunks, and after restarting the Mac and power-cycling the unit).
DIAG from a fresh power-up `primes 1, completions 1`; after one attempt
`primes 18, completions 9`, `lastReceived 48`. Nothing was installed. The
contributor also saw USB DISK MODE fail to eject cleanly on the base. A
direct connection was not possible, so hub, MKI and base are not told apart;
this goes to Modwerk as a separate report, not as a Modulation defect.
