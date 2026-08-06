# rtttl-bench

A single-page RTTTL player for testing ringtone strings before you flash them to
hardware. Paste a tune, hear it, see it as a piano roll, and audition it through
models of the transducers it will actually come out of — including a preset for
the piezo buzzer used in Apollo Automation's ESPHome devices.

Built because a tune that sounds fine through laptop speakers often sounds
nothing like that through a 12 mm piezo disc, and the difference is worth
knowing before you write the automation.

**Live:**: https://rtttl.jjajan.dev/

## What it does

- **Faithful RTTTL parsing** — the Nokia spec pattern with a lenient fallback
  that warns instead of failing. Spec defaults (`d=4`, `o=6`, `b=63`), dotted
  notes, and both `,` and `;` separators.
- **Correct pitch** — A4 = 440 Hz by default, with an option to reproduce the
  nine-semitone-sharp behaviour of the original rtttl.js demo for comparison.
- **Eleven voices** — four bare oscillators, four transducer models, two chip
  pulse widths, and a bell. The transducer voices apply a highpass, a resonant
  peak, and a hard PWM gate, which is what makes a buzzer sound like a buzzer.
- **PWM duty control** — the analogue of ESPHome's `gain:`, which on a piezo
  changes timbre as much as level.
- **Piano roll** — pitch against time with a live playhead, so you can see
  where rests and dotted notes actually land.
- **Stop vs cancel** — stop lets the sounding note finish; cancel cuts now.
- **Service notes panel** — six tabs documenting how every part works and,
  importantly, which numbers are measured and which are estimated.

No build step, no dependencies, no analytics, no storage. One HTML file.

## Layout

```
.
├── public/
│   ├── index.html      the entire application
│   └── _headers        cache + security headers (Cloudflare serves natively)
├── wrangler.toml       Workers static-assets config
├── CLAUDE.md           context for Claude Code
├── LICENSE             MIT, plus upstream third-party notice
└── README.md
```

Everything lives in `public/index.html` — markup, styles, and script. That is
deliberate; see CLAUDE.md for why, before you reach for a bundler.

## Local development

Any static file server works, since there is nothing to compile:

```bash
python3 -m http.server 8000 --directory public
# then open http://localhost:8000
```

Or, to test against Cloudflare's own runtime and pick up `_headers`:

```bash
npx wrangler dev
```

Audio requires a user gesture in every browser, so the first Play click is what
unlocks the `AudioContext`. That is expected, not a bug.

## Deploying to Cloudflare

### Option A — Git integration (recommended)

Connect this repo once and every push to `main` deploys automatically.

1. In the Cloudflare dashboard, go to **Workers & Pages → Create → Workers →
   Import a repository**.
2. Select this repo.
3. Leave the build command empty. Set the deploy command to `npx wrangler deploy`
   (or accept the default, which reads `wrangler.toml`).
4. Deploy. Subsequent pushes to `main` redeploy on their own.

### Option B — CLI

```bash
npx wrangler deploy
```

### A note on Pages vs Workers

Cloudflare now recommends Workers with static assets for new projects, and as of
early 2026 Workers reached parity with Pages for static assets, SSR, and custom
domains. Pages remains fully supported; this repo targets Workers because it is
where new features land first, and because a static site can grow a backend later
without re-platforming.

Whichever you choose, the choice is sticky: a Pages project created through Git
integration cannot be switched to Direct Upload later, and vice versa.

If you would rather use Pages, point a Pages project at this repo with an empty
build command and `public` as the build output directory. `wrangler.toml` is
ignored in that case.

## Accuracy and what is guessed

The **Service notes** panel in the app itself is the real documentation, and it
tags every claim as one of three things: read out of published source, stated by
a vendor, or estimated by me.

The short version: the parser and the default envelope are a faithful port of
published code. Apollo's use of a passive piezo on an ESP32 `ledc` PWM output is
documented by Apollo. The filter chains modelling each transducer are **not**
measurements of specific parts — they are plausible models of the class of
component, and the resonant frequencies in particular are estimates. If you have
one of these on a bench and can measure where the peak actually sits, changing it
is a one-line edit in the `VOICES` object.

Known gaps are listed in the app under **Not modelled**: LEDC frequency
quantisation, enclosure resonance, drive nonlinearity, and a few others.

## Credits and licence

MIT. See [LICENSE](LICENSE).

The RTTTL parser and the default note envelope are derived from
[1j01/rtttl.js](https://github.com/1j01/rtttl.js) by Isaiah Odhner, MIT licensed,
and that notice is reproduced in full in `LICENSE`. The voice modelling, piano
roll, interface, and documentation are original to this project.

RTTTL itself is a Nokia format; a copy of the specification circulates widely if
you need the details the app does not cover.
