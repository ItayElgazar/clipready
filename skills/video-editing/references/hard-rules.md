# Hard rules (production correctness — non-negotiable)

These produce **silent failures** if violated. The CLI enforces the rendering
rules internally; the EDL-authoring rules are on you.

## The CLI guarantees (do not try to replicate by hand)

- **Per-segment extract → lossless `-c copy` concat** (no double-encode).
- **30ms audio fades at every segment boundary** (`afade` in/out) — which is why
  tight pads (down to ~30ms) at retake joins are safe.
- **Loudnorm** to −14 LUFS / −1 dBTP / LRA 11 (two-pass for final, one-pass for preview).
- Subtitles, if ever enabled, are applied LAST in the filter chain.
- Word-level verbatim ASR with auto-heal; transcripts cached per source.

## EDL-authoring rules (yours to uphold)

- **Never cut inside a word.** Snap every edge to a transcript word boundary.
- **Pad every cut edge** (30–200ms). Scribe timestamps drift 50–100ms; padding absorbs it.
  Tightest (~0.03–0.05s) at retake joins, staying inside the silence.
- A **retake range ends at the last clean word BEFORE the abandoned attempt.**
- Clipped tokens ending in `-` (`s-`, `re-`, `li-`) are danger zones — leave an 80–150ms
  guard band before them when there's room.
- Keep the first token of the chosen take even if it's a connector ("and/so/but").
- `grade:"none"`, `overlays:[]`, no subtitles (clean-cut pass).
- Keep `total_duration_s` = Σ(end−start); ranges chronological + non-overlapping.

## The two mandatory gates

1. **SCREEN before cutting** — read `flags.txt`; resolve every [A], judge every [B]. See `scribe-collapse.md`.
2. **CONTENT-VERIFY after preview** — `video-editing verify`; a visual/waveform check alone
   misses a retake left in. See `adversarial-review.md`.

## Anti-patterns (consistently fail)

- Trusting the transcript without screening it (Scribe collapses/drops hide retakes).
- Running ffmpeg or ad-hoc helper scripts by hand — forbidden; use only the `video-editing` CLI.
- A visual-only check at a retake join (the duplicated content reads as continuous clean audio).
- Re-transcribing an unchanged source (paid + pointless; it's cached).
