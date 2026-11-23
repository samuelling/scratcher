# LiveSkratch

A live-input DVS for the **Headache Sound WOW** (Wow & Flutter machine). The WOW plays a control tone instead of music. LiveSkratch decodes the platter's motion from that tone and uses it to scratch **whatever audio is coming in right now**: a synth, a mic, a drum machine.

It comes in two forms:

- **iPad / iPhone app + AUv3 plugin** (`ios/`): native, lowest latency, full interface channel routing, a one-switch WOW setup screen, and an Audio Unit effect for AUM and other hosts.
- **Web app**, a single self-contained page: **https://linglab.org/scratcher/**. It matches the iPhone app feature for feature: Scratch / Setup / Feel tabs, ? help and a first-time walkthrough, the one-switch WOW setup, input channels picked by number, and the built-in “ahhh / fresh” loop. Use **Chrome or Firefox**: other browsers may pass only 2 channels of audio in and out. Switching the WOW from the browser needs Chrome or Edge.

The two apps are kept aligned: a change to one goes into the other (see `CLAUDE.md`).

## How it works

- **Tape.** Live input is written continuously into a 32-second ring buffer.
- **Needle.** A read head is driven by the decoded platter motion. Hands off at 1× and it rides just behind live, so you hear the input straight through. Pull back and you're scratching the last few seconds. Let go and it snaps back to live, or stays behind if you prefer that.
- **Freeze.** Stops the tape and loops the last N seconds, so you can scratch a captured phrase like a record.
- **Fader.** The WOW's own crossfader becomes LiveSkratch's fader, read one of two ways:
  - **Pilot tone (recommended).** In the WOW's `wow_settings.txt`, set `fader_left = beat` and `fader_right = beat`, and put the pilot in the `beats` folder (or let `wow/scratcher-mode.command` do all of this). The fader then cuts an 18 kHz pilot tone whose level is the fader position, while the control tone is never cut, so the platter stays tracked through every cut. Two wide notches remove the pilot before decoding. A low-pass would rotate the 3rd harmonic's phase and flip direction above about 2×. Calibrate measures the pilot's exact frequency and level.
  - **Tone level.** The fader cuts the control tone itself. It's simpler, but platter motion is invisible while the fader is closed.
- **Virtual fader.** An on-screen cut fader with cut-in, curve and hamster settings, used alone or together with the WOW's fader.

## The control tone

A 1500 Hz sine whose **loudness** follows a lopsided triangle (in log amplitude): it rises over 32 ms of file and falls over 8 ms, repeating 250 times over a 10 s seamless loop.

- **Speed** comes from zero-crossing timing (median of recent half-periods).
- **Direction** comes from the loudness trend. A slow rise or a fast fall means forward; a slow fall or a fast rise means backward. The rate is compared with what's expected at the current speed. The loudness is measured over exactly one carrier cycle, so the carrier's own ripple cancels.
- **Why loudness, not waveform shape:** diagnostic recordings showed that the WOW plays reverse with the fine waveform still running *forward*. File position goes backward smoothly, but every millisecond-scale snippet is forward-shaped. Any shape-based scheme (a phased harmonic, a sawtooth waveform) therefore always reads "forward" on the WOW. Loudness follows file position, so it reverses correctly. The cost is about 10–15 ms of direction latency at turnarounds.

`test/test_wow.js` checks the decoder against both true reverse and WOW-style reverse.

## Live, Record, Loop

LiveSkratch records what comes in onto a tape and your platter moves a needle along it. The needle can go anywhere that has been recorded, but never ahead of now.

- **Live**: after a scratch the needle jumps back to now once the platter is back at normal speed.
- **Record**: a normal record. The needle stays wherever you leave it and plays on from there.
- **Loop**: the tape stops recording and holds still (up to the whole tape), wrapping round at the ends or stopping like a run-out groove.

**Memory** sets how far back you can go: 45 s, 2 min or 5 min (the AUv3 goes up to 2 min). The tape is stored as 16-bit audio, so 5 minutes of stereo is about 58 MB.

## Gear: what to plug in

LiveSkratch needs a USB mixer or audio interface that:

1. **works without a driver** (USB class compliant), so an iPhone or iPad can use it;
2. **sends each input over USB as its own channel**, not only its stereo mix, so the app can keep the control signal apart from the music;
3. **has enough inputs**: 2 or 3 for a WOW (the mono tone plus your source), 4 for timecode (left and right plus a stereo source);
4. **plays LiveSkratch's output back out**, ideally with a way to keep the raw control signal out of your speakers.

### The ideal: Teenage Engineering TX-6

Pocket-sized, battery powered, and it ticks every box. In its *12 out / 2 in* USB mode it sends each of its channels to LiveSkratch separately and takes the scratch back into its main mix.

1. TX-6 system menu → **usb**: choose **12 out / 2 in**, and set **main out source** to **ext**, so the main out plays only what comes back from LiveSkratch.
2. WOW (or timecode deck) into **channel 1**; the source you'll scratch into another channel. Listen on **Main out**, not Cue.
3. On the control channel: fader up and left alone, EQ flat, compressor and filter off, no FX sends. The TX-6 sends to USB *after* the fader, so the fader can't be used to mute it.
4. In LiveSkratch → Setup: WOW on input 1 (timecode: L on input 1, R on input 2); Live L and R on that channel's pair (TX-6 channel 6 is inputs 11 and 12); output 1–2.
5. Calibrate.

Menu names can differ between TX-6 firmware versions.

### Mixers like it

Any mixer with a multitrack USB mode works the same way:

- Switch USB to **multitrack**, so every channel goes out separately.
- Send the **USB return** to the main outs.
- Keep the raw control signal out of what you hear. If the mixer sends to USB **before** the fader, pull the control channel's fader down. If it sends **after** the fader (like the TX-6), use a main mode that plays only the USB return, or turn direct monitoring off.
- Keep EQ, compression and effects off the control channel. Then pick the channels in Setup.

### Examples that meet the requirements

These are examples of the kind of gear that fits, not endorsements, and not all of them have been tested with LiveSkratch. Check the maker's iOS support before you buy.

- Mixers with multitrack USB: Behringer Flow 8; Zoom LiveTrak L6 (some iPad users report only its master inputs showing up).
- Recorders and interfaces: Zoom H6essential (6 inputs), MOTU M4 (4 inputs).
- DJ mixers with class-compliant USB audio, such as the Allen & Heath Xone:96. Battle mixers built for Serato, such as the Pioneer DJM-S5, are unconfirmed on iPhone.

**Stereo-only mixers** that send just their mix over USB (the Roland GO:MIXER PRO and PRO-X, for example) can't keep the control signal apart from the music. Workaround with a WOW: pan the WOW hard left and a mono source hard right, then pick input 1 for the WOW and input 2 for both Live L and R.

**In a browser**, use Chrome or Firefox: other browsers may pass only 2 channels in and out.

## Setup

1. WOW out → interface input. Live source → another input.
2. Open the page and press **Start audio**. On the **Setup** tab pick the control input and the channel it's on, what you scratch (its left and right inputs), and the output. To switch the WOW itself, plug it in over USB, press **Choose the WOW drive** and flip **Normal / LiveSkratch**, then eject and restart the WOW.
3. Motor on, hands off, fader open → **Calibrate**.
4. Scratch. <kbd>Space</kbd> jumps to live, <kbd>F</kbd> freezes, hold <kbd>C</kbd> for a keyboard cut.

To try it without the WOW, set the control source to **Virtual platter** and drag the record with the mouse. The built-in test loop gives you something to scratch.

## Dev

```
src/dsp.js      decoder + scratch engine (shared by the worklet and the tests)
src/worklet.js  AudioWorklet processor
src/app.html    UI; build.py inlines the DSP into index.html
test/           test_wow.js: accuracy under true and WOW-style reverse; test_pilot.js: tracking through pilot-fader cuts; test_engine.js
```

Run `python3 build.py` after editing anything in `src/`.


## Control signals: WOW tone or stereo timecode

- **WOW tone (mono).** The WOW has a mono output, so it plays the LiveSkratch tone described above.
- **Stereo timecode.** Serato or Traktor control vinyl and CDs, or `wow/00-liveskratch-timecode.wav` (a copyright-free 1 kHz quadrature pair in Serato's phase: the right channel leads the left when playing forward, as on Serato and Traktor records) played by an SC-1000 / SC-500. A timecode deck has no WOW fader, so in timecode mode only the on-screen fader cuts. The two channels carry the same carrier a quarter-cycle apart, so the pair traces a circle whose angle is the position within a cycle and whose turning direction is the platter's. LiveSkratch scratches live audio rather than a track, so it needs only speed and direction: a relative-mode decoder that ignores the position bits works with any such timecode. Calibrate finds the exact carrier (1 or 2 kHz), the left/right balance and the direction. Simulated scratches track within 0.3–3.4 ms, against 15–38 ms for the mono WOW tone.

The on-screen fader is a tap pad by default, like a PT01's cut switch (touch cuts, or touch opens); the slider is still available.

## iPad / iPhone (ios/)

Open `ios/LiveSkratch.xcodeproj` in Xcode 16+, pick your iPad, and press Run. Two targets:

- **LiveSkratch**, the app. Pick the WOW's input channel, the live audio channel(s) and the output pair under *Audio routing*. With nothing plugged in it scratches a built-in loop (synthesised "ahhh" and "fresh" stabs over a beat); *Load loop…* swaps in any audio file and keeps it as the default.
- **LiveSkratchAU**, an AUv3 effect ("Sam Ling: LiveSkratch"). In AUM, make a stereo channel from two interface inputs, the WOW on the left and the live audio on the right, and insert LiveSkratch. Crossfader, cut, freeze, jump-to-live, gain and source are host parameters you can map to MIDI.

**WOW setup without Terminal.** Plug the WOW into the iPad, open *WOW setup*, choose the WOW's drive in Files once, then flip *Normal / LiveSkratch*. The switch edits `wow_settings.txt` (keeping its alignment and line endings), swaps the tone files in, and parks your own beats and samples in a `LiveSkratch/` folder on the WOW, with a backup of your settings, so switching back works from any device. It also recognises a WOW switched by `wow/scratcher-mode.command`.

Layout: `ios/Core` is the DSP (a line-for-line port of `src/dsp.js` and `src/worklet.js`, no allocation on the audio thread), `ios/Shared` the settings, UI and loop code both targets use, `ios/App` the RemoteIO host and WOW setup, `ios/AUv3` the plugin. The DSP has command-line tests that also run on Linux:

```
swiftc -O ios/Core/*.swift ios/Tests/main.swift -o /tmp/liveskratch-tests && /tmp/liveskratch-tests
```
