# `video-editing` CLI reference (v3 — pure API client)

All execution goes through this CLI, and the CLI runs NOTHING locally: every
verb is HTTP against the ClipReady cloud plus small artifact pulls. Quote
paths (they contain spaces).

**Every verb requires ClipReady credentials.** Resolution:

- **base**: `--api-base` → env `CLIPREADY_API_BASE` → the hosted cloud (default)
- **key** (sent as `Authorization: Bearer <key>`): `--api-key` → env `CLIPREADY_API_KEY` → the config file written by `video-editing auth api-key`
- **video id**: `--video-id` → `<job>/job.json` (written by `init`) → env `VIDEO_ID`

A missing base or key is a clear error → non-zero exit, nothing runs. There is
**no ffmpeg/ffprobe/hyperframes requirement and no ElevenLabs
configuration** — media work is server-side in the ClipReady cloud.

Most verbs support `--json` for a machine-readable result. Exit codes: `0` ok,
`1` error (incl. missing credentials), `2` validation failed, `3` ambiguous
anchor (timeline group).

**Large uploads are chunked automatically** (no flag needed): the init source
streams up in 8MB parts with per-part retry — a 2GB file is never buffered in
memory — and on networks where direct storage PUTs keep dying, parts fall back
to an authenticated relay through the API. Servers without chunked-upload
support transparently fall back to a single retried upload.

## Pipeline commands

- **`init --source <path> [--job-dir <dir>] [--title <t>] [--poll-interval 5]`**
  ONE command covers what init+transcribe used to do. Streams the source up,
  creates the cloud job, runs server-side **ingest** (probe + audio
  extraction + transcription + editorial screening), polls with progress
  lines, then pulls into the job dir: `transcripts/*.json`, `word_dump.txt`,
  `takes_packed.md`, `flags.txt`, `coverage.json`, `probe.json`, and the
  editorial brief `prompts/author-edl.prompt.md` (plus any other
  `prompts/*.md`). Writes `job.json` (`video_id`, api base, stem,
  `duration_s` from the server probe, `source_path`) and prints the video's
  web page link (`web_url` in `--json`). The internal stem is sanitized to a
  filename-safe charset, so apostrophe/space/unicode filenames work end to
  end with no rename.

- **`transcribe --job-dir <dir> [--poll-interval 5]`**
  Re-run/pull only (init already transcribed): when the transcript artifacts
  exist server-side it just pulls them; otherwise it starts a fresh
  server-side ingest, polls, and pulls. No media flags — there is no local
  extraction.

## Timeline commands (`timeline.json` v2)

- **`timeline validate --job-dir <dir> [--json]`**
  Local schema + semantic + capability validation of `<job>/timeline.json`
  (the one purely local verb — it touches no media). Exit `2` if invalid.

- **`timeline resolve --job-dir <dir> --phrase <text> [--occurrence <n>] [--source <name>] [--json]`**
  Resolve a transcript phrase to **exact source seconds** (word-accurate,
  server-side). The ONLY way to turn a phrase into an anchor time. Multiple
  hits → exit `3` with per-occurrence context; pick one with `--occurrence`
  (1-based). Not found → fuzzy suggestions, exit `1`.

- **`timeline compile --job-dir <dir> [--json]`**
  **Cloud compile**: uploads `timeline.json`, the server validates, resolves
  anchors, sources silences from the job's cloud files, derives caption
  cues, and returns `resolved/plan.json` + `resolved/diagnostics.json`
  (written into the job dir). On success prints the `View & comment` watch
  link — the compiled cut is already watchable in the web app. Iterate until
  exit `0`. Exit `2` = validation failed (fix the reported `path`s), `3` =
  ambiguous anchor (disambiguate with `timeline resolve`). Never hand-edit
  anything under `resolved/`.

- **`timeline captions generate --job-dir <dir> [--preset grouped|opus-karaoke] [--json]`**
  Enable captions in `timeline.json` (preset only), then server-compile to
  derive/report cues.

## Render commands

- **`render [--mode preview|final] --job-dir <dir> [--no-wait] [--poll-interval 10] [--out <path>] [--json]`**
  THE render verb: submits a cloud render of the compiled plan (works right
  after `timeline compile`), **waits by default**, and downloads the result
  to `<job>/preview.mp4` / `<job>/final.mp4`. Mode defaults to `preview`.
  `--no-wait` submits and prints `render_id` only.

- **`cloud render --mode preview|final --job-dir <dir> [--wait] [--poll-interval 10] [--out <path>] [--json]`**
  Same implementation, legacy spelling (`--wait` is opt-in here).

- **`cloud status <render_id> [--json]`**
  One status GET: `pending|running|completed|failed` (+ `video_url` when
  completed). `failed` → exit `1`.

## Verification / QA commands

- **`verify --job-dir <dir> [--full] [--window <s>] [--json]`**
  **Fully server-side verification** of the rendered preview: the server
  re-checks every join for duplicated phrases (a retake left in → error) and
  over-long gaps (dead air → warning/error) against its own copy of the
  preview — nothing is uploaded. Writes `verify.json`; exit `2` on
  needs-fixes. Caveat: the duplicated-phrase check over-triggers on
  thematically narrow monologues — if every join is flagged (including plain
  pause-tightening joins), adjudicate against `flags.txt` instead of blindly
  recutting.

- **`qa frames (--start <s> --end <s> | --join <i> [--span 2] | --out-start <s> --out-end <s>) [--cols <n>] [--interval <s>] --job-dir <dir> [--json]`**
  **Server-rendered** filmstrip PNG(s) → `<job>/qa/frames_<start>_<end>.png`.
  Three addressing modes (exactly one): `--start/--end` in SOURCE seconds;
  `--join <i>` renders the two sides of compiled-plan join `i` (1-based —
  the last/first `--span` seconds of the segments meeting there) as
  `…-a.png` (tail) + `…-b.png` (head); `--out-start/--out-end` in OUTPUT
  seconds, mapped through `resolved/plan.json` (windows crossing a join
  split into `-a`/`-b`/… PNGs). Read the PNGs to inspect frames.

- **`qa waveform --start <s> --end <s> --job-dir <dir> [--json]`**
  **Server-rendered** waveform PNG + word-timing sidecar →
  `<job>/qa/waveform_<start>_<end>.png` and
  `<job>/qa/waveform_<start>_<end>.words.json` (word list with timings for
  the window, SOURCE seconds). Use for cut-edge adjudication.

- **`qa inspect (--start <s> --end <s> | --join <i> [--span 2] | --out-start <s> --out-end <s>) [--n-frames <n>] --job-dir <dir> [--json]`**
  **THE close-look tool** — a **server-rendered** combined composite PNG →
  `<job>/qa/inspect_<start>_<end>.png`: filmstrip + waveform + word labels
  in one image, with the compiled plan's cut regions shaded translucent
  red (labeled "cut") over the waveform. Same three addressing modes as
  `qa frames` (exactly one): `--start/--end` in SOURCE seconds; `--join <i>`
  renders the two sides of compiled-plan join `i` as `…-a.png` (tail) +
  `…-b.png` (head) — the sides are discontiguous in source time, hence two
  PNGs; `--out-start/--out-end` in OUTPUT seconds mapped through
  `resolved/plan.json`. `--n-frames` sets the filmstrip frame count
  (default 10). Prefer this over separate frames + waveform calls when
  adjudicating a cut.

## File sync commands

- **`files pull --job-dir <dir> [--only <name,name>] [--json]`**
  List the cloud job's files and fetch each (or the `--only` subset) into
  the job dir, **preserving relative paths** (`resolved/plan.json` →
  `<job>/resolved/plan.json`, `prompts/author-edl.prompt.md` →
  `<job>/prompts/author-edl.prompt.md`). Byte-exact, sha256-skipped. Use to
  recover job state on a fresh machine.

- **`files push --job-dir <dir> <paths…> [--json]`**
  Upload local job files (e.g. `timeline.json`, referenced assets) to the
  cloud job. Content-hash-cached — unchanged files are not re-uploaded.

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
  `--timeline` compiles `<job>/timeline.json` in the cloud (exit `2` if not
  clean) and pushes a `timeline_update` — the review page updates live.
  `--complete` compiles fresh (must be clean) and pushes the terminal
  `completed` event (`{edl, timeline, plan, duration_s, outcomes?}`;
  `--outcomes` is a JSON file of `[{change_id, ok, note?}]`). Both
  `--timeline` and `--complete` upload the timeline's referenced assets
  automatically (sha256-cached in `<job>/review/assets.json`).

- **`review prompt --job-dir <dir> [--json]`** — print the pulled round
  instruction.

## Utility

- **`--version`** — print the CLI version.
- **`status --job-dir <dir> [--json]`** — cloud job state + local artifact existence.
- **`skill install [--target <dir>] [--json]`** — copy the bundled
  `video-editing` agent skill into `<target|~/.claude/skills>/video-editing`.
- **`config set-key <key> | set-base <url> | path`** — store credentials in
  `~/.config/video-editing/.env` (the only verb that runs without one).

## Typical session

```
video-editing auth api-key            # once per machine (skip if already logged in)
video-editing init --source "clip.mp4" --job-dir jobs/clip   # upload + ingest + pull (transcribe included)
# READ jobs/clip/prompts/author-edl.prompt.md — it is the editorial brief. Author timeline.json.
video-editing timeline validate --job-dir jobs/clip --json
video-editing timeline compile --job-dir jobs/clip --json    # fix until exit 0; prints the watch link
video-editing render --job-dir jobs/clip                     # cloud preview render (waits + downloads)
video-editing verify --job-dir jobs/clip --json              # fix → recompile → re-render → re-verify
video-editing qa inspect --join 1 --job-dir jobs/clip        # close-look at the first join's two sides (filmstrip+waveform+cuts)
video-editing render --mode final --job-dir jobs/clip
```
