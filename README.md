# ClipReady

Agents-first AI video editing. The **video-editing** skill + CLI let any coding
agent (Claude Code, Cursor, etc.) edit talking-head reels end to end: screen the
raw takes, cut bad retakes and dead air, add captions, zooms and stickers, and
run live review rounds — with hard verification gates so the cut is actually
clean, not just plausible.

## Install

```bash
npm install -g video-editing                                  # the CLI
npx skills add ItayElgazar/clipready --skill video-editing    # the skill
```

## Quickstart

1. Drop a raw talking-head clip anywhere on disk.
2. Ask your agent: "edit this reel" (or "cut the bad retakes", "tighten this").
3. The agent transcribes and screens the takes, authors the cut, and renders a
   vertical 9:16 preview.
4. It content-verifies every join (re-transcribes the render) before finalizing.
5. You get a tight, clean final cut — captions/zooms/stickers on request.

## Requirements

- Node ≥ 20
- `ffmpeg` / `ffprobe` on PATH
- An ElevenLabs API key (`ELEVENLABS_API_KEY`) for transcription

## Cloud review

Live review rounds and cloud rendering (`video-editing review …`, `cloud render`)
need a ClipReady account and API key (`CLIPREADY_API_BASE`, `CLIPREADY_API_KEY`).
Everything else runs fully local.

## License

Free to install and use; no redistribution or derivative works. See [LICENSE](LICENSE).
