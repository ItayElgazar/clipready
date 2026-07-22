# Gate 2 — content-verify after preview

A single editing pass leaves retakes in. The reliable failure mode: the abandoned
first attempt and the keeper render as **one continuous clean-audio span**, so a
waveform / visual self-eval looks perfect while the *content* is duplicated across
the join. The fix is to **re-transcribe the rendered preview at each join** — which
is exactly what `video-editing verify` does. Run it for every real cut.

## Run it

```
video-editing verify --job-dir "<job>" --json
```

- Default (per-join): re-transcribes a ~6s window centered on each cut boundary
  (from `cut_boundaries.json`), **healing OFF**.
- `--full`: re-transcribes the whole preview and also checks head/tail dead air.

It reports, per join:
- **duplicated phrase across the join** → a retake left in (severity `error`). It only
  flags **near-adjacent** duplication (the retake fingerprint), not legitimate topical
  repeats spaced apart.
- **over-long gap straddling the join** → dead air not tightened (warning, or error if large).

Writes `verify.json` (same shape family as `review.json`); exit code 2 on `needs-fixes`.

## Adjudicate + fix

For any finding, eyeball it: `video-editing inspect --job-dir "<job>" --join <i>` and read
the PNG (clipped word, leftover false-start syllable, audio pop, broken frame?).

Then FIX the EDL and re-render + re-verify:
- **Retake still present** → set that range's END at the **last clean word BEFORE the
  abandoned attempt**.
- **Intentional repeat wrongly cut** → restore it.
- **Clipped boundary** → re-snap to the transcript word boundary.
- **Leading / trailing dead air** → trim it.

Re-snap every edge to a word boundary; keep pads tight (30–200ms); keep `grade:"none"`,
`overlays:[]`; update `total_duration_s`. Re-render `--mode preview`, re-run `verify`.
Cap at ~3 fix passes; if findings persist, surface them rather than looping forever.

A real retake left in / an intentional repeat wrongly cut / a clipped word at a join /
leading-or-trailing dead air ⇒ not clean. Minor taste calls are not defects.
