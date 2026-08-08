# Multiplication Facts

A single-file practice app for building automaticity with multiplication facts. No build step, no
dependencies, no server — open `index.html` in a browser and it works.

**Live:** https://itmeson.github.io/math-facts/

## How it works

Problems are presented one at a time. Type an answer and press Enter (or use the on-screen keypad).

- **Right on the first try** → brief affirmation, then it waits for Enter before moving on.
- **Wrong** → "Not quite. Try once more." One more attempt.
- **Wrong twice, or "Show me"** → the correct answer stays on screen until you press Enter.

A timer runs during every problem but is never shown. The goal is recall, not racing, and a visible
clock encourages guessing.

## Facility

Each fact gets a facility score from 0 to 100, computed over the last 6 encounters with it:

```
facility = 100 × accuracy × (0.35 + 0.65 × speed)
```

- **accuracy** — mean per-encounter credit: `1.0` right on the first try, `0.4` right on the retry,
  `0` for a miss or a give-up.
- **speed** — linear from 1.5 s (full credit) to 6.0 s (none), taken from the median of *unaided
  first-try solves only*. A retry time measures re-calculation, not recall, so it isn't counted.
  When there is no clean solve to time yet, speed falls back to a neutral `0.4` rather than the
  worst case.

For reference, six identical encounters produce:

| Pattern | Facility |
| --- | --- |
| First try, ~1.2 s | 100 |
| First try, 6 s | 35 |
| Second try every time | 24 |
| Missed or given up | 0 |

Slow-but-correct scores low on purpose — that pattern means the fact is being computed rather than
recalled, which is exactly what practice is meant to eliminate.

Problem selection is weighted toward weakness: `weight = 0.5 + 3.5 × (1 − facility/100)^1.5`, with
untested facts at `4`. The fact just asked is never repeated immediately.

## Tuning

The constants worth adjusting are at the top of the `<script>` block in `index.html`:

| Constant | Default | Meaning |
| --- | --- | --- |
| `WINDOW` | `6` | Encounters per fact that count toward facility |
| `FAST_MS` | `1500` | At or below this, fully automatic |
| `SLOW_MS` | `6000` | At or above this, no speed credit |
| `SECOND_TRY` | `0.4` | Credit for a correct retry (`0.65` scores such facts near 40) |
| `NEUTRAL_SPEED` | `0.4` | Assumed speed when nothing clean has been timed |

Credit is applied when scores are read, not when they're saved, so changing these re-scores existing
history rather than invalidating it.

## Data

Everything is stored in `localStorage` under `mathfacts.mult.v1`, per browser and per device —
nothing is uploaded and there are no accounts. Records saved before partial credit existed used a
boolean and are still read correctly. "Erase all progress" on the Progress page clears it; range and
keypad preferences survive.

Practice range is selectable up to 9×9, 10×10, 12×12, 15×15, or 20×20. Widening the range brings in
new facts; narrowing it hides results without deleting them.

## Saving, moving, and handing in progress

The **Progress data** section at the bottom of the Progress page has three buttons.

**Save a copy** writes `mathfacts-<name>-<date>.json`. The name is asked for once, then remembered.
The file is plain, uncompressed JSON — open it in a text editor and you can read it.

```json
{
  "app": "mathfacts", "schema": 1,
  "student": "Sam R.", "exported": "2026-08-08T18:22:11.402Z", "range": 12,
  "totals": { "asked": 431, "firstTry": 302, "timeMs": 812340, "timed": 302, "scoped": false },
  "facts": { "7x8": [ { "s": 1, "ms": 2140, "at": 1754332911402 } ] }
}
```

**Load & combine** merges a saved file into what's already on the device. Every encounter carries a
timestamp, so the merge interleaves both histories in true time order and de-duplicates on the exact
`(s, ms, at)` triple. Practising on a phone and a laptop and combining both is safe, importing the
same file twice does nothing, and because facility reads the most recent 6 encounters, a merge yields
the genuinely most recent 6 across both devices. Up to 24 encounters per fact are retained.

**Load & replace** clears local progress first. For a shared or borrowed device.

Import refuses any file whose `schema` it doesn't recognise rather than guessing — the record format
has already changed once. Malformed individual records are dropped rather than failing the whole
file, and the older boolean form is read correctly.

One honest caveat: `totals` are running counters that can't be merged without double-counting, so
after any import they're recomputed from retained history and the labels switch from "all time" to
"retained history" (tracked by `totals.scoped`). Before any import they're true lifetime figures.

### Moving between your own devices

Save a copy into a synced folder (OneDrive, Drive, Dropbox), then use **Load & combine** on the other
device. There's no server, so this is deliberate rather than automatic — which also means it works
offline and there's nothing to sign in to.

Progress is stored per origin. Data from opening `index.html` off the disk won't appear on the
Pages URL, and vice versa.

### Handing in to a teacher

Students **Save a copy** and upload the file to a Canvas assignment. Canvas prefixes each file with
the student's name on bulk download, and the export carries the name inside it too, so a downloaded
zip is unambiguous either way.

## Deploying

GitHub Pages serves this as-is from the repository root — there's nothing to build. `.nojekyll` tells
Pages to skip Jekyll processing.

## License

MIT
