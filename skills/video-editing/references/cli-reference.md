# `video-editing` CLI reference

All execution goes through this CLI. Quote paths (they contain spaces).

## Pipeline commands

- **`init-job --source <video> [--job-dir <dir>] [--force]`**
  Probe the source and create the job dir + `source.json`. Always pass `--job-dir`
  explicitly on the sandbox (e.g. `/vercel/sandbox/jobs/<job_id>`) — there is no
  `~/Downloads` to default into. Prints the job dir.

- **`transcribe (--source <video> | --job-dir <dir>) [--language <iso>] [--num-speakers <n>] [--no-heal] [--force] [--api-key <key>]`**
  Word-level ElevenLabs Scribe transcription (auto-heals collapsed/dropped spans)
  + screening artifacts: `flags.txt`, `takes_packed.md`, `word_dump.txt`, `coverage.json`.
  Cached per source — `--force` re-transcribes (paid). `--no-heal` disables healing.

- **`cut-silences (--source|--job-dir) [--min-gap 0.30] [--target-gap 0.13] [--lead-pad 0.05] [--trail-pad 0.08] [--render preview|none] [--force-transcribe]`**
  Prepare a silence-only EDL-authoring prompt (`prompts/cut-silences.prompt.md`);
  renders a preview if `edl.json` already exists.

- **`cut-bad-retakes (--source|--job-dir) [--with-sound-waves] [--quality-review] [--render preview|none] [--language] [--num-speakers] [--force-transcribe]`**
  Prepare the full author-edl prompt (`prompts/author-edl.prompt.md`). `--with-sound-waves`
  writes join QC PNGs (needs `edl.json` + `preview.mp4`). `--quality-review` writes the review prompt.

- **`desilence --job-dir <dir> [--noise-db -36] [--min-silence 0.14] [--lead-pad 0.04] [--trail-pad 0.05] [--render preview|none] [--force]`**  ← Gate 3
  Amplitude silence-trim. Runs `silencedetect=noise=<db>dB` over each EDL source, then for every
  keep-range **excises internal silences ≥ `--min-silence`** (leaving `lead-pad + trail-pad` of air)
  and **trims silent leading/trailing edges**. Catches silence the transcript can't see (dead air
  baked inside a stretched word token, room tone, breaths). **Transcript-aware** — never splits inside
  a single word token (spelled acronyms, held vowels, numbers). Backs the original EDL up to
  `edl.raw.json`, rewrites `edl.json`, marks it `desilenced`. Idempotent. Higher `--noise-db`
  (e.g. −34) + lower `--min-silence` = more aggressive.
  **Runs automatically at the start of every `render`** (skip with `render … --no-desilence`); the
  standalone command is for tuning + inspecting the result before rendering.

- **`render --job-dir <dir> --mode preview|draft|final [--build-subtitles] [--no-subtitles] [--no-loudnorm] [--no-desilence]`**
  Render the EDL: **desilence (unless `--no-desilence`)** → per-segment extract → lossless concat →
  (optional subs) → loudnorm (−14 LUFS / −1 dBTP / LRA 11). preview=CRF22/1080p, draft=CRF28/720p,
  final=CRF20/1080p. Writes `preview.mp4`/`draft.mp4`/`final.mp4` + `cut_boundaries.json`.

## QC / verification commands

- **`verify --job-dir <dir> [--full] [--window 6] [--gap 0.6] [--json] [--api-key <key>]`**  ← Gate 4 (content)
  Re-transcribes the rendered preview with healing OFF and reports, per join,
  **duplicated phrases** and **over-long gaps** (dead air not tightened → warning/error).
  A duplicate of **≥3 words** = a retake left in → **error**. A bare **2-word** repeat = **warning**
  (spelled acronym / number / emphasis like "WW" → "דבל יו דבל יו" — adjudicate with `inspect`, usually safe).
  Default: per-join ~6s windows from `cut_boundaries.json`; `--full` re-transcribes the whole preview and
  also checks head/tail dead air. Writes `verify.json`; exit code 2 if `needs-fixes`.

- **`inspect --job-dir <dir> (--join <i> | --start <s> --end <s>) [--video preview|vertical_src] [--transcript <path>] [-o <png>] [--n-frames 10]`**
  Filmstrip + waveform + word-label PNG for a window or a cut-boundary index.
  `timeline` is an alias. Use it to adjudicate a verify finding or pick a cut edge.

## TimelineV2 commands (`timeline.json` v2 — preferred when present)

Exit codes for this group: `0` ok, `1` error, `2` validation failed, `3` ambiguous anchor.

- **`timeline validate --job-dir <dir> [--json]`**
  Schema + semantic + capability validation of `<job>/timeline.json`. Prints errors/warnings and
  the capabilities the doc `uses`. Exit `2` if invalid.

- **`timeline compile --job-dir <dir> [--json]`**
  Compile `timeline.json` → `resolved/plan.json` + `resolved/diagnostics.json` + **legacy `edl.json`**
  (so render/verify are unchanged). Iterate until it exits `0`. Exit `2` = validation failed,
  `3` = an ambiguous phrase anchor (disambiguate with `timeline resolve`). Never hand-edit `resolved/`.

- **`timeline resolve --job-dir <dir> --phrase <text> [--near <srcSeconds>] [--occurrence <n>] [--source <name>] [--json]`**
  Resolve a transcript phrase to **exact source seconds** (normalized, word-accurate match). The
  ONLY way to turn a phrase into an anchor time — never eyeball `word_dump.txt`. Multiple hits →
  exit `3` with per-occurrence context; pick one with `--near` (nearest source time) or
  `--occurrence` (1-based). Not found → fuzzy suggestions, exit `1`.

- **`timeline captions generate --job-dir <dir> [--preset grouped|opus-karaoke] [--json]`**
  Enable captions in `timeline.json` (preset only — cues are **derived at compile**, not hand-authored).
  Default preset `grouped`: up to `captions.style.maxWords` (default 4) words per cue, breaking on
  word gaps >0.6s, cut joins, and sentence-ending `.` `?` `!`; `opus-karaoke` is the word-by-word
  opt-in. Style knobs in `captions.style`: `maxWords` (4), `leadS` (0.08 — cues display that early,
  clamped), `holdS` (0.9 — cues hold to bridge the gap to the next cue). Reports the derived cue
  count via a dry compile.

- **`compose --job-dir <dir> [--render] [--quality draft|standard|high] [--fps 30] [--out <path>] [--proxy-crf 23] [--json]`**
  Build a local HyperFrames composition from `resolved/plan.json` (cut + captions/zooms/overlays)
  in `<job>/composition/`: `media/source.mp4` (compressed proxy of `vertical_src.mp4`, libx264
  CRF `--proxy-crf`, skipped when up to date), `media/audio.m4a` (the CONTINUOUS cut audio
  extracted from `preview.mp4` — the QC preview must exist), referenced image/sticker assets
  copied under `media/` (preserving their job-dir-relative paths), and `index.html` (with a
  generated GSAP timeline when the plan uses animated vocabulary). Without `--render`,
  prints the composition dir + the exact `hyperframes render` command. With `--render`, runs
  `hyperframes render --quality <q> --fps <n> --output <out>` (default `<job>/composed.mp4`)
  in the composition dir — requires the `hyperframes` CLI on PATH (`npm install -g hyperframes`).
  `--json` prints `{compositionDir, entry, out?, segments, cues, zooms, overlays}`. This is the
  LOCAL delivery path for compiled captions/zooms — the ffmpeg `render` is cut-only; never use
  `--build-subtitles` for timeline-v2 jobs.

## Cloud thin-client commands (clipready render/QA/artifact API)

Every cloud verb resolves credentials the same way and supports `--json` (machine shape;
exit `0` ok, `1` error). Resolution precedence:

- **base**: `--api-base` → env `CLIPREADY_API_BASE` → env `API_BASE`
- **credential** (sent as `Authorization: Bearer <credential>`): `--api-key` → env `CLIPREADY_API_KEY` → env `RUN_TOKEN`
- **video id**: `--video-id` → env `VIDEO_ID`

A missing base / credential / (required) video id is a clear error → exit `1`.

- **`cloud render --job-dir <dir> --mode preview|final [--wait] [--poll-interval 10] [--out <path>] [--json]`**
  Submit a render. Without `--wait`: prints `render_id` and exits `0`. With `--wait`: polls status
  (max 30 min) until terminal; on `completed`, downloads the signed `video_url` to `--out`
  (default `<job>/preview.mp4` or `<job>/final.mp4`). `failed` → exit `1` with the error. Needs a video id.

- **`cloud status <render_id> [--json]`**
  One status GET: `pending|running|completed|failed` (+ `video_url` when completed). `failed` → exit `1`.

- **`qa frames --start <s> --end <s> [--cols <n>] [--interval <s>] [--job-dir <dir>] [--json]`**
  Server-render a filmstrip PNG for the window and download it to `<job>/qa/frames_<start>_<end>.png`
  (`./qa/` when no `--job-dir`). Prints the local path + grid metadata (Read the PNG to inspect). Needs a video id.

- **`qa waveform --start <s> --end <s> [--job-dir <dir>] [--json]`**
  Same, for a waveform: downloads `<job>/qa/waveform_<start>_<end>.png` and writes the word list to
  `<job>/qa/waveform_<start>_<end>.words.json`. Prints both paths. Needs a video id.

- **`job pull [--job-dir <dir>] [--only <name,name>] [--json]`**
  List the video's artifacts and fetch each (or the `--only` subset) into the job dir (default cwd),
  **preserving relative paths** (`resolved/plan.json` → `<job>/resolved/plan.json`). Prints a summary.
  `.json` artifacts are written pretty-printed; others verbatim. Needs a video id. Allowlisted names include
  `word_dump.txt`, `takes_packed.md`, `flags.txt`, `coverage.json`, `edl.json`, `verify.json`,
  `cut_boundaries.json`, `timeline.json`, `resolved/plan.json`, `resolved/diagnostics.json`.

- **`review pull [--job-dir <dir>] [--source] [--json]`**
  Fetch the clipready review session for the video into the job dir (default cwd): writes
  `<job>/review/round.json` (round/run ids, mode, status, requested changes) and
  `<job>/review/instruction.md` (the round's editorial instruction — or a "no active round" note).
  Writes `<job>/timeline.json` from the session ONLY when no local timeline.json exists (never
  overwrites yours). With `--source`, also downloads the source video (to the `source.json`
  filename, else `<job>/source.mp4`) when absent locally. Needs a video id.

- **`review push (--stage <type> [--message <text>] | --timeline | --complete --duration <s> [--outcomes <file>]) [--job-dir <dir>] [--run-id <id>] [--json]`**
  Push review-round events (exactly one mode per call; run id defaults to `review/round.json`
  from `review pull`). `--stage` posts a progress event
  (`provisioning|transcribing|edl|render_preview|qc|render_final|uploading|log`), optional
  `{message}` payload. `--timeline` compiles `<job>/timeline.json` locally (same sequence as
  `timeline compile`, needs ffmpeg) and pushes a `timeline_update` with the raw timeline JSON,
  the compiled plan, and its hash — the review page updates live. `--complete` compiles fresh
  (must be clean) and pushes the terminal `completed` event
  (`{edl, timeline, plan, duration_s, outcomes?}`; `--outcomes` is a JSON file of
  `[{change_id, ok, note?}]`, bare array or `{outcomes: [...]}`). Compile failures print
  diagnostics and exit `2` (nothing is pushed); cloud errors exit `1`. Needs a video id.
  Both `--timeline` and `--complete` **upload the timeline's referenced assets
  automatically** (image/sticker overlays, audio beds): each ref is sha256-cached in
  `<job>/review/assets.json` so unchanged files are never re-uploaded, and the push
  payload includes an `assets: {ref: asset_id}` manifest. A missing asset file fails the
  compile (`asset-missing`) before anything is uploaded.

- **`skill install [--target <dir>] [--json]`**
  Copy the bundled `video-editing` agent skill into `<target|~/.claude/skills>/video-editing`
  (overwriting). Prints what was installed.

## Utility

- **`--version`** — print the CLI version.
- **`status --job-dir <dir> [--json]`** — job state + artifact existence.
- **`review prompt --job-dir <dir> [--write-warnings]`** — write the review prompt;
  optionally convert `review.json` → `warnings.json`. (`--open-app` is not usable on the
  sandbox — it launches a local browser; skip it, there is no display.) This was the old
  top-level `review` verb — `review` is now a group (`prompt` / `pull` / `push`).
- **`config set-key <key>` / `config path`** — manage the ElevenLabs key
  (`~/.config/video-editing/.env`; on the sandbox the CLI reads env `ELEVENLABS_API_KEY`,
  no need to run `config set-key`).

## Typical session

```
video-editing init-job --source "source.mp4" --job-dir /vercel/sandbox/jobs/<job_id>
video-editing transcribe --job-dir /vercel/sandbox/jobs/<job_id>
# read flags.txt (Gate 1), author edl.json
video-editing render --job-dir /vercel/sandbox/jobs/<job_id> --mode preview
video-editing verify --job-dir /vercel/sandbox/jobs/<job_id> --json   # Gate 2
# fix edl.json as needed, re-render, re-verify
video-editing render --job-dir /vercel/sandbox/jobs/<job_id> --mode final
```
