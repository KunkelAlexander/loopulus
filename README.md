# Loopulus

Vibecoding project for lack of good looper apps on Android that don't want my data. 
A browser-based guitar looper pedal available [here](kunkelalexander.github.io/loopulus/). 

## Usage

### Playing a loop

1. Open the page on your phone, tap **POWER ON**, and grant microphone access.
2. Set **BPM** and **BARS** with the steppers (hold to repeat).
3. Optionally toggle **METRONOME · IN LOOP** if you want clicks to keep playing after the count-in.
4. Tap the big **START** footswitch:
   - One bar of count-in clicks
   - Recording begins automatically on the next downbeat (red ring)
   - At the end of N bars, playback starts automatically (green ring)
5. Tap the main switch again to **arm an overdub** — it engages on the next downbeat (blue ring). Tap again to end the overdub. Layer count increments.
6. **UNDO** removes the last overdub; **REDO** brings it back.
7. **STOP** pauses the loop without erasing it; **CLEAR** wipes everything.

### Keyboard shortcuts

Useful if you pair a Bluetooth foot pedal that sends keystrokes:

| Key   | Action       |
|-------|--------------|
| Space | Main switch  |
| Z     | Undo         |
| Y     | Redo         |
| S     | Stop         |
| C     | Clear        |

## Design choices

### Single-loop with unlimited overdubs

The most beginner-friendly model — one piece of state ("the loop"), with new layers stacked on top. Same mental model as a Ditto or RC-1. Multi-track and verse/chorus modes add UX surface area that gets in the way when you're holding a guitar.

### Quantized to bars, not free-length

You set BPM and bar count before recording. The loop length is computed as `bars × 4 × (60 / BPM) × sampleRate` samples, locked from the first beat. This means:

- You never need to time a button press precisely. The count-in handles the entry.
- Overdubs always align to the downbeat regardless of when you tap.
- Latency between tap and audio doesn't drift the loop — recording is scheduled against the audio clock, not the tap.

The tradeoff is that you commit to a tempo before you play. For a beginner-friendly v1, this is the right tradeoff.

### Count-in before recording

One bar of metronome clicks before record arm. Removes the need for the player to anticipate the downbeat — you just listen to the click and play on "1".

### AudioWorklet for audio processing

The looper's record/playback/overdub logic runs in an `AudioWorkletProcessor` on the dedicated audio thread, not the main JS thread. The older `ScriptProcessorNode` would have produced audible glitches whenever the main thread is busy (UI updates, garbage collection, scrolling). The worklet keeps a circular buffer the length of the loop and does per-sample mixing: in record mode it writes input; in playback it reads; in overdub it reads existing audio while adding new input to the buffer.

### No dry signal monitoring

The headphone amp already lets you hear your guitar directly. If Loopulus also routed mic-in to the output, you'd hear yourself twice with a small delay between the two paths — a comb-filter mess. So the looper only outputs the recorded loop. Your live playing is monitored by the amp; the recorded layers come back from the phone.

### Mic input with effects disabled

`getUserMedia` defaults to `echoCancellation`, `noiseSuppression`, and `autoGainControl` — all aimed at voice calls and all destructive to guitar tone. All three are explicitly disabled.

### Sample-accurate metronome scheduling

The metronome schedules clicks against `AudioContext.currentTime`, not `setInterval`. `setInterval` drifts (it isn't aligned to audio frames); the Web Audio clock is sample-accurate.

### Undo via buffer snapshots

Before each overdub arms, the current loop buffer is copied off the audio thread and pushed onto an undo stack. Undo pops and sends the buffer back to the worklet. Memory cost is ~1.5 MB per layer for a 4-bar loop at 48 kHz — fine for dozens of layers.

### No external dependencies

A single HTML file with inline CSS and JS. Two fonts pulled from Google Fonts. Nothing to build, nothing to bundle, nothing to break.

## Known limitations

- **Latency varies by device.** Browser audio adds buffering on top of the phone's interface latency. If recorded layers drift relative to your live playing, you'd need a calibration step (planned for v1.1).
- **Old layers can get buried** as you stack overdubs, since each new layer adds energy to the same buffer with no per-layer mixing. A decay/feedback control would help (planned for v1.1).
- **Tempo is fixed once recording starts.** Changing BPM mid-loop would require resampling.
- **Background tabs may suspend audio** on mobile browsers. Keep the tab in the foreground and the screen awake.

## License

MIT