---
name: video-editing
description: >-
  Cloud agent workflow for editing a talking-head video into a tight, clean
  cut with the `video-editing` CLI + the ClipReady cloud. The CLI is a PURE
  API client — requirements are Node ≥ 20 and a ClipReady API key
  (`CLIPREADY_API_KEY`) only; no ffmpeg, no ffprobe, no local rendering
  tools. The source uploads once at init; transcription, editorial
  screening, timeline compilation, rendering, verification, and QA visuals
  all run server-side; the editorial brief arrives from the server as pulled
  `prompts/*.md`. Use whenever the user drops a talking-head clip / reel /
  vlog / interview and wants it edited or tightened — "edit this reel",
  "cut the bad retakes", "clean up this recording", "tighten this",
  "add captions", or just a video path with an editing ask — or when
  working a ClipReady review round. Handles Hebrew/RTL.
---

# Video Editing (ClipReady cloud — pure API client)

Turn a raw talking-head recording into a tight, clean cut. The
**`video-editing`** CLI (v3) is a pure API client for the ClipReady cloud: the
cloud owns EVERYTHING executable — probing, transcription, screening,
editorial prompts, timeline compilation, rendering, verification, and QA
visuals. Your machine only streams the source up once and pulls small
artifacts back. **You** read the pulled transcript + brief and author
`timeline.json`; the CLI does the rest over HTTP.

## Install & auth

```bash
npm install -g video-editing                                  # the CLI (v3)
npx skills add ItayElgazar/clipready --skill video-editing    # this skill
```

Requirements — this is the FULL list:

- **Node ≥ 20**
- **`CLIPREADY_API_KEY`** — from the ClipReady settings page
- **`CLIPREADY_API_BASE`** — the ClipReady API base URL

**No ffmpeg. No ffprobe. No hyperframes. No ElevenLabs key.** Nothing runs on
the local machine except the CLI itself — do not install media tools, and do
not set or ask for `ELEVENLABS_API_KEY`.

**Every command fails without the API key.** If a verb errors with a
missing-key message, stop and ask the user for their ClipReady API key;
never work around it.

## HARD RULE — execution boundary (read first)

- Use **only** the `video-editing` CLI (`video-editing <subcommand>`). Run
  `video-editing --help` to see commands.
- **NEVER** run ffmpeg/ffprobe/transcription by hand — there is simply no
  local media in this workflow at all; every frame lives in the cloud. If you
  have a memory of local renders, `compose`, `--master`, or "extract the WAV
  first", that memory is from a retired pipeline — discard it.
- If a capability seems missing, STOP and report it as a CLI gap. Do not
  reach around the CLI.
- Your only hand-authored artifact is `<job>/timeline.json`.
- **Captions are timeline-authored and rendered by the cloud renderer —
  NEVER ffmpeg subtitle filters, libass, `.srt` sidecars, or `drawtext`.**

## The workflow

1. **Init** — `video-editing init --source "<path>"` (add `--job-dir "<dir>"`
   to choose the working dir). ONE command does everything through
   transcription: it streams the source up (chunked, retried, 2GB-safe),
   creates the cloud job, runs server-side ingest (probe + transcription +
   editorial screening), and pulls the results into the job dir:
   `transcripts/`, `word_dump.txt`, `takes_packed.md`, `flags.txt`,
   `coverage.json`, `probe.json` — **and `prompts/author-edl.prompt.md`**,
   your editorial brief. Writes `<job>/job.json` (the `video_id` every later
   verb uses) and prints the video's web link. There is no separate
   transcribe step — `transcribe` exists only to re-pull/re-run later.
   Filenames with apostrophes, spaces, or unicode are safe — the internal
   stem is sanitized automatically; never rename the user's file.
2. **READ the pulled `prompts/*.md` and follow them** — they are your
   editorial brief (what to cut, what to keep, the screening gate, the pad
   and boundary rules). They are authoritative and may change between runs;
   never edit them, never substitute a remembered brief.
3. **Author `<job>/timeline.json`** per the pulled brief plus
   `references/timeline-v2-vocabulary.md` (cut ranges, captions, zooms,
   overlays, audio). The **sources KEY must equal the transcript stem** (the
   filename of `transcripts/<stem>.json`) — captions and resolve bind
   through it. Resolve phrase → seconds ONLY with
   `video-editing timeline resolve --phrase "…" [--occurrence <n>]`
   — never eyeball `word_dump.txt`. Sanity-check locally anytime with
   `video-editing timeline validate --job-dir "<job>" --json`.
4. **Compile** — `video-editing timeline compile --job-dir "<job>" --json`.
   Compiles in the cloud; writes `resolved/plan.json` +
   `resolved/diagnostics.json` and prints the `View & comment` watch link —
   the user can already watch/comment on the compiled cut. Read the
   diagnostics and fix until it exits `0` (`2` = validation failed — fix the
   reported paths; `3` = ambiguous anchor — disambiguate with
   `timeline resolve`). Never hand-edit `resolved/`.
5. **Render the preview** — `video-editing render --job-dir "<job>"`
   (cloud render, preview mode, waits and downloads `<job>/preview.mp4` by
   default; it works right after `timeline compile`). Share the printed
   watch link with the user.
6. **Verify** — `video-editing verify --job-dir "<job>" --json`. Fully
   server-side verification of the rendered preview (catches retakes left in
   across a join and dead air a visual check misses); nothing is uploaded.
   Can run while the user reviews. Fix `timeline.json`, re-compile,
   re-render, re-verify until clean; cap at ~3 fix passes and surface
   remaining findings rather than looping.
7. **QA looks** — spot-check any window with server-rendered visuals:
   `video-editing qa frames --start <a> --end <b> --job-dir "<job>"`
   (filmstrip PNG, SOURCE seconds), or plan-addressed:
   `qa frames --join <i>` (the two sides of plan join i → `-a`/`-b` PNGs)
   and `qa frames --out-start <a> --out-end <b>` (OUTPUT seconds mapped
   through the compiled plan). `video-editing qa waveform --start <a> --end <b>`
   adds a waveform PNG + word-timing sidecar JSON. Read the PNGs to
   adjudicate.
8. **Final** — `video-editing render --mode final --job-dir "<job>"`.

## Your editorial brief arrives from the cloud

The deep editorial brief — retake/keep rules, pause tightening, cold-open,
screening and verification procedure — is **not in this skill**. It is pulled
from the server into `<job>/prompts/*.md` by `init` (and refreshed by
`files pull`). Old references in this skill (`editing-brief.md`,
`hard-rules.md`, `adversarial-review.md`, `scribe-collapse.md`) are stubs
pointing there. If `prompts/author-edl.prompt.md` is missing, run
`video-editing files pull --job-dir "<job>"` — do not improvise a brief.

## Known caveats

- **`verify` over-triggers on thematically narrow monologues.** The
  duplicated-phrase join check flags legitimately repeated wording as
  "retake left in". If EVERY join comes back flagged — including plain
  pause-tightening joins that obviously aren't retakes — adjudicate each
  finding against `flags.txt` (the screening output) instead of blindly
  recutting; only recut joins that flags.txt corroborates as real retakes.
- **Caption mistranscriptions** are fixed with a captions override in
  `timeline.json` (anchored on the MIS-transcribed text — see
  `references/timeline-v2-vocabulary.md`), then recompile + re-render.
  Never with subtitle files or filters.

## File sync

`video-editing files pull --job-dir "<job>" [--only <name,name>]` fetches the
cloud job's artifacts into the job dir preserving relative paths (transcripts,
`prompts/*.md`, `resolved/plan.json`, `timeline.json`, …);
`video-editing files push --job-dir "<job>" <paths…>` uploads local artifacts
(e.g. a hand-authored `timeline.json`, referenced assets) back to the job. Use
`pull` to recover state on a fresh machine — the cloud job is the source of
truth.

## Working a ClipReady review round

When the job is a clipready review round (the video has an active round):

1. `video-editing review pull --job-dir "<job>"` — writes
   `<job>/review/round.json` (round/run ids, requested changes) and
   `<job>/review/instruction.md`; hydrates `<job>/timeline.json` from the
   server only when you don't already have one (never overwrites yours).
2. Edit `timeline.json` per the instruction, then
   `video-editing review push --timeline --job-dir "<job>"` after each
   meaningful change — it compiles in the cloud (diagnostics; exit `2` if
   not clean) and the review page updates live. Referenced assets (stickers,
   audio beds) upload automatically, sha256-cached.
3. Render + verify exactly as in the workflow above.
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

- `references/cli-reference.md` — every v3 subcommand + flag.
- `references/timeline-v2-vocabulary.md` — the full `timeline.json` v2
  vocabulary, one JSON example per feature (captions + word styling, karaoke,
  zoom ramps, stickers, enter/exit, audio beds, grade).
- `references/editing-brief.md`, `references/hard-rules.md`,
  `references/scribe-collapse.md`, `references/adversarial-review.md` —
  stubs; the live content is server-pulled into `<job>/prompts/*.md`.
