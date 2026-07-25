# Development notes

Architecture, algorithms, and hard-won constraints for `paulstretch-crt.html`. Read this before changing anything below the CSS.

- [Design constraints](#design-constraints)
- [File layout](#file-layout)
- [The Paulstretch engine](#the-paulstretch-engine)
- [The spectral FX chain](#the-spectral-fx-chain)
- [Offline rendering & seamless loop export](#offline-rendering--seamless-loop-export)
- [The 3D terrain](#the-3d-terrain)
- [The UI layer](#the-ui-layer)
- [CRT presentation layer](#crt-presentation-layer)
- [Browser & mobile constraints](#browser--mobile-constraints)
- [Testing](#testing)
- [Design history](#design-history)

---

## Design constraints

1. **One file, no build.** Everything — markup, CSS, DSP, 3D, UI — lives in a single HTML file (~80 KB). The only external dependencies are three.js r128 and one Google Font, both from CDNs. There is no bundler, transpiler, or framework; the file must open from disk.
2. **The DSP is a port, not an homage.** The spectral modules follow Paul Nasca's C++ sources closely enough that constants (`14.71280603`, the 0.368 rectangular threshold, the octave normalization floor of 0.5) carry over verbatim. Do not "clean up" magic numbers without checking them against `ProcessedStretch.cpp` — they're load-bearing. The port is GPL v2.
3. **The engine is presentation-agnostic.** Three complete UI skins (neon, hardware chassis, CRT terminal) have been built over the same `engine`/`viz`/`ui` objects with zero DSP changes. Keep it that way: `engine` must never read the DOM; `ui` owns all elements; `viz` touches only its canvas and `engine`'s public fields.

---

## File layout

The inline `<script>` is ordered so each section only depends on the ones above it:

| Section | Contents |
|---|---|
| **FFT** | `makeFFT(n)` — iterative in-place radix-2 with precomputed twiddles and bit-reversal table. Forward and inverse via a flag; inverse scales by 1/n. |
| **HILBERT** | `makeAP`, `makeHilbert` — two 4-stage allpass cascades (Niemitalo coefficients) producing an analytic pair for the binaural shifter. Stateful; one instance per channel. |
| **ENVELOPE EDITOR** | `makeEnvEditor(id, pts, cfg)` — the FreeEdit curve UI. Points are `{x,y}` in [0,1]; click adds (max 50), drag moves (x clamped between neighbors ±0.002), double-click removes interior points. Endpoints are fixed in x. |
| **WAV ENCODER** | `encodeWav(channels, sr, bits)` — 16/24-bit PCM. Applies the same `tanh` soft clip as live playback so exports match what was auditioned. 24-bit is written little-endian byte-by-byte with two's-complement wrap. |
| **DEMO** | `synthDemo(ctx)` — 8 s stereo bell cascade synthesized at load (inharmonic partials `1, 2.76, 5.40, 8.93`), detuned drone, noise swell. Exists so there's zero embedded audio data. |
| **ENGINE** | `engine` — everything below. |
| **VIZ** | `viz` — three.js terrain, locator, camera. |
| **UI** | `ui` — element wiring, timeline, FX bindings, export flow. |
| **BOOT** | Try/catch init of `viz` then `ui`; failures surface in the status line, and a 3D failure does not take audio down. |

---

## The Paulstretch engine

### Core loop (`genHop`)

Per hop, per channel:

1. Read `N` samples at `floor(pos)` (zero-padded at edges), multiply by the window.
2. Forward FFT → keep only **magnitudes** for bins `0…N/2`.
3. Run the magnitude spectrum through `processSpectrum` (the FX chain).
4. Assign every bin a **uniform-random phase**; mirror bins `N/2+1…N−1` as complex conjugates so the IFFT is real.
5. IFFT, window again, overlap-add at 50 %: output `hop = tail + first_half·win`, save `tail = second_half·win`.
6. Advance `pos += dir · (N/2) / stretch` (times the stretch-envelope multiplier when enabled).

That's the whole algorithm: random phase is what freezes the sound. The window is `(1−x²)^1.25` — Nasca's choice; with the double windowing (analysis + synthesis) and 50 % overlap it sums nearly flat, and the residual dip is patched by a fixed 1.1 makeup gain. If you change the window, re-derive that gain.

Output hops accumulate in per-channel `queue` arrays; `process()` (the ScriptProcessor callback) drains them, generating hops on demand. This decouples hop size from the 2048-frame audio callback.

### Playback state

- `pos` is a float sample index into the **source**; scrubbing = writing `pos`. Since each grain is analyzed fresh from `pos`, scrubbing is instant and artifact-free by construction.
- Direction = `(reverse ? −1 : 1) · (pingpong ? bounce : 1)`; `bounce` flips at loop boundaries.
- Loop wrap starts a **crossfade**: `startXfade` records a ghost position that keeps advancing in the old direction while the analysis blends ghost and new reads with equal-power weights over `xfadeTime` worth of hops. The blend happens in the *time-domain input* to the FFT — cheap and correct, since both reads share the randomized-phase resynthesis.
- `setWindow(N)` reallocates everything (window, scratch buffers, tails, queues). It's called on window-size change and on load; it silently resets in-flight grain state, which is fine because the next hop rebuilds from `pos`.

### Audio graph

`ScriptProcessor(2048, 1 in, 2 out) → gain → analyser → destination`, plus a silent `ConstantSource` feeding the input — **WebKit never fires `onaudioprocess` on a zero-input ScriptProcessor**. A muted-oscillator fallback covers old WebKit without `ConstantSourceNode`. A `tanh` soft clip guards the final output.

ScriptProcessor is deprecated but kept deliberately: it's the only way to stay single-file (an `AudioWorklet` needs a module URL; a Blob-URL worklet breaks under strict CSP). Revisit if single-file ever stops being a requirement.

---

## The spectral FX chain

`processSpectrum(freq)` mutates the magnitude array in the original module order (order matters and is part of the port):

**harmonics → freq shift → pitch shift → octave → spread → filter → free filter → compressor**

Notes per module:

- **Pitch/octave** share `psPitch(f1, f2, rap)`: integer bin remap; downward shifts *accumulate* energy into target bins (`+=`), upward shifts sample (`=`). This asymmetry is Nasca's and audible — keep it.
- **Harmonics** builds a gaussian comb, cutoff at `x² ≤ 14.71280603` (where `e^(−x²)` hits ~4·10⁻⁷); non-gauss mode thresholds the same comb at 0.368 (`1/e`).
- **Spread** resamples the spectrum to log-frequency, runs a bidirectional one-pole smoother twice (coefficient scaled by `8192/N` so perceived width is window-independent), and resamples back.
- **Filter**'s high damping is a per-bin decaying gain `dmp *= 1 − (hdamp/2)⁴` — an exponential tilt, not a shelf.
- **Free filter** pre-bakes a per-bin gain table (`updateFreeFilter`) from the envelope; the table is rebuilt on envelope edits and window changes, not per hop.
- **Compressor** normalizes by `rms^(−power)` with an RMS floor of 10⁻³ to avoid exploding silence.
- **Binaural** is *not* in this chain — it's a time-domain post-process on the interleaved output (`doBinaural`): mono-blend, then quadrature multiply against the Hilbert pair, −f/2 on left and +f/2 on right (swapped by REVERSE). It runs per audio callback in live playback and as a block-wise post-pass in export (with fresh Hilbert state, positions replayed from `posLog` so the envelope tracks correctly).

---

## Offline rendering & seamless loop export

`renderOffline(progress, loops)` reuses `genHop` verbatim — export is bit-identical in character to live playback. It saves and restores *all* live state (`pos`, `bounce`, tails, queues, Hilbert state) in a `finally`, and yields to the event loop every 32 hops so the progress line paints.

Three termination modes:

1. **One-shot** (loop off): render start → end, done.
2. **Ping-pong**: count direction flips until `targetPasses`, then keep rendering while the final crossfade drains.
3. **Seamless loop** (loop on, xfade > 0): render one pass, continue *through* the wrap so the crossfade back to the start is captured, plus `EXTRA = 4` continuation hops.

Mode 3 then **rotates** the buffer so the file *opens* mid-crossfade: layout `[pass][fade][extra]` becomes `[fade][blended join][rest of pass]`. The join between the fade's grain chain and the pass's grain chain is blended over the EXTRA region with equal-power weights — a hard splice between two independently random-phased overlap-add chains clicks even when the source content matches exactly. That blend is the subtle part of the whole exporter; don't remove it.

Guards: hard cap of 1 hour of output (`maxHops`), a `tailCap` so a crossfade longer than the loop region can't spin forever, size confirmation at 15 min, refusal at 60 min.

---

## The 3D terrain

Built once per loaded file from a mono mixdown (`viz.build`):

- 400 time columns × 176 log-frequency rows. Each column is a 4096-point Hann FFT at its position; rows average bins in log-spaced bands (~40 Hz → Nyquist); heights are `log10(1 + mag·8)`, peak-normalized to 5.2 world units.
- Surface: `PlaneGeometry` with per-vertex color on a monochrome amber ramp (ember → amber → hot → near-white by `h³`), translucent, `depthWrite:false`.
- **Wireframe** is a *separate decimated lattice*, not three.js wireframe mode: every `SX`-th column and `SZ`-th row (strides chosen for ~1.1-world-unit square cells → currently every 17th/14th line, ~7 % of full density). Each line still carries a vertex at **every** sample it crosses, read off the surface geometry, so wires hug the terrain instead of chording. Additive-blended.
- **Locator**: five stacked `Line`s at small vertical offsets fake bloom; their vertices resample the full-resolution heightfield `viz.hf` every frame at the playhead's time slice — so the contour keeps full detail even between wires. It pulses with analyser level and dims when paused. A ground-track group mirrors the timeline (post, loop ticks, section marks).
- Camera: orbit is manual spherical math (no OrbitControls — **absent from the r128 core build**). `fitCamera` frames the mesh's bounding sphere against both FOV axes, then zooms to 35 % of fit distance at yaw 30°, pitch 0.466.
- All sizing and raycasting uses the **canvas box**, not the window — the canvas doesn't fill the viewport. `resize()` early-outs when dimensions are unchanged; both a window listener and a `ResizeObserver` feed it (the console appearing/disappearing resizes the stage without a window resize).
- On rebuild, the old terrain and beam groups are `traverse`d and every geometry/material disposed — the wireframe owns its own geometry now, so shared-geometry double-dispose is not a hazard, but leaks were before.

Coordinate convention: world x = time (−W/2 … W/2), z = frequency (front = low), `scrub(frac)` maps a terrain raycast hit back through `(x + W/2)/W`.

---

## The UI layer

- `ui.els` caches every element once. All controls are wired in `ui.init` / `ui.wireFx`; FX sliders use a tiny `bind(id, fn)` helper that also runs `fn` once for initial state.
- **The `<input type=range>` elements are the single source of truth for parameters.** The hardware-skin era proved this out: custom widgets wrote to the hidden inputs and dispatched `input` events, and every handler kept working. Any future custom control must do the same — never bypass the inputs.
- Timeline pointer logic: a 9-px tolerance around either loop marker grabs the marker; otherwise the pointer scrubs. Marker drags clamp to a 2 % minimum region.
- `afterLoad` reveals the console **before** building the visuals — the tube and waveform strip must measure their final boxes first. Keep that ordering.
- Status line (`#status`) is the app's only messaging channel: decode errors, render progress, export results, audio-unlock hints all go there. No modals except the export size `confirm`/`alert`.
- The play button has a fixed width so `▶ PLAY` / `❚❚ HOLD` can't reflow the row; labels are `white-space:nowrap` for the same reason (a wrapping label once misaligned a control at narrow widths).
- Keyboard handling ignores events when an `INPUT` has focus, and digit shortcuts bail on modifier keys.

---

## CRT presentation layer

The current skin draws the *entire* UI on one simulated tube:

- `#crtfx` is a `position:fixed` overlay at `z-index:50`, `pointer-events:none`, stacking scanlines, phosphor bloom (with a keyframed flicker), a rolling retrace band, and a heavy vignette **over both the visuals and the controls** — that shared glass is what unifies the look.
- Palette is variables only: `--amber` family plus `--ember` for end-of-range markers. Glow = `text-shadow`/`box-shadow` in the amber family. Active toggles are inverse video (`background: var(--amber); color: var(--screen)`).
- Sliders are restyled native inputs: 2px glowing track, 12×20 block-cursor thumb, 28px total hit height. Both `-webkit-` and `-moz-` pseudo-element sets must be kept in sync.
- `prefers-reduced-motion` disables the spinner, roll bar, and bloom flicker. `env(safe-area-inset-*)` pads the header and console on notched phones.

---

## Browser & mobile constraints

Learned the hard way; all still live in the code:

| Constraint | Handling |
|---|---|
| iOS won't start audio outside a user gesture, and `resume()` after an `await` doesn't count | `primeAudio` creates+resumes the context **synchronously** inside click handlers; capture-phase `pointerdown`/`touchend` listeners retry the unlock |
| WebKit ScriptProcessor with zero inputs never fires | Silent `ConstantSource` (or muted oscillator) feeds input 1 |
| Older Safari lacks promise-form `decodeAudioData` | Wrapped to support callback and promise forms |
| iOS file picker greys out files with missing MIME types | Long-press / right-click the picker button temporarily drops the `accept` attribute |
| Touch scrolling vs. slider drags | Horizontal sliders: `touch-action:pan-y`; vertical minis: `pan-x`; stage/timeline/envelopes: `none` |
| Wheel direction follows the OS "natural scrolling" preference | Accepted; browsers don't expose the preference for detection |
| `devicePixelRatio` memory blowups | All 2D canvases clamp DPR to 2 |
| Console appearing changes stage size without a window resize | `ResizeObserver` on the stage canvas |

Vertical range inputs use `writing-mode:vertical-lr; direction:rtl` — supported in current Chrome/Firefox/Safari; very old Safari falls back to horizontal rendering, which is degraded but functional.

---

## Testing

There's no test framework; the file is validated by scriptable checks that have caught real bugs (an unclosed `<div>`, a stale element reference, a wheel-direction regression). Reproduce them with Node + Python:

1. **JS syntax**: extract the inline script, `node --check`.
2. **Structure**: `<div>` open/close balance; every `getElementById('x')` / `$('x')` resolves to a defined `id`; no orphaned ids after refactors.
3. **CSS**: `var(--x)` references all defined; brace balance.
4. **DSP smoke tests** (stub the DOM/canvas, `eval` an extracted section):
   - *Texture/knob era*: exercised every pointer/wheel/key handler against a canvas stub and swept `draw()` across (and beyond) the value range.
   - *Wireframe lattice*: assert every segment joins grid-adjacent indices, indices stay in `[0, cols·rows)`, and count segments vs. full density.
   - *Tile textures*: mean/min/max levels and seam continuity (`|col₀ − col_{S−1}|` must match interior column deltas).
5. **Fact checks after edits**: grep that removed features left no references (`makeKnob`, `screw`, `hammertone`…).

If you add a subsystem, add an extractable seam (a comment header) so it can be eval-tested the same way.

---

## Design history

Context for "why is it like this," and what was proven at each step:

1. **v1 — neon synthwave.** Original build: magenta/violet/cyan gradient skin, full-window canvas, gradient-styled native sliders.
2. **v2 — hardware chassis.** Amber CRT for the 3D view inside a bezel; canvas-drawn bakelite rotary knobs over the hidden range inputs; procedurally generated hammertone, then matte-grey, painted-metal chassis; machined switches and screws (screws later removed — absolutely-positioned decoration misaligns under scaling). Proved the inputs-as-source-of-truth pattern and forced the canvas-box sizing fix in `viz`.
3. **v3 — current.** The chassis fought mobile: knobs are imprecise under touch and the bezel wasted viewport. The whole skin was rebuilt as a single edge-to-edge amber terminal; parameters returned to (restyled, touch-sized) native sliders; the CRT overlay extended across the entire UI; the console became an independently scrolling readout capped below the always-visible visual field. The engine, visualizer internals, exporter, and all keyboard/timeline behavior carried over untouched — which is the whole argument for constraint #3.

Regressions worth remembering: the wireframe was once 140 k lines of solid haze (now a decimated lattice at ~7 %); the play button once resized between PLAY and HOLD; a wrapping label once knocked one knob out of line; exports once clicked at the loop join until the rotation-blend was added.
