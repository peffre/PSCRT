# PAULSTRETCH · Deep Time Audio

A single-file, browser-based port of **Paulstretch** (Paul's Extreme Sound Stretch) with real-time scrubbable playback, the full spectral effects chain from the original software, a 3D spectral-terrain visualizer, and WAV export — presented as a vintage amber CRT terminal.

Drop in any audio file and smear it across minutes or hours. Paulstretch is the algorithm behind the "Justin Bieber 800% slower" phenomenon and countless ambient/drone works: unlike ordinary time-stretchers, it makes no attempt to preserve transients, instead freezing the *spectrum* of a sound into an evolving wash.

Everything runs locally in the browser. No server, no upload, no build step — the entire application is one HTML file.

---

## Quick start

1. Open `paulstretch-crt.html` in a modern browser (Chrome, Edge, Firefox, or Safari — desktop or mobile).
2. Drop an audio file anywhere on the page, tap **Choose audio file**, or tap **Demo audio** for a synthesized bell cascade.
3. Press **▶ PLAY**.

Accepted input: anything the browser can decode — WAV, MP3, M4A/AAC, FLAC, OGG/Opus, AIFF, CAF, and audio tracks of MP4/WebM. 32-bit-float and RF64 WAVs fail in some browsers; re-export as 16/24-bit PCM if a file won't decode.

> **iOS notes.** Audio starts only after a tap (the play button counts). If you hear nothing, check the silent-mode switch. If a file appears greyed out in the picker, long-press **Choose audio file** to open an unfiltered file dialog.

---

## The interface

### Visual field (upper screen)

The terrain is a spectrogram of the loaded sound rendered as a 3D landscape: time runs left → right, frequency front → back (log-spaced, ~40 Hz to Nyquist), height and brightness are energy.

| Action | Result |
|---|---|
| Drag | Orbit the camera |
| Click/tap a spot on the terrain | Jump the playhead to that moment |
| Scroll wheel | Zoom |

The glowing contour line hugging the surface is the playhead. It pulses with the live output level and dims when paused. The ground track along the front edge mirrors the timeline: playhead post, loop markers, and section ticks.

### Timeline strip

The waveform strip above the sliders scrubs on drag. The pale and ember vertical markers are the **loop start** and **loop end** — drag them to confine playback (and export) to a region. Numbers 1–9 mark nine equal sections of the loop region.

### Main controls

| Control | Range | Notes |
|---|---|---|
| **Stretch** | 1× – 60× (log) | How far time is smeared. |
| **Window** | 2048 / 4096 / 8192 / 16384 | Grain size in samples. Small = more rhythmic detail, less tonal smoothness; large = the opposite. 8192 suits most material at 44.1 kHz. |
| **Pitch** | ±36 semitones | Spectral pitch shift, independent of speed. Double-click/tap the slider to reset to 0. |
| **Volume** | 0 – 100 % | Output gain (soft-clipped safety limiter after it). |
| **Loop xfade** | 0 – 5 s | Crossfade at the loop point. A ghost read head continues past the boundary while the new position fades in — equal-power, click-free. |
| **LOOP** | on/off | Loop between the markers. |
| **PING-PONG** | on/off | Reverse direction at each boundary instead of jumping. Implies loop. |
| **REVERSE** | on/off | Play backwards. Combines with ping-pong. |

### Keyboard shortcuts (desktop)

| Key | Action |
|---|---|
| `Space` | Play / hold |
| `←` / `→` | Scrub by 2 % |
| `1` – `9` | Jump to section *n* of the loop region |

---

## Spectrum FX

**SPECTRUM FX** opens the processing bay — JavaScript ports of the spectral modules from Paul Nasca's `ProcessedStretch.cpp`, `BinauralBeats.cpp`, and `FreeEdit`, applied per-grain in the original order:

- **OCTAVE MIXER** — blend the sound with copies at −2, −1, +1, +1.5 (a twelfth), and +2 octaves.
- **FREQ SHIFT** — linear frequency shift in Hz (±2 kHz). Breaks harmonic ratios; deliberately dissonant.
- **FILTER** — band-pass or band-stop (BSTOP) between two edges, with high-frequency damping.
- **HARMONICS** — a comb of gaussian (or rectangular) bands at harmonics of a chosen fundamental. Turns anything into a chord.
- **SPREAD** — widens every partial by smoothing the log-frequency spectrum. Chorus-of-a-choir effect.
- **COMPRESS** — spectral RMS leveling; raises quiet spectra, tames loud ones.
- **FREE FILTER** — draw an arbitrary EQ curve (−60…+15 dB, 20 Hz–20 kHz). Click adds a point, drag moves it, double-click removes it.
- **STRETCH ENV** — draw a stretch *multiplier* (×⅛…×8, log) across the sample, so different parts of the sound stretch by different amounts.
- **BINAURAL BEATS** — Hilbert-based frequency shift, half down on the left channel, half up on the right, with a drawable beat-frequency envelope (0.1–30 Hz, log) and mono-blend control. Headphones required to perceive it.

Each module has its own ON/OFF; all envelopes use the same editor.

---

## Export

**EXPORT WAV** renders the loop region offline with every current setting — stretch, direction, FX, envelopes — and downloads a PCM WAV.

- **Bit depth**: 24-bit (default) or 16-bit.
- **Loop exports** render *through* the wrap and rotate the file so it opens with the crossfade already in progress — the resulting WAV loops seamlessly in a sampler or DAW.
- **Ping-pong exports** render the number of passes chosen in the `pp ×N` selector.
- Estimated outputs over 15 minutes ask for confirmation; over 1 hour are refused (browser memory).
- Rendering shows progress in the status line and never blocks the page.

Live playback state is untouched by an export; you resume exactly where you were.

---

## Privacy

Nothing leaves the machine. Audio is decoded, processed, visualized, and rendered entirely client-side. The only network requests are the two CDN fetches for the typeface and three.js at page load.

---

## Credits & licensing

- **Algorithm & original software** — Paulstretch (Paul's Extreme Sound Stretch), © 2006–2011 Paul Nasca — <https://hypermammut.sourceforge.net/paulstretch/> — GNU GPL v2.
- **Spectral DSP** — pitch & frequency shift, octave mixer, harmonics, spread, filter, compressor, free filter, stretch envelope & binaural beats ported to JavaScript from `ProcessedStretch.cpp`, `BinauralBeats.cpp` and `FreeEdit` by Paul Nasca.
- **Hilbert transformer** — allpass coefficients by Olli Niemitalo.
- **Source consulted** — mirror at <https://github.com/brianloveswords/paulstretch>.
- **Data visualization and playback modifications** — Kurt Kurasaki.
- **Libraries & platform** — three.js r128 (MIT), Web Audio API, WebGL.
- **Type** — JetBrains Mono via Google Fonts (SIL OFL).
- **JavaScript port, interface & 3D visualization** — built with Claude by Anthropic. The ported DSP remains under **GPL v2**.
