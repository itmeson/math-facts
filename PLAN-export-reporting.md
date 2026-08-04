# Plan: export / import / sync, and a teacher report

Status: **not built** — design notes to argue with before writing code. Nothing here is committed to.

Three separable pieces, in the order I'd build them:

1. Export and import in the app (unlocks everything else)
2. Cross-device sync (falls out of #1 almost for free)
3. A teacher-side report script over a folder of exports

---

## 0. Prerequisite: a student name

Exports are currently anonymous, and a folder of `progress.json` files is useless to a teacher. Add a
name field on the Progress page, stored at `db.settings.student`.

Prompt for it on first export rather than on first launch — asking a kid to type their name before
they can do anything is a bad first impression, and a solo user at home never needs it.

See **Privacy** at the bottom before deciding what goes in this field.

---

## 1. Export / import

### File format

```json
{
  "app": "mathfacts",
  "schema": 1,
  "student": "Sam R.",
  "exported": "2026-08-04T18:22:11.402Z",
  "range": 12,
  "totals": { "asked": 431, "firstTry": 302, "timeMs": 812340, "timed": 302 },
  "facts": {
    "7x8": [ { "s": 1, "ms": 2140, "at": 1754332911402 }, { "s": 0.4, "ms": 0, "at": 1754333042118 } ]
  }
}
```

This is nearly the `localStorage` blob as-is, plus `app`, `schema`, `student`, and `exported`.
Deliberately not compressed and not encoded — a teacher should be able to open one in a text editor
and see what they've been handed. It's a few tens of KB.

`schema` matters more than it looks. The stored records already changed shape once (boolean `ok` →
numeric `s`), and the reader must not guess. Import should reject anything with a `schema` it doesn't
know rather than silently misreading it.

### Export

Blob + `URL.createObjectURL` + a synthetic `<a download>`. Filename
`mathfacts-sam-r-2026-08-04.json`, slugified from the name.

### Import: merge, don't replace

This is the part worth getting right. Every encounter already carries an `at` timestamp, so:

```
for each fact key:
  merged = existing ++ incoming
  dedupe on (at, s, ms)          # same encounter arriving twice
  sort by at
  keep last 24                    # existing cap
```

Because facility only reads the last `WINDOW` (6) encounters, a merge of two devices yields the
genuinely most recent 6 across both — which is the correct answer, not an approximation.

`totals` can't be merged this way (they're running counters, and the same practice could be counted
twice). Two options: recompute them from the merged `facts` arrays — accurate but only covers the
last 24 per fact — or take the max of each field, which is wrong but monotonic. I'd recompute and
accept that lifetime totals become "totals over retained history," then rename the labels to match.
Silently-wrong counters are worse than honestly-scoped ones.

Offer **Merge** and **Replace** as separate buttons. Replace is what you want when a device has junk
data on it from someone else trying the app; merge is the everyday case. Confirm on replace.

### UI

Four controls in a new "Progress data" section at the bottom of the Progress page: name field,
Export, Import (file input), and the existing Erase. Keep them well away from the heatmap.

---

## 2. Cross-device sync

There's no server, so there is no true background sync without adding one. Ranked by effort:

**a. Export file → cloud drive → import.** Zero new code beyond #1. Works today, works offline, no
accounts. The friction is real but the failure modes are all visible.

**b. Transfer code.** Base64 of a minified payload (drop `ms` precision to 100ms units, drop `at` to
minutes since an epoch, single-letter keys) shown in a `<textarea>` to copy, with a matching paste
box. Probably 3–5 KB of text for a well-used 12×12 — too long to retype, fine to copy/paste or send
over a chat app. **Worth it specifically because of locked-down school iPads**, where the Files app
is often the hardest part of the whole workflow. This is the one I'd actually build after (a).

**c. QR code.** Charming, and it caps out around 2–3 KB even at high density. A student with real
history won't fit. Skip unless the payload gets much smaller than I think.

**d. A real backend.** Firebase/Supabase free tier, or a Google Apps Script endpoint writing to a
Sheet. This is the only option that gives a teacher a live dashboard instead of a collection ritual.
It is also the option that turns a static file you fully control into a service that stores student
data, with the review that implies at a school. Not a "later that afternoon" decision.

My recommendation: (a) immediately, (b) when a second device actually annoys you, (d) only if you end
up running this with multiple classes and the collection step becomes the bottleneck.

---

## 3. Teacher report script

`report.py`, Python 3 standard library only — no pip install, runs on a school laptop.

```
python report.py exports/ -o report.html
```

Reads every `*.json` in the folder, skips anything that isn't a valid export (with a warning naming
the file), writes one self-contained HTML report.

### What it should compute

**Per student**

- Average facility, count at 80+, and total encounters. Volume must be shown next to facility —
  facility 30 over 4 encounters and facility 30 over 60 encounters are different situations and the
  same number hides it.
- Ten weakest facts.
- **Slow-but-accurate vs. genuinely unknown.** The data separates these and the distinction drives
  what you do next: high accuracy with long medians means a working strategy that hasn't been
  automatized (drill it), while low accuracy means the fact isn't there (reteach it). Report them as
  two separate lists. I think this is the single most useful thing in the report.
- Last practiced date, from `max(at)` — tells you whose data is stale before you read anything into
  it.

**Class-wide**

- Mean-facility heatmap, same color scale as the app, so it reads identically to what students see.
- **Fact families:** aggregate by factor (all facts involving 7, involving 8, …). This is the view
  that changes tomorrow's lesson, in a way that 81 individual cells does not.
- Hardest facts by class median, with the number of students contributing. Expect the usual suspects
  (7×8, 6×8, 7×9, 8×9) plus whatever is specific to this group — the second part is the interesting
  part.
- **Commutative gaps:** students where `facility(a,b)` and `facility(b,a)` differ by more than ~30.
  Points at how a fact is being represented rather than whether it's known, and it's invisible
  without exactly this kind of paired data.
- Coverage warning: facts nobody in the class has attempted, so an empty region of the heatmap isn't
  misread as a weakness.

### Cautions to build in

- **Small-n suppression.** Don't report a class median for a fact only two students have tried. Show
  the n, and grey out anything under a threshold.
- **Range differences.** A student practicing 9×9 has no data above 9. Aggregate over the
  intersection of ranges, or report per-range bands; don't average across them silently.
- **Facility is not a grade.** The report should say so in a line at the top. A number between 0 and
  100 next to a student's name will be read as a percentage score by someone, eventually, and it
  isn't one — it's a recall-fluency estimate over the last six encounters, and it moves fast in both
  directions.

### Output

Single HTML file, inline CSS, no CDN — it'll get emailed and opened offline. A `--csv` flag for the
per-student summary table would let you pull it into a spreadsheet without me guessing at what
columns you want.

---

## Privacy

Exports contain fact-level performance plus whatever is typed in the name field. No account
identifiers, no device fingerprint, no free text from the student. That's a narrow payload — but it
becomes a student record the moment it lands in your hands, regardless of how little is in it.

Two things worth settling before this runs with a live class:

- **Real names or roster IDs?** If students type an ID and you hold the mapping separately, the
  export folder stops being a student record and the report becomes much easier to share with a
  co-teacher or aide. Costs nothing to decide now and is painful to retrofit.
- **Who owns this call at Seattle Academy?** Collecting performance data from minors, even
  self-reported and even this thin, is normally somebody's policy question rather than a teacher's.
  Worth one conversation before the first collection, not after.

Option (d) above — a real backend — changes this materially, because the data would leave the
student's device automatically rather than by a deliberate act. I'd treat that as a separate
decision requiring a separate conversation.
