---
name: video-editing
description: >-
  Edit a talking-head video reel into a tight, clean cut using ONLY the local
  `video-editing` CLI — never the video-use repo, never the clean-cut skill,
  never raw Python helpers or ffmpeg by hand. Removes bad retakes, false starts,
  stutters and cut-off words (keep the LAST complete take, but keep the CLEANEST
  take when an earlier one is clean and a later one stutters), tightens every
  pause to ~0.13s, cold-opens on the strongest hook, trims outro dead air.
  Vertical 9:16, clean cuts only (no grade/subtitles/overlays by default). Two
  mandatory gates: SCREEN before cutting and CONTENT-VERIFY (re-transcribe the
  rendered preview at every join) after. Use whenever the user drops a
  talking-head clip / reel / vlog / interview and wants it edited or tightened —
  phrasings like "edit this reel", "cut the bad retakes", "remove the false
  starts", "clean up this recording", "tighten this reel", "tighten this", "I
  flubbed some lines, fix it", "get rid of the dead air", "make a clean cut of
  this", or just a video path with "make this cleaner". Handles Hebrew/RTL.
  This is the single source of truth for reel edits; it supersedes and replaces
  the clean-cut and video-use skills.
---

# Video Editing (CLI-only)

Turn a raw talking-head recording into a tight, clean vertical cut. The local
**`video-editing`** CLI is the single source of truth and owns ALL execution
(probe, transcribe, screen, bake master, render, QC, content-verify). **You**
read the transcript and author the EDL; the CLI does everything executable.

## Install

Requirements: **Node ≥ 20**, **ffmpeg**/**ffprobe** on PATH, and an **ElevenLabs API key**
(`ELEVENLABS_API_KEY`) for transcription.

```bash
npm install -g video-editing                                  # the CLI
npx skills add ItayElgazar/clipready --skill video-editing    # this skill
```

## HARD RULE — execution boundary (read first)

- Use **only** the `video-editing` CLI (`video-editing <subcommand>`). Run
  `video-editing --help` to see commands.
- **NEVER** run ffmpeg/ffprobe/transcription by hand — the CLI owns all execution.
- If a capability seems missing, STOP and report it as a CLI gap. Do not reach around the CLI.
- Your only hand-authored artifact is `<job>/edl.json`.

## Prerequisites

- `video-editing` on PATH (`which video-editing`), plus `ffmpeg`/`ffprobe`.
- ElevenLabs key: the CLI owns it (env `ELEVENLABS_API_KEY` or `~/.config/video-editing/.env`;
  set once with `video-editing config set-key <key>`). Transcription is cached per source
  and is a paid call — never force a re-transcribe of an unchanged source.

## Workflow

1. **Init** — `video-editing init-job --source "<video>" --job-dir "/vercel/sandbox/jobs/<job_id>"`
   → job dir + `source.json`. Always pass `--job-dir` explicitly (there is no `~/Downloads` on
   the sandbox); use the job dir given in the task prompt.
2. **Transcribe + screen** — `video-editing transcribe --job-dir "<job>"` → word-level
   transcript + screening artifacts (`flags.txt`, `takes_packed.md`, `word_dump.txt`, `coverage.json`).
3. **GATE 1 — SCREEN before cutting** — read `<job>/flags.txt`: resolve every **[A]**
   over-long/collapsed token (retakes hide here), judge every **[B]** repeated phrase
   (retake vs intentional), note **[C]** gaps/audio-events. Confirm `<job>/coverage.json`
   (last word within ~2s of duration). See `references/scribe-collapse.md`.
4. **Author the EDL** — read `<job>/word_dump.txt` (and `prompts/author-edl.prompt.md` if present),
   then write `<job>/edl.json` per the brief below. To inspect any moment visually:
   `video-editing inspect --job-dir "<job>" --video vertical_src --start <a> --end <b>`.
   (Or let the CLI scaffold the prompt: `video-editing cut-bad-retakes --job-dir "<job>"`.)
5. **Render preview** — `video-editing render --job-dir "<job>" --mode preview`.
   Render **auto-runs the amplitude silence-trim pass** (Gate 3) first: it scans the audio
   with `silencedetect` and excises every sub-floor silence the transcript can't see —
   pauses baked *inside* a stretched word token, room tone, breaths — then trims silent
   range edges. Your authored `edl.json` is backed up to `edl.raw.json` and rewritten with
   the tightened ranges (more ranges, shorter total). It is **transcript-aware**: it never
   splits inside a single word token (spelled acronyms like "WWX", held vowels, numbers).
   Tune or disable: `video-editing desilence --job-dir "<job>"` (flags below) /
   `render … --no-desilence`.
6. **GATE 2 — CONTENT-VERIFY** — `video-editing verify --job-dir "<job>" --json`.
   It re-transcribes the preview at every join (healing OFF) and flags duplicated phrases
   across a join (= a retake left in) and over-long gaps (= dead air). Adjudicate with
   `video-editing inspect --job-dir "<job>" --join <i>`. See `references/adversarial-review.md`.
7. **Fix → re-render → re-verify** until `verify` is clean (`references/fix` rules apply).
8. **Final** — `video-editing render --job-dir "<job>" --mode final`.

## The standing editing brief (apply exactly)

- Output **VERTICAL 9:16**, **CLEAN CUTS ONLY** (`grade:"none"`, no subtitles, no overlays).
- **CUT** bad retakes / false starts / cut-off / stuttered words — keep the **LAST complete take**,
  BUT keep the **CLEANEST** take when an earlier take is clean and a later one stutters.
- **TIGHTEN every pause** to ~0.12–0.14s of air (remove dead air). The render's amplitude
  pass (Gate 3) enforces this automatically — it cuts **every silence below the dB floor**
  (default −36 dB, pauses ≥0.14s, leaving ~0.09s air). This catches the silences the
  transcript misses (dead air baked inside a long word token, room tone, breaths). It runs
  on every render; you don't hand-tighten silences in the EDL.
- **CUT ALL SILENCES below the dB floor** — for a more/less aggressive cut, tune the pass:
  `video-editing desilence --job-dir "<job>" --noise-db -39 --min-silence 0.30` (gentler) or
  `--noise-db -34 --min-silence 0.12 --lead-pad 0.03 --trail-pad 0.04` (harder). Re-render +
  re-verify after. To start clean, restore `cp edl.raw.json edl.json` then re-run.
- **COLD-OPEN** on the strongest hook (trim weak/dead intro).
- **TRIM outro dead air** (trailing breaths/ambient/silence/device noise).
- **KEEP intentional rhetorical repeats.**
- Never cut inside a word — snap every edge to a transcript word boundary; pad cut edges
  30–200ms (lead ~0.05s, trail ~0.08s; ~0.03–0.05s at retake joins, staying inside the silence).
  The renderer adds 30ms audio fades, so tight pads are safe.
- Hebrew/RTL: read the transcript word-by-word with explicit timestamps to avoid reorder confusion.

## The gates

1. **SCREEN before cutting** — Scribe hides retakes in over-long/dropped tokens; always read `flags.txt` first.
2. **DESILENCE on render** (automatic) — the amplitude pass cuts every sub-dB-floor silence the
   transcript can't see and trims silent edges. Transcript-aware (won't split spelled acronyms).
3. **CONTENT-VERIFY after preview** — `video-editing verify` re-transcribes joins; a waveform/visual check
   alone does NOT catch a retake left in (abandoned take + keeper render as one continuous clean span).
   A **≥3-word** duplicate across a join = a real retake left in → **error**, fix it. A bare **2-word**
   repeat = **warning**: adjudicate with `inspect` — it is usually a spelled acronym / number / emphasis
   (e.g. "WW" → "דבל יו דבל יו"), not a retake, and is safe to keep.

## EDL format

```json
{
  "version": 1,
  "sources": {"SRC": "<job>/vertical_src.mp4"},
  "grade": "none",
  "ranges": [
    {"source": "SRC", "start": 11.35, "end": 14.12, "beat": "HOOK", "quote": "…", "reason": "cleanest take; stops before the slip at 38.46"}
  ],
  "overlays": [],
  "total_duration_s": 51.33
}
```

Ranges are KEEP segments in source seconds; the gaps between them are the cuts. A retake range
must END at the **last clean word before the abandoned attempt**. `total_duration_s` = Σ(end−start).

## Timeline v2 (preferred when present)

When the CLI exposes a `timeline compile` verb (run `video-editing timeline --help`),
**author `<job>/timeline.json` instead of hand-writing `edl.json`.** timeline.json is the
single authored artifact: cuts (keep-ranges) plus optional **captions, zooms, overlays, and
audio blocks**. The compiler is compiler-grade — it validates, resolves phrase anchors to exact
source seconds, and emits `resolved/plan.json`, `resolved/diagnostics.json`, and a legacy
`edl.json`.

**Prerequisite — bake the vertical master first.** `timeline compile` probes
`<job>/vertical_src.mp4`, and `transcribe` alone does NOT create it: after transcribing, run
`video-editing cut-bad-retakes --job-dir "<job>" --render none` once (it bakes the vertical
master AND writes the screening artifacts the SCREEN gate needs anyway). Skipping this makes
`timeline compile` fail with a raw ffprobe "No such file or directory" — that error means run
the bake, not that your timeline is wrong.

Loop until clean:

1. Author/patch `<job>/timeline.json`.
2. `video-editing timeline compile --job-dir "<job>" --json` — read the diagnostics and iterate
   until it exits **0**:
   - **exit 2 = validation failed** (schema / semantic / capability) — fix the reported `path`s.
   - **exit 3 = ambiguous anchor** — disambiguate with
     `video-editing timeline resolve --phrase "…" --near <t>` (or `--occurrence <n>`), then bake the
     resolved seconds / occurrence into the anchor and re-compile.
3. **Never hand-edit anything under `resolved/`** — it is generated; re-compile instead.
4. compile emits the legacy `edl.json`, so the **render → desilence → verify → final** steps are
   **unchanged** — run them exactly as in the workflow above.

**Phrase → seconds ONLY via `timeline resolve`.** Never eyeball `word_dump.txt` for anchor
times — `resolve` returns word-accurate `start`/`end` and flags ambiguity so you disambiguate
deliberately (`--near <srcSeconds>` or `--occurrence <n>`).

Captions: enable with `video-editing timeline captions generate --job-dir "<job>" [--preset grouped]`
— cues are **derived at compile**, you don't hand-write cue timings. The default preset `grouped`
puts up to 4 words in each cue, starting a new cue on long word gaps (>0.6s), cut joins (a cue
never spans a join), and sentence-ending `.` `?` `!` (same characters in Hebrew); `opus-karaoke`
is the word-by-word opt-in. Timing/grouping knobs live in `captions.style` in timeline.json:
`maxWords` (default 4), `leadS` (default 0.08 — each cue appears that many seconds early, clamped
to the previous cue / segment start), `holdS` (default 0.9 — each cue holds on screen up to that
long past its last word to bridge the gap to the next cue, killing blink). **Word styling**: an
`op:"style"` caption override colors and/or emphasizes the anchored words — e.g. "make word X red"
is `{"op":"style","anchor":{"type":"words","source":"SRC","phrase":"X"},"color":"#ff5c5c"}`;
`emphasis:"pop"|"highlight"` animates the word at its onset. `captions.style.karaoke: true`
adds a per-word highlight sweep inside grouped cues (staged). Validate anytime with
`video-editing timeline validate --job-dir "<job>" --json`.

### Authoring zooms

A zoom is a punch-in on `rect` (unit-canvas crop, `x+w<=1`, `y+h<=1`) at `scale`
(`1 < scale <= 3`), anchored to a phrase or source span. Static by default (hard in/out).
For a **ramp**, add `to: {scale?, rect?}` — the zoom animates from the base rect/scale to
`to` over the zoom window, shaped by `ease: {in, out}` (both > 0 → smooth in-out). Zooms
resolve onCut like everything else (`clip` default). Full examples:
`references/timeline-v2-vocabulary.md`.

### Authoring overlays & assets

- Text: `kind:"text"` (positioned) / `kind:"title-card"` (full-frame card) + `text`.
- **Image/sticker**: `kind:"sticker"` (or `"image"`) + `asset: {ref: "assets/logo.png"}` —
  `ref` is a **job-dir-relative path**; keep files under `<job>/assets/`. Position with
  `layout.x/y` (center), size with `size.w/h` (canvas fractions; sticker default ~0.25 of
  width), stack with `z` (0–50). Animate with `enter`/`exit`
  (`{preset: fade|pop|slide-up|slide-down|scale-in, durationS ≤ 2}`).
- A referenced file that does not exist fails `timeline compile` with `asset-missing` —
  put the file in place first.
- On `review push --timeline` / `--complete`, referenced assets are **uploaded to
  clipready automatically** (sha256-cached in `<job>/review/assets.json`, so unchanged
  files are never re-uploaded); the push payload carries the `{ref: asset_id}` manifest.
- `compose` copies referenced assets into `<job>/composition/media/` and renders them.

### Delivering captions/zooms

The ffmpeg `render --mode final` path is **CUT-ONLY** — it never draws the compiled
captions/zooms/overlays from `resolved/plan.json`. Deliver those visual layers with either:

- **`video-editing compose --job-dir "<job>" --render`** — local and free. Builds a declarative
  HyperFrames composition in `<job>/composition/` (compressed proxy of `vertical_src.mp4`,
  the CONTINUOUS cut audio extracted from `preview.mp4`, and `index.html`) and renders it with
  the `hyperframes` CLI (needs it on PATH: `npm install -g hyperframes`). Prerequisites:
  `resolved/plan.json` (run `timeline compile`) and `preview.mp4` (run `render --mode preview`).
  Without `--render` it only builds the composition and prints the exact `hyperframes render`
  command to run. Output defaults to `<job>/composed.mp4`.
- **The ClipReady cloud render** (`cloud render`), which renders the same plan server-side
  (requires a ClipReady account + API key — see the review-round section below).

Do **NOT** fall back to the legacy `render --build-subtitles` ASS burn for timeline-v2 jobs —
it requires a libass ffmpeg and renders the old 2-word subtitle style, not the compiled cues.

### Working a ClipReady review round

> Cloud features require a ClipReady account + API key (`CLIPREADY_API_BASE`, `CLIPREADY_API_KEY`).

When the job is a clipready review round (you have an API base + credential + video id), the
CLI syncs the round for you — no hand-rolled curl:

1. `video-editing review pull --job-dir "<job>"` — writes `<job>/review/round.json` (run id,
   requested changes) and `<job>/review/instruction.md`, and hydrates `<job>/timeline.json`
   from the server when you don't already have one (it never overwrites a local timeline).
   Add `--source` if the source video isn't local yet.
2. `video-editing review push --stage edl --job-dir "<job>"` when you start working the round.
3. Edit `timeline.json` per the instruction, then
   `video-editing review push --timeline --job-dir "<job>"` after each meaningful change — it
   compiles locally (diagnostics + exit 2 if not clean) and the review page updates live.
4. Preview/verify locally exactly as in the workflow above (render preview, CONTENT-VERIFY, QC).
5. `video-editing review push --complete --duration <s> --outcomes outcomes.json --job-dir "<job>"`
   — compiles fresh (must be clean) and delivers the terminal payload (edl + timeline + plan).
   `outcomes.json` is `[{"change_id": "chg_…", "ok": true, "note": "…"}]`, one entry per
   `[chg_…]` marker in the instruction; omit `--outcomes` when there are no markers.

## References

- `references/cli-reference.md` — every subcommand + flag (incl. `verify`, `inspect`).
- `references/timeline-v2-vocabulary.md` — the full timeline.json v2 vocabulary, one JSON
  example per feature (word styling, karaoke, zoom ramps, stickers, enter/exit, grade).
- `references/editing-brief.md` — the full EDL-authoring procedure.
- `references/hard-rules.md` — production-correctness rules + anti-patterns.
- `references/scribe-collapse.md` — why retakes hide in the transcript (Gate 1).
- `references/adversarial-review.md` — the content-verify pass (Gate 2).
