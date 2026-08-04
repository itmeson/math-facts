# Multiplication Facts

A single-file practice app for building automaticity with multiplication facts. No build step, no
dependencies, no server — open `index.html` in a browser and it works.

**Live:** https://USERNAME.github.io/math-facts/

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

## Deploying

GitHub Pages serves this as-is from the repository root — there's nothing to build. `.nojekyll` tells
Pages to skip Jekyll processing.

## License

MIT
