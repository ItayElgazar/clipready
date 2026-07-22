# `video-editing` CLI reference (v2 — cloud-only)

All execution goes through this CLI. Quote paths (they contain spaces).

**Every verb requires ClipReady credentials.** Resolution:

- **base**: `--api-base` → env `CLIPREADY_API_BASE`
- **key** (sent as `Authorization: Bearer <key>`): `--api-key` → env `CLIPREADY_API_KEY`
- **video id**: `--video-id` → `<job>/job.json` (written by `init`) → env `VIDEO_ID`

A missing base or key is a clear error → non-zero exit, nothing runs. There is
**no ElevenLabs configuration** — transcription is server-side in the ClipReady
cloud.

Most verbs support `--json` for a machine-readable result. Exit codes: `0` ok,
`1` error (incl. missing credentials), `2` validation failed, `3` ambiguous
anchor (timeline group).

## Pipeline commands

- **`init --video <path> [--job-dir <dir>] [--master] [--force]`**
  Upload the source and create the cloud job. Writes the job dir (default: a
  new dir named for the clip) with `job.json` (`video_id`, api base) and
  `source.json`. `--master` bakes the vertical 9:16 master locally (ffmpeg)
  before upload, for non-vertical sources destined for a vertical deliverable.

- **`transcribe --job-dir <dir> [--language <iso>] [--num-speakers <n>] [--force]`**
  Upload the audio; the **cloud** transcribes and screens it, then the CLI
  pulls the results into the job dir: `transcripts/*.json`, `word_dump.txt`,
  `takes_packed.md`, `flags.txt`, `coverage.json`, and the editorial brief
  `prompts/author-edl.prompt.md` (plus any other `prompts/*.md` the server sends).
  Cached server-side per source; `--force` re-transcribes.

## Timeline commands (`timeline.json` v2)

- **`timeline validate --job-dir <dir> [--json]`**
  Local schema + semantic + capability validation of `<job>/timeline.json`.
  Exit `2` if invalid.

- **`timeline resolve --job-dir <dir> --phrase <text> [--near <srcSeconds>] [--occurrence <n>] [--source <name>] [--json]`**
  Resolve a transcript phrase to **exact source seconds** (word-accurate).
  The ONLY way to turn a phrase into an anchor time. Multiple hits → exit `3`
  with per-occurrence context; pick one with `--near` or `--occurrence`
  (1-based). Not found → fuzzy suggestions, exit `1`.

- **`timeline compile --job-dir <dir> [--json]`**
  **Cloud compile**: uploads `timeline.json`, the server validates, resolves
  anchors, derives caption cues, and returns `resolved/plan.json` +
  `resolved/diagnostics.json` (written into the job dir). Iterate until exit
  `0`. Exit `2` = validation failed (fix the reported `path`s), `3` =
  ambiguous anchor (disambiguate with `timeline resolve`). Never hand-edit
  anything under `resolved/`.

## Preview / render commands

- **`compose --job-dir <dir> [--render] [--quality draft|standard|high] [--fps 30] [--out <path>] [--json]`**
  Build the local HyperFrames composition from `resolved/plan.json` (cut +
  captions/zooms/overlays) in `<job>/composition/` — media proxies, referenced
  assets, and `index.html`. Without `--render`, prints the composition dir +
  the exact `hyperframes render` command to run. With `--render`, runs it
  (requires the `hyperframes` CLI on PATH: `npm install -g hyperframes`);
  default output `<job>/preview.mp4` (the render IS the preview in the cloud-first flow). This is the LOCAL delivery path.

- **`cloud render --job-dir <dir> --mode preview|final [--wait] [--poll-interval 10] [--out <path>] [--json]`**
  Submit a server-side render of the compiled plan. Without `--wait`: prints
  `render_id` and exits `0`. With `--wait`: polls until terminal; on
  `completed`, downloads the video to `--out` (default `<job>/preview.mp4` or
  `<job>/final.mp4`). `failed` → exit `1` with the error.

- **`cloud status <render_id> [--json]`**
  One status GET: `pending|running|completed|failed` (+ `video_url` when
  completed). `failed` → exit `1`.

## Verification / QA commands

- **`verify --job-dir <dir> [--json]`**
  **Cloud verification** of the rendered preview: uploads/points at the
  preview and the server re-checks every join for duplicated phrases (a
  retake left in → error) and over-long gaps (dead air → warning/error).
  Writes `verify.json`; exit `2` on needs-fixes. Requires a rendered preview
  (`compose` + hyperframes render, or `cloud render --mode preview`).

- **`qa frames --range <start>-<end> [--cols <n>] [--interval <s>] --job-dir <dir> [--json]`**
  **Local ffmpeg** filmstrip PNG for the window →
  `<job>/qa/frames_<start>_<end>.png`. Read the PNG to inspect frames.

- **`qa waveform --range <start>-<end> --job-dir <dir> [--json]`**
  **Local ffmpeg** waveform PNG + word-timing sidecar →
  `<job>/qa/waveform_<start>_<end>.png` and
  `<job>/qa/waveform_<start>_<end>.words.json` (word list with timings for
  the window). Free and fast — use for cut-edge adjudication.

## File sync commands

- **`files pull --job-dir <dir> [--only <name,name>] [--json]`**
  List the cloud job's artifacts and fetch each (or the `--only` subset) into
  the job dir, **preserving relative paths** (`resolved/plan.json` →
  `<job>/resolved/plan.json`, `prompts/author-edl.prompt.md` →
  `<job>/prompts/author-edl.prompt.md`). `.json` artifacts pretty-printed. Use to
  recover job state on a fresh machine.

- **`files push --job-dir <dir> [--only <name,name>] [--json]`**
  Upload local job artifacts (e.g. `timeline.json`, referenced assets under
  `assets/`) to the cloud job. Content-hash-cached — unchanged files are not
  re-uploaded.

## Review round commands (ClipReady review sessions)

- **`review pull --job-dir <dir> [--source] [--json]`**
  Fetch the review session: writes `<job>/review/round.json` (round/run ids,
  mode, status, requested changes) and `<job>/review/instruction.md`. Writes
  `<job>/timeline.json` from the session ONLY when no local one exists (never
  overwrites yours). `--source` also downloads the source video when absent.

- **`review push (--stage <type> [--message <text>] | --timeline | --complete --duration <s> [--outcomes <file>]) --job-dir <dir> [--run-id <id>] [--json]`**
  Push review-round events (one mode per call; run id defaults to
  `review/round.json`). `--stage` posts a progress event
  (`provisioning|transcribing|edl|render_preview|qc|render_final|uploading|log`).
  `--timeline` compiles `<job>/timeline.json` (same as `timeline compile`,
  exit `2` if not clean) and pushes a `timeline_update` — the review page
  updates live. `--complete` compiles fresh (must be clean) and pushes the
  terminal `completed` event (`{edl, timeline, plan, duration_s, outcomes?}`;
  `--outcomes` is a JSON file of `[{change_id, ok, note?}]`). Both
  `--timeline` and `--complete` upload the timeline's referenced assets
  automatically (sha256-cached in `<job>/review/assets.json`).

## Utility

- **`--version`** — print the CLI version.
- **`status --job-dir <dir> [--json]`** — cloud job state + local artifact existence.
- **`skill install [--target <dir>] [--json]`** — copy the bundled
  `video-editing` agent skill into `<target|~/.claude/skills>/video-editing`.

## Typical session

```
export CLIPREADY_API_BASE=https://…  CLIPREADY_API_KEY=…
video-editing init --video "clip.mp4" --job-dir jobs/clip
video-editing transcribe --job-dir jobs/clip
# READ jobs/clip/prompts/author-edl.prompt.md — it is the editorial brief. Author timeline.json.
video-editing timeline compile --job-dir jobs/clip --json     # fix until exit 0
video-editing compose --job-dir jobs/clip --render            # local preview render
video-editing verify --job-dir jobs/clip --json               # fix → recompile → re-render → re-verify
video-editing qa frames --range 0-5 --job-dir jobs/clip       # spot-check looks
video-editing cloud render --job-dir jobs/clip --mode final --wait
```
