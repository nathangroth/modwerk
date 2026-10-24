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
