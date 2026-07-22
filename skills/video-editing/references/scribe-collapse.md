# Gate 1 — the Scribe collapse / drop gotcha

The single most important thing before reasoning about cuts: **retakes hide in
the transcript.** Trust the text without screening and you will leave duplicated
content in the cut.

## What happens

When Scribe hits a hard-to-segment chunk (fast speech, a stutter, a false start
immediately followed by the retake), it sometimes:

- **Collapses** the chunk into ONE over-long token (a "word" lasting >~1.2s), or
- **Drops** a stretch of audio silently — no tokens for that span.

Either way the retake is invisible in the text, a waveform looks fine, and the
cut "reads" clean — but the abandoned take is still in the render.

## How the CLI + you catch it

1. **`video-editing transcribe` auto-heals** most collapses/drops (re-transcribes the
   suspect window in isolation and splices the recovered words back). Watch its log.
2. **`video-editing transcribe` writes `flags.txt`** — read it (Gate 1):
   - **[A]** over-long / collapsed tokens (>~1.2s) — content is hidden inside one of these.
   - **[B]** repeated phrases — the retake fingerprint (false starts, reworded restarts).
   - **[C]** audio events + big gaps.
   Resolve every [A]; judge every [B] (retake vs intentional).
3. **Verify coverage** via `coverage.json`: the last word must end within ~2s of the
   media duration. If it ends well before EOF, a tail was dropped.

## When a span is still suspect

Inspect it: `video-editing inspect --job-dir <job> --video vertical_src --start <a> --end <b>`
and read the filmstrip+waveform PNG to see what was actually said before choosing the cut.

> Screening + inspecting suspicious windows is what isolates retakes Scribe collapsed
> into one token. Skipping it is the #1 cause of a "clean-looking" cut that still has a
> retake in it.
