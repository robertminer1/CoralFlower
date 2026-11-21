# Experimental fast MIDI Keys path

**Status: unbuilt, untested on hardware. Do not merge.** This is a test build, not a production release. The change is compile-gated with `CF_MIDI_KEYS_FAST` at the top of `firmware/src/coralflower.c`. Set it to `0` to restore the previous behavior. Existing USB CDC `lat` diagnostics remain enabled.

## Why

The pre-change diagnostic (62 isolated EP-133 notes, NOTES IN = FM-1 keys) reported:

- MIDI parser completion -> UI simulated key: latest 10.162 ms, max 12.628 ms
- parser completion -> chord key application: latest 10.939 ms, max 13.427 ms
- parser completion -> estimated scheduled DMA output: latest 13.838 ms, max 16.327 ms

Direct-mode dispatch was much faster. Main-loop/UI rendering cadence was likely causing timing variation.

## What changed

With NOTES IN = FM-1 keys **and the HOME screen open**, the audio ISR maps the incoming MIDI pitch to the FM-1's 27-key layout and calls the existing chord key engine directly. MIDI-generated key events are still mirrored through the audio-to-UI ring, but are marked as already played to avoid duplicate audio dispatch.

The audio ISR maintains a separate MIDI note -> key index map and per-key reference counts. The UI retains its own displayed virtual-key state; the ISR owns only fast audio-held keys. Physical-key state is published by the UI so overlapping presses do not retrigger; releases check the live physical key scan. Slow-held MIDI keys are also published so a fast release does not silence them.

This deliberately **does not** activate on EDIT, sequencer or other screens. Other modes and screens use the unchanged queue-to-UI behavior. Retrigger semantics, chord construction, synth engines, effects and DMA block size are unchanged.

## Build

On Ubuntu x86-64, after pulling this branch:

```sh
git fetch origin
git switch feature/midi-low-latency
git pull --ff-only origin feature/midi-low-latency
./build.sh --release 0.7.1-keysfast1
```

Expected: `build/release-0.7.1-keysfast1/coralflower-0.7.1-keysfast1.fwsc`.

Back up settings and use the CoralFlower browser installer, choosing the locally built .fwsc. Keep a known-good previous firmware package for rollback.

## Compare

1. Keep EP-133 TRS MIDI connected; use FM-1 USB for the CDC console.
2. Set **GLO > MIDI > NOTES IN > FM-1 keys** and **return to HOME**. No ARP or glide, attack 1 ms, effects off, Classic output.
3. Connect on Mac: `python3 -m serial.tools.miniterm /dev/cu.usbmodem1103 115200` (substitute enumerated port).
4. Trigger isolated MIDI notes, then enter `lat`.
5. Verify `fast_note_ons` increases. `fast_enabled 1` when on HOME; it should be `0` on other screens.
6. Compare `rx_to_key_apply_us` and `rx_to_dma_us` against the pre-change numbers, after a fresh reboot. In fast mode `rx_to_ui_us` is set to 0 because the UI is intentionally bypassed **for sound dispatch**; the UI still catches up for display.

Then test: different simultaneous MIDI notes mapping to the same physical key; holding MIDI and physical key simultaneously; releasing in each order; chord-button plus root sequences; rapidly repeated short notes; changing NOTEs IN or channel while a note is held; changing from HOME to EDIT with keys held; ARP and sequencer usage. Immediately revert on stuck notes, audible clicks, resets, or display divergence.

## Limitations

This fast path has not yet been compiled or tested. It may still have concurrency or state-synchronization bugs. Physical key/MIDI overlap and transitions between different screens deserve particular attention.

The `lat` command is still a rough **whole-mix** threshold detector, not a precise waveform onset measurement, and does not measure EP-133 scheduling or the initial MIDI serial bytes.

## Rollback

Reflash the previous working diagnostic firmware `0.7.1-latency-keysdiag` or the stock official CoralFlower package. To disable the fast patch at source level, change `#define CF_MIDI_KEYS_FAST 1` to `0` in `firmware/src/coralflower.c` and rebuild.
