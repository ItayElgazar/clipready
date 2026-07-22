# ClipReady

Agents-first AI video editing. The **video-editing** skill + CLI let any coding
agent (Claude Code, Cursor, etc.) edit talking-head reels end to end: the
ClipReady cloud transcribes and screens the raw takes and sends the agent its
editorial brief; the agent authors the cut (plus captions, zooms, stickers) as
a `timeline.json`; the cloud compiles and content-verifies it; rendering and
QA visuals run locally on your machine.

**Cloud-only:** every command requires a ClipReady API key. The editorial
engine runs in ClipReady cloud; rendering and QA run locally.

## Install

```bash
npm install -g video-editing                                  # the CLI
npx skills add ItayElgazar/clipready --skill video-editing    # the skill
```

## Auth

```bash
export CLIPREADY_API_BASE=https://…   # the ClipReady API base
export CLIPREADY_API_KEY=…            # from the ClipReady settings page
```

Nothing runs without the key. Transcription happens in the ClipReady cloud —
you do not need any transcription API key of your own.

## Quickstart

1. Drop a raw talking-head clip anywhere on disk.
2. Ask your agent: "edit this reel" (or "cut the bad retakes", "tighten this").
3. The agent uploads it (`init`), the cloud transcribes + screens it and sends
   back the editorial brief (`prompts/*.md`).
4. The agent authors `timeline.json`, compiles it in the cloud, renders a
   preview locally, and cloud-verifies every join before finalizing.
5. You get a tight, clean final cut — captions/zooms/stickers on request —
   plus live review rounds (`review pull` / `review push`).

## Requirements

- Node ≥ 20
- `ffmpeg` / `ffprobe` on PATH
- A ClipReady API key (`CLIPREADY_API_KEY` + `CLIPREADY_API_BASE`)

## License

Free to install and use; no redistribution or derivative works. See [LICENSE](LICENSE).
