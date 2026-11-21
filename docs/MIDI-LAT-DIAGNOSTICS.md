# Experimental TRS MIDI-to-output timing: Direct and FM-1 Keys

This branch adds an experimental read-only USB CDC console command, `lat`, intended to isolate **Direct-mode and FM-1 Keys-mode** TRS MIDI Note On delays. It does not change DSP, buffering, envelope, or output behavior.

## Build and flash

```sh
git fetch origin
git switch feature/midi-low-latency
git pull --ff-only origin feature/midi-low-latency
./build.sh --release 0.7.1-latency-keysdiag
```

Flash `build/release-0.7.1-latency-keysdiag/coralflower-0.7.1-latency-keysdiag.fwsc` using the normal installer (back up settings first).

## Record a clean measurement

1. Connect EP-133 TRS MIDI OUT to FM-1 MIDI IN; also connect the FM-1 USB data cable to your Ubuntu computer for the serial console. The EP-133 still supplies MIDI over TRS.
2. Set **GLO > MIDI > NOTES IN > FM-1 keys** for the problem reproduction (then repeat in **Direct** for comparison); use an audible, fast-attack preset. Set RETRIG Notes, GLIDE Off, ARP Off.
3. Disable the FM-1 reverb/delay effects and use **Options > Output > Classic**. Stop any internal sequences, arps and sustained notes. Let tails die away.
4. Send one short isolated Note On from EP-133. Pause between notes. Watch CPU load if available.
5. Find the serial device (typically `/dev/ttyACM0`): `ls /dev/ttyACM*`. Open it using `python3 -m pip install pyserial` and `python3 -m serial.tools.miniterm /dev/ttyACM0 115200`. You may need permission to open the port.
6. Enter `lat` at the prompt. Repeat after several notes and copy the output.

## Readout

- `received_notes`: Direct-capable Note On packets captured over TRS (eligible forwarding counted).
- `output_matches`: pending note events that found an above-threshold output sample.
- `rx_to_ui_us`: Keys mode only; microseconds from UART parser packet completion to the UI handling the simulated note (last accepted, non-duplicate virtual key).\n- `rx_to_key_apply_us`: Keys mode only; microseconds to the audio ISR applying the UI-posted physical-key event.\n- `keys_stage`: 0 idle/completed, 1 waiting for UI, 2 waiting for audio event, 3 waiting for output.\n- `rx_to_dispatch_us`: microseconds from UART parser packet completion to Direct-mode note dispatch **or** Keys-mode audio key application, including audio ISR scheduling.
- `rx_to_dispatch_max_us`: largest observed across this power-on session.
- `rx_to_dma_us`: estimated microseconds from UART packet completion to *scheduled DAC playback* of the first output-buffer sample exceeding roughly -42 dBFS, latest match.
- `rx_to_dma_max_us`: largest observed.
- `pending`: a note still waiting for an above-threshold output match; expires after about 200 ms.

## Important limitations

This is **not** a measurement from the EP-133 key press or the start of the DIN MIDI serial message: the receive timestamp is taken when its final data byte has been parsed. Thus it omits TRS transmission and EP-133 scheduling.

The output match tests the **whole mix**, not the specific note. Existing sustained audio, other notes, delays, and reverb tails can produce false-early matches. Very quiet presets may never cross the threshold. The output timestamp uses the interrupt entry time plus the expected DMA half-buffer offset, so it is an approximation of scheduled playback, not a scope measurement of the analog jack.

Concurrent notes can overwrite the single pending marker. This deliberately favors an isolated-note diagnostic over a complicated tracing subsystem. The measurements are not performance benchmarks and are not expected to exactly equal subjective latency.

If `rx_to_ui_us` is high in Keys mode, the UI handoff is likely responsible. If `rx_to_key_apply_us` grows further, inspect the return event queue and audio scheduling. If Direct-mode `rx_to_dispatch_us` is high, inspect MIDI processing/scheduling. If dispatch is fast but `rx_to_dma_us` is high, investigate the synth envelope, effects/master chain and DMA timing. For hardware-accurate analog timing, capture the audio jack and a trigger reference together with an external interface or oscilloscope.

Remove or disable the diagnostic macro for normal releases.

## Compare Modes

Restart the FM-1 between the two test runs if you want reset max counters. In each mode, send isolated notes, and read `lat` after each. In FM-1 Keys mode, the `rx_to_ui_us` and `rx_to_key_apply_us` fields are the main new data. In Direct mode, use `rx_to_dispatch_us`. The other mode's prior readings persist until reboot; do not compare across modes without noting that.
