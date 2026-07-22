---
name: video-editing
description: >-
  Cloud-first agent workflow for editing a talking-head video into a tight,
  clean cut with the `video-editing` CLI + the ClipReady cloud. Requires a
  ClipReady API key (`CLIPREADY_API_KEY`) — every command talks to the cloud.
  Transcription and editorial screening run server-side (no ElevenLabs key
  needed); the editorial brief arrives from the server as pulled
  `prompts/*.md`; the timeline compiles in the cloud; rendering and QA
  visuals run locally on your machine (ffmpeg + hyperframes). Use whenever
  the user drops a talking-head clip / reel / vlog / interview and wants it
  edited or tightened — "edit this reel", "cut the bad retakes", "clean up
  this recording", "tighten this", "add captions", or just a video path with
  an editing ask — or when working a ClipReady review round. Handles
  Hebrew/RTL.
---

# Video Editing (ClipReady cloud + local render)

Turn a raw talking-head recording into a tight, clean cut. The **`video-editing`**
CLI is a thin client for the ClipReady cloud: the cloud owns transcription,
screening, editorial prompts, timeline compilation, and verification; your
machine owns rendering and QA visuals. **You** read the pulled transcript +
brief and author `timeline.json`; the CLI does everything executable.

## Install & auth

```bash
npm install -g video-editing                                  # the CLI (v2)
npx skills add ItayElgazar/clipready --skill video-editing    # this skill
```

Requirements:

- **Node ≥ 20**
- **ffmpeg**/**ffprobe** on PATH (local render + QA visuals)
- **`CLIPREADY_API_KEY`** — from the ClipReady settings page
- **`CLIPREADY_API_BASE`** — the ClipReady API base URL

**No ElevenLabs key.** Transcription runs in the ClipReady cloud with
ClipReady's own transcription — do not set or ask for `ELEVENLABS_API_KEY`.

**Every command fails without the API key.** If a verb errors with a
missing-key message, stop and ask the user for their ClipReady API key;
never work around it.

## HARD RULE — execution boundary (read first)

- Use **only** the `video-editing` CLI (`video-editing <subcommand>`). Run
  `video-editing --help` to see commands.
- **NEVER** run ffmpeg/ffprobe/transcription by hand — the CLI owns all execution.
- If a capability seems missing, STOP and report it as a CLI gap. Do not reach around the CLI.
- Your only hand-authored artifact is `<job>/timeline.json`.
- **Captions are timeline-authored and rendered by `compose --render`
  (hyperframes) — NEVER ffmpeg subtitle filters, libass, `.srt` sidecars, or
  `drawtext`.** If you have a memory about "ffmpeg missing libass",
  "brew reinstall ffmpeg for captions", or a `--build-subtitles` verb, that
  memory is from a retired pipeline — discard it and update your notes.

## The workflow

1. **Init** — `video-editing init --video "<path>"` (add `--job-dir "<dir>"` to
   choose the working dir). Uploads the source and creates the cloud job;
   writes `<job>/job.json` with the `video_id` every later verb uses.
2. **Transcribe** — `video-editing transcribe --job-dir "<job>"`. Uploads the
   audio; the cloud transcribes **and screens** it, then the CLI pulls the
   results into the job dir: `transcripts/`, `word_dump.txt`,
   `takes_packed.md`, `flags.txt`, `coverage.json` — **and
   `prompts/author-edl.prompt.md`**, your editorial brief.
3. **READ the pulled `prompts/*.md` and follow them** — they are your
   editorial brief (what to cut, what to keep, the screening gate, the pad
   and boundary rules). They are authoritative and may change between runs;
   never edit them, never substitute a remembered brief.
4. **Author `<job>/timeline.json`** per the pulled brief plus
   `references/timeline-v2-vocabulary.md` (cut ranges, captions, zooms,
   overlays, audio). Resolve phrase → seconds ONLY with
   `video-editing timeline resolve --phrase "…" [--near <t> | --occurrence <n>]`
   — never eyeball `word_dump.txt`. Sanity-check locally anytime with
   `video-editing timeline validate --job-dir "<job>" --json`.
5. **Compile** — `video-editing timeline compile --job-dir "<job>" --json`.
   Compiles in the cloud; writes `resolved/plan.json` +
   `resolved/diagnostics.json`. Read the diagnostics and fix until it exits
   `0` (`2` = validation failed — fix the reported paths; `3` = ambiguous
   anchor — disambiguate with `timeline resolve`). Never hand-edit `resolved/`.
6. **Render the preview** — `video-editing compose --job-dir "<job>"` builds
   the local HyperFrames composition in `<job>/composition/` (with
   `index.html`), then render it locally with the `hyperframes` CLI
   (`npm install -g hyperframes`; `compose` prints the exact command, or pass
   `--render`). No hyperframes / no local horsepower? Use
   `video-editing cloud render --job-dir "<job>" --mode preview --wait` instead.
7. **Verify** — `video-editing verify --job-dir "<job>" --json`. Cloud
   verification of the rendered preview (catches retakes left in across a
   join and dead air a visual check misses). Fix `timeline.json`, re-compile,
   re-render, re-verify until clean; cap at ~3 fix passes and surface
   remaining findings rather than looping.
8. **QA looks** — spot-check any window locally:
   `video-editing qa frames --range <a>-<b> --job-dir "<job>"` (filmstrip PNG)
   and `video-editing qa waveform --range <a>-<b> --job-dir "<job>"`
   (waveform PNG + word-timing sidecar JSON). Local ffmpeg — free and fast;
   Read the PNGs to adjudicate.
9. **Final** — `video-editing cloud render --job-dir "<job>" --mode final --wait`
   (or render the composition locally at final quality).

## Your editorial brief arrives from the cloud

The deep editorial brief — retake/keep rules, pause tightening, cold-open,
screening and verification procedure — is **not in this skill**. It is pulled
from the server into `<job>/prompts/*.md` by `transcribe` (and refreshed by
`files pull`). Old references in this skill (`editing-brief.md`,
`hard-rules.md`, `adversarial-review.md`, `scribe-collapse.md`) are stubs
pointing there. If `prompts/author-edl.prompt.md` is missing, run
`video-editing files pull --job-dir "<job>"` — do not improvise a brief.

## Vertical master

`init --video` uploads the source as-is. When the deliverable is a vertical
9:16 reel from a non-vertical source, pass `--master` to `init` so the
vertical master is baked before upload (needs local ffmpeg). Check
`video-editing init --help` for the current flag surface.

## File sync

`video-editing files pull --job-dir "<job>" [--only <name,name>]` fetches the
cloud job's artifacts into the job dir preserving relative paths (transcripts,
`prompts/*.md`, `resolved/plan.json`, `timeline.json`, …);
`video-editing files push --job-dir "<job>"` uploads local artifacts (e.g. a
hand-authored `timeline.json`, referenced assets) back to the job. Use `pull`
to recover state on a fresh machine — the cloud job is the source of truth.

## Working a ClipReady review round

When the job is a clipready review round (the video has an active round):

1. `video-editing review pull --job-dir "<job>"` — writes
   `<job>/review/round.json` (round/run ids, requested changes) and
   `<job>/review/instruction.md`; hydrates `<job>/timeline.json` from the
   server only when you don't already have one (never overwrites yours).
   Add `--source` if the source video isn't local yet.
2. Edit `timeline.json` per the instruction, then
   `video-editing review push --timeline --job-dir "<job>"` after each
   meaningful change — it compiles (diagnostics; exit `2` if not clean) and
   the review page updates live. Referenced assets (stickers, audio beds)
   upload automatically, sha256-cached.
3. Preview + verify exactly as in the workflow above.
4. `video-editing review push --complete --duration <s> --outcomes outcomes.json --job-dir "<job>"`
   — compiles fresh (must be clean) and delivers the terminal payload.
   `outcomes.json` is `[{"change_id": "chg_…", "ok": true, "note": "…"}]`,
   one entry per `[chg_…]` marker in the instruction; omit `--outcomes` when
   there are no markers.

## Hebrew / RTL

Hebrew/RTL sources are fully supported. Read the transcript word-by-word with
explicit timestamps to avoid visual-reorder confusion; anchor phrases via
`timeline resolve` exactly as with LTR text. Sentence-ending punctuation for
caption grouping is the same characters in Hebrew.

## References

- `references/cli-reference.md` — every v2 subcommand + flag.
- `references/timeline-v2-vocabulary.md` — the full `timeline.json` v2
  vocabulary, one JSON example per feature (captions + word styling, karaoke,
  zoom ramps, stickers, enter/exit, audio beds, grade).
- `references/editing-brief.md`, `references/hard-rules.md`,
  `references/scribe-collapse.md`, `references/adversarial-review.md` —
  stubs; the live content is server-pulled into `<job>/prompts/*.md`.
