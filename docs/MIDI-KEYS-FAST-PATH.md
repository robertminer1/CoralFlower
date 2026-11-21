# FM-1 Keys: low-latency MIDI input

**Status:** Hardware-tested optimization; release-candidate source. The previous diagnostic build was confirmed on an M-VAVE FM-1 using a Teenage Engineering EP-133 over TRS MIDI. This cleaned source still needs a fresh build and final smoke test before tagging or merging.

## Performance observed on hardware

Measurements are from completed UART MIDI Note On packet to audio-engine application or **estimated scheduled DMA output** (not analog onset). Isolated-note test, using the same synth and EP-133 in FM-1 Keys mode.

| Measurement | Before | Optimized | Change |
| --- | ---: | ---: | ---: |
| Note → chord engine | 10.939 ms | 1.124 ms | 89.7% faster |
| Note → estimated output | 13.838 ms | 3.775 ms | 72.7% lower |
| Maximum estimated output | 16.327 ms | 5.330 ms | 67.4% lower |

The optimized diagnostic run tracked 57 received MIDI notes, 57 fast-path note-ons, and 57 output-threshold matches. These are *not* guaranteed end-to-end MIDI-to-analog-output measurements; the analog DAC and source device's own scheduling were not measured. The performer reported the firmware felt **“way tighter.”**

## Implementation

With **GLO → MIDI → NOTES IN → FM-1 keys** and the **HOME** screen open, incoming MIDI notes use the existing chord engine directly in the audio ISR, instead of waiting for the main-loop/UI queue to send the key back to the audio ISR. UI-held key lights and bookkeeping are still updated asynchronously, tagged to suppress duplicate note-on/note-off dispatch. Incoming velocity retains the existing fixed **GLO → VELOCITY** semantics.

The audio-ISR-only fast note-to-key map tracks repeated/octave-folded keys, and key-off handling considers physical keys and slow-path MIDI keys. Mode/channel changes release fast-held keys. EDIT, sequencer and other screens retain the original UI-mediated MIDI behavior. Synth engine, audio buffer, DSP, ARP, glide, FX and chord rules are unchanged.

The source remains explicitly switchable: `#define CF_MIDI_KEYS_FAST 1` at `firmware/src/coralflower.c`. Changing this to `0` disables the optimized route.

The temporary `CF_MIDI_LAT_DIAG` instrumentation and the `lat` console command have been **removed** from the clean release candidate. Earlier timings remain recorded above and in `docs/MIDI-LAT-DIAGNOSTICS.md` (archived research instructions; requires the older diagnostic commit).

## Validation performed

The user tested rapid sequences, simultaneous physical and MIDI keys, and navigation away from HOME, reporting that **all worked correctly** with the instrumented fast build.

## Release-candidate build and final checks

On Ubuntu x86-64:

```sh
git fetch origin
git switch feature/midi-low-latency
git pull --ff-only origin feature/midi-low-latency
./build.sh --release 0.7.1-midi-rc1
```

Expected artifact: `build/release-0.7.1-midi-rc1/coralflower-0.7.1-midi-rc1.fwsc`. Verify package checksums:

```sh
cd build/release-0.7.1-midi-rc1
sha256sum -c SHA256SUMS
```

Back up settings, keep the previous known-good firmware for recovery, and flash the RC with the CoralFlower browser installer: https://ficusficusov.github.io/CoralFlower/.

Smoke checks after installing the **clean** build:

1. HOME + FM-1 Keys: TRS sequence responds promptly; fixed velocity and chord forms unchanged.
2. Rapid notes/chords, octaves mapping to the same FM-1 key: no stuck or missed notes.
3. MIDI plus physical key overlap, in each release order: no premature stops or double triggers.
4. Leaving HOME and returning during MIDI activity: no stuck notes or unexpected edits.
5. Other input modes (Direct, Roots), ARP and sequencer: original behavior preserved.
6. Verify firmware boots cleanly and stored settings/sounds survive reboot.

**Do not create a release tag or merge until a clean build and hardware smoke test pass.**

## Rollback

Flash the previous known-good firmware. For a source-level rollback, set `CF_MIDI_KEYS_FAST` to `0` and rebuild. The TRS UART polling change in `main.c` is a separate change from the fast Keys path.
