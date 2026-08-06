# CLAUDE.md

Context for agentic work in this repo. Read this before editing anything.

## What this is

A single-file RTTTL player used to audition ringtone strings before flashing
them to ESP32 hardware. The whole application is `public/index.html` — markup,
CSS, and JavaScript in one file, deployed as a static asset by Cloudflare
Workers.

The audience is one technical person auditing tunes. Optimise for correctness
and for making the model's assumptions legible, not for feature count.

## Hard invariants

Violating any of these silently breaks the point of the project:

1. **Stay a single file.** No bundler, no framework, no npm dependencies, no
   external scripts. The only network request is the Google Fonts stylesheet.
   If asked to "modernise" or "refactor into modules", push back first — the
   single-file property is why this can be dropped anywhere and audited in one
   read.
2. **Storage is settings-only.** Exactly one `localStorage` key,
   `rtttl-bench:settings`, holding the bench control values (voice, duty,
   volume, tuning, loop) plus whether the Service notes panel is open
   (`notes`), as JSON. No cookies, `sessionStorage`, or IndexedDB;
   nothing else may be stored, and a visible "Reset settings" control must
   always exist. Stored values are untrusted input — validate against the
   controls before applying.
3. **No telemetry.** No analytics, no beacons, no error reporting services.
4. **The parser is a port, not a rewrite.** The RTTTL parsing and the `smooth`
   envelope come from `rtttl.coffee` and `player.coffee` in
   [1j01/rtttl.js](https://github.com/1j01/rtttl.js) (MIT, Isaiah Odhner). If
   you change parsing behaviour, verify against a second implementation and say
   so in the Origin tab. Attribution in `LICENSE` must stay.
5. **Keep the Service notes honest.** Every factual claim in the notes panel
   carries one of three tags: `from source` (read out of published code),
   `from docs` (stated by a vendor), `estimated` (a modelling guess). If you add
   or change a voice, filter, or figure, update the corresponding note and tag
   it correctly. Presenting an estimate as a measurement is the worst possible
   failure mode here.

## Architecture

Read in this order; the file is organised the same way.

- **`parseRTTTL(str, tuning)`** — returns `{name, o, d, b, notes[], warnings[],
  total}`. Each note carries `{token, name, octave, rest, seconds, semi,
  frequency, at}` where `at` is its offset in seconds from the start. Strict
  spec regex first, lenient fallback second with a warning pushed. Throws on
  anything neither pattern matches.
- **`VOICES`** — the data model for every voice: `partials` (oscillator type or
  pulse `duty`, optional `ratio` and `gain`), `env` (a key into `ENVS`),
  `filters` (an ordered biquad chain), `trim` (output level), and optional
  `fixedDuty`. **The Voices table in the notes panel is generated from this
  object at runtime**, so it cannot drift — do not hand-write that table.
- **`ENVS`** — four envelope profiles. `smooth` is the ported one; `gate` models
  a hard PWM switch; `snap` and `decay` are in between and for the bell.
- **`applyEnv`** — schedules one note's gain envelope. Uses a monotonic `at()`
  helper so events can never be scheduled out of order, which Web Audio treats
  as an error. All timings scale down proportionally on short notes.
- **`buildVoice` / `schedule`** — the entire tune is written to the audio clock
  in one pass before playback starts. Timing is sample-accurate and does not
  drift, at the cost of needing a reschedule when a control changes mid-song.
- **Transport** — `startPlayback`, `stopPlayback` (graceful: current note
  finishes, rest is discarded, pending loop abandoned), `cancelPlayback`
  (immediate, 3 ms ramp to avoid a click), `endPlayback` (UI reset only).
- **`tokenizeRTTTL(str)`** — the editor's syntax highlighter, a **view layer
  only**. It returns `{s, e, cls, sev}` spans over the *raw* string. It must
  never call `parseRTTTL` or change it; to stop the two drifting it reuses the
  `STRICT`/`LOOSE` patterns and mirrors the parser's per-token normalisation.
  `parseRTTTL` throws on the first bad note, so the message line names one
  token while the editor marks them all — that difference is deliberate.
  `renderHighlight` paints the spans into a `<pre>` stacked under a transparent
  `<textarea>`; both layers must keep identical font, padding and wrapping or
  the colours visibly drift off the glyphs. Escape HTML — the input is
  user-controlled.
- **`paint` / `tick`** — canvas piano roll, redrawn per frame from the audio
  clock rather than from a timer.

## Domain notes worth not relearning

- **Pitch.** The upstream demo computes `440 * 2^(octave - 4 + index/12)`, which
  makes C4 = 440 Hz — everything is nine semitones sharp. This app defaults to
  the corrected `(index - 9)` form and offers the original under the Tuning
  select. Do not "fix" the demo option; it exists for comparison.
- **Why filters matter more than waveform.** A small piezo produces essentially
  nothing below ~1.5 kHz, so a tune in `o=5` reaches your ear as harmonics only.
  Swapping sine for square is a small change; adding the highpass and resonant
  peak is the big one.
- **Pulse waves** are built from Fourier coefficients (`2/(nπ)·sin(nπd)`, 96
  harmonics) via `createPeriodicWave`, so the browser band-limits them. Do not
  replace them with a naive square — it aliases badly at high octaves.
- **ESPHome `gain:`** maps to PWM duty cycle, not amplitude. That is why the
  duty slider changes timbre, and why it is labelled as it is.
- **Apollo preset.** Apollo documents a passive piezo on an ESP32 `ledc` output.
  They do not publish the part or its resonance. The 4.2 kHz peak is an estimate
  and is tagged as such — keep it that way unless someone measures one.

## Conventions

- Two-space indent, double quotes in JS, semicolons.
- Comments explain *why*, especially where a number is a physical claim. Do not
  strip them; they are the reason the file is auditable.
- Type sizes come from the `--fs-*` scale, spacing from `--sp-*`, radii from
  `--r-*`, all declared at the top of `index.html`. Every font-size in the file
  is a scale step — keep it that way, since a drift of near-identical sizes is
  what made the page look unfinished before. Structural dimensions (button
  heights, panel widths, canvas height) are still plain pixels; that is fine.
- The palette is **Gruvbox dark**. Exactly two colour roles exist —
  **chassis/UI** and the **syntax token palette** (`--tok-*`), which is used
  only inside the RTTTL editor. Do not add a hue outside those roles.
- **Flat terminal UI is the default.** The project is not committed to a
  skeuomorphic "bench instrument" look, and earlier versions of this file said
  otherwise — do not restore that. Physical-object styling is allowed in
  exactly one place, the LCD panel, because the piano roll genuinely lives in
  it and the metaphor does work there. Everywhere else, avoid gradients,
  offset shadows faking raised keys, and press-translate animations; they date
  the page badly. Buttons are a filled primary and an outlined secondary in
  the mono face.
- **Chrome must not imply meaning it doesn't have.** A previous revision had
  three coloured dots in the title bar, intended as status LEDs. They read as
  macOS traffic lights, were `aria-hidden`, and only ever duplicated state
  already shown in words — so they were removed. Do not add ornament that
  looks like instrumentation. The status bar shows only what is *not* visible
  elsewhere: transport state lives on the Play button's label, parse errors on
  the message line and the editor rail.
- The piano roll's colours are `--roll-*` custom properties read by
  `readRollColors()`. Do not hard-code colours in `paint()` again — that is what
  the cache exists to prevent.
- Interactive elements need visible `:focus-visible` styles and correct ARIA.
  The tabs are a real tablist with arrow-key navigation; keep it working.

## Verification

There is no test suite. Before claiming a change works:

```bash
# JS syntax check
python3 -c "import sys;print(open('public/index.html').read().split('<script>')[1].split('</script>')[0])" > /tmp/app.js
node --check /tmp/app.js
```

For parser changes, check a known tune against a second implementation. The
opening of `Back to the Future` should resolve to 450 ms @ 784 Hz, a 75 ms rest,
450 ms @ 523.3 Hz, then another 75 ms rest.

For envelope changes, confirm scheduled times stay monotonic and inside the note
for every profile across note lengths from ~12 ms to several seconds.

Then load the page and actually listen. Audio bugs are frequently silent in a
syntax check and obvious in one second of playback.

## Deployment

`wrangler.toml` targets Cloudflare Workers with static assets serving `./public`.
No build step. `npx wrangler deploy`, or push to `main` if Git integration is
connected. Do not set `not_found_handling` to `single-page-application` — this is
not an SPA.
