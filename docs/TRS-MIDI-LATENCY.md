# EP-133 TRS MIDI latency experiment

This branch contains a narrow first-stage improvement to TRS MIDI receive timing.

## Change

In `fm1_timer5_irq`, service the UART RX DMA ring when TIMER5 interrupts an
ongoing audio half-buffer render. Previously the nested TIMER5 handler skipped
both UART and USB work, deferring TRS MIDI reception until after the audio render.
USB polling continues to be deferred during a render; only the UART RX ring is
read in the nested path. The existing `midi_in_q` event parsing, FM-1 Keys
mapping, UI ownership/reference counts and audio buffering are unchanged.

This is **not** a full FM-1 Keys fast path. In `NOTES IN -> FM-1 keys`, MIDI still
passes through `cf_xq_put`, `cf_xq_drain` and `CFE_KEY` before `cfp_key`.
The extra UI-loop handoff is a possible source of jitter and remains a candidate
for a separately tested change.

## Measurement procedure

Equipment: EP-133 KO II, TRS MIDI, FM-1 running CoralFlower; audio recorded to
a DAW or scope alongside the EP-133's direct percussive reference signal.

1. On both the base firmware and this branch, select the same short-attack
   patch. Turn off arpeggiator, strum, glide and time-based effects.
2. Set `GLO > MIDI > NOTES IN > FM-1 keys`, and configure matching MIDI channels.
3. Program several hundred identical, widely spaced EP-133 notes at a fixed
   tempo, recording the EP-133 click/drum transient and FM-1 audio output
   simultaneously.
4. Compare onset *distributions* (median, 95th percentile and range) between
   base and branch. Re-run with `NOTES IN > Direct` to separate key-mode/UI delay
   from UART and audio-buffer delay.
5. Test note release, fast retrigger, chord-button plus root combinations, two
   MIDI notes folding to the same FM-1 key, MIDI+physical key overlap and MIDI
   clock sync. Verify that no notes stick or drop.

## Expected result and limitations

Draining UART during an in-progress audio render should reduce a possible
receive-path wait, but its net effect depends on exactly when DMA packets arrive
and when the subsequent audio control block handles them. The hardware's
128-frame audio half buffer remains approximately 2.9 ms at 44.1 kHz, and
the FM-1 Keys UI handoff remains. No particular latency improvement is promised
without a hardware comparison.

## Validation status

Source-level change only. No firmware build, host tests or hardware recordings
have been run as part of this PR. Review real-time interrupt load and run the
existing host tests and hardware regression checks before merging.
