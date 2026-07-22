# Editing brief — authoring the EDL

The standing style for a single talking-head vertical reel and the exact
step-by-step for authoring `<job>/edl.json`. All execution is via the
`video-editing` CLI; you only write the EDL.

## Standing brief (apply exactly)

- **Output VERTICAL 9:16.** The CLI bakes `vertical_src.mp4`; point the EDL at it.
- **CUT:** bad retakes, false starts, cut-off / stuttered words — keep the **LAST
  complete take**, EXCEPT keep the **CLEANEST** take when an earlier take is clean
  and a later one stutters into completion.
- **TIGHTEN EVERY PAUSE:** trim 0.3–1s inter-phrase gaps down to ~0.12–0.14s of air.
- **COLD-OPEN** on the strongest hook (trim weak/dead intro).
- **TRIM outro dead air** (footsteps / ambient / silence / trailing breaths / device noise).
- **CLEAN CUTS ONLY:** `grade:"none"`, no subtitles, no overlays.
- **KEEP intentional rhetorical repeats.**

### Retake-vs-intentional discriminator (for a flagged repeated phrase)

- **RETAKE** (cut it, keep the last/cleanest complete pass) if the earlier
  instance cuts off mid-word (a fragment / trailing "-") OR is immediately restarted.
- **INTENTIONAL** (keep BOTH) if both instances are complete and serve rhetoric /
  parallel structure and are separated by other content.
- Decide every flagged repeat — never blind-cut.

## Steps

1. **Init + transcribe + screen** (CLI does these): `init-job` → `transcribe`.
   Then read `<job>/flags.txt` (Gate 1) and confirm `<job>/coverage.json`.
2. **Read the word-level transcript** at `<job>/word_dump.txt` (every word/audio_event
   as `[start-end] type: text` with `---- gap Xs ----` at gaps ≥0.30s). THINK HARD and
   list, with exact timestamps: every retake/false-start/stutter (and which take to KEEP),
   every silence (head/inter-phrase/outro), the cold-open point, the intentional repeats.
   When text and audio seem to disagree, `video-editing inspect --video vertical_src --start <a> --end <b>` and read the PNG.
3. **Write `<job>/edl.json`** — KEEP segments in source seconds; gaps between them are
   the cuts. Snap every in/out to a transcript word boundary (lead ~0.05s, trail ~0.08s;
   ~0.03–0.05s at retake joins, clamped to stay inside the silence and never include a
   false-start word). Ranges chronological, non-overlapping. `total_duration_s` = Σ(end−start).
   **A retake range must END at the LAST CLEAN WORD before the abandoned attempt.**
4. **Render** `--mode preview`, then **verify** (Gate 2), fix, repeat. Finalize `--mode final`.

See `cli-reference.md` for exact commands, `hard-rules.md` for correctness, and
`scribe-collapse.md` / `adversarial-review.md` for the two gates.
