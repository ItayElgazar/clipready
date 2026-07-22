# Timeline v2 vocabulary — feature-by-feature reference

Every feature of the authored `<job>/timeline.json` (version 2), with one minimal JSON
example each. Validate with `video-editing timeline validate`; compile with
`timeline compile` (cloud compile — it resolves phrase anchors, derives caption cues
and emits `resolved/plan.json`). Items all carry a unique `id`; word anchors
(`{type:"words", source, phrase}`) resolve via the transcript — use `timeline resolve`
to disambiguate. Capability status per feature: run `timeline validate` — `staged`
features compile but warn (render path not yet proven); `ready` features render via
`compose` / the cloud render.

## Cut (keep-ranges)

```json
{ "cut": { "ranges": [ { "id": "rng_1", "source": "SRC", "start": 11.35, "end": 14.12, "beat": "HOOK" } ] } }
```

## Captions (derived at compile — never hand-write cues)

```json
{ "captions": { "enabled": true, "preset": "grouped", "style": { "maxWords": 4 } } }
```

### Word styling — the canonical "make word X red"

`op:"style"` overrides tag the anchored words with `color` (hex or simple CSS name)
and/or `emphasis` (`"pop"` = scale pop at the word's onset, `"highlight"` = background
highlight sweep). At least one of color/emphasis is required. Styling never changes
cue grouping.

```json
{
  "captions": {
    "enabled": true,
    "overrides": [
      {
        "id": "capfix_red",
        "op": "style",
        "anchor": { "type": "words", "source": "SRC", "phrase": "important" },
        "color": "#ff5c5c"
      }
    ]
  }
}
```

Emphasis variant: add `"emphasis": "pop"` (or `"highlight"`) to the same override.

### Karaoke sweep (staged)

`style.karaoke: true` marks every derived cue for a per-word highlight sweep (each
word brightens at its onset within the grouped cue).

```json
{ "captions": { "enabled": true, "style": { "karaoke": true } } }
```

### Flow overrides (existing)

`op:"replace-text"` (needs `text`), `op:"hide"`, `op:"break-line"` — same anchor shape.

## Zooms

Static punch-in: `rect` (unit-canvas crop region, `x+w<=1`, `y+h<=1`) + `scale`
(`1 < scale <= 3`).

```json
{
  "zooms": [
    {
      "id": "zm_1",
      "anchor": { "type": "words", "source": "SRC", "phrase": "the key moment" },
      "rect": { "x": 0.25, "y": 0.2, "w": 0.5, "h": 0.5 },
      "scale": 1.35
    }
  ]
}
```

### Zoom ramp

Add `to` (`rect` and/or `scale`, same bounds) — the zoom animates from the base
rect/scale to `to` over the zoom window. `ease: {in, out}` shapes the ramp
(both > 0 → smooth in-out; one side → one-sided ease; both 0 → linear).

```json
{
  "zooms": [
    {
      "id": "zm_ramp",
      "anchor": { "type": "source", "source": "SRC", "start": 18, "end": 22 },
      "rect": { "x": 0.25, "y": 0.2, "w": 0.5, "h": 0.5 },
      "scale": 1.2,
      "to": { "scale": 1.8, "rect": { "x": 0.1, "y": 0.1, "w": 0.4, "h": 0.4 } },
      "ease": { "in": 0.25, "out": 0.25 }
    }
  ]
}
```

## Overlays

Text / title-card (existing): `kind:"text"|"title-card"` + `text`.

```json
{
  "overlays": [
    { "id": "ov_1", "kind": "text", "text": "wait for it…", "anchor": { "type": "output", "at": "start", "durationS": 2 } }
  ]
}
```

### Sticker / image (positioned image, no card chrome)

`kind:"sticker"` (or `"image"`) requires `asset.ref` — a **job-dir-relative path**
(convention: keep files under `<job>/assets/`). `layout.x/y` is the center (unit
canvas), `size.w/h` are canvas fractions (sticker default ~0.25 of width, square),
`z` (0–50) is the stacking order. `enter`/`exit` are animation presets
(`fade | pop | slide-up | slide-down | scale-in`, `durationS` ≤ 2, capped by the
overlay's on-screen window; exit ends exactly at the overlay's end).

```json
{
  "overlays": [
    {
      "id": "ov_logo",
      "kind": "sticker",
      "asset": { "ref": "assets/logo.png" },
      "anchor": { "type": "words", "source": "SRC", "phrase": "our product", "edge": "start" },
      "durationS": 2.5,
      "layout": { "x": 0.3, "y": 0.35 },
      "size": { "w": 0.25 },
      "z": 10,
      "enter": { "preset": "pop", "durationS": 0.35 },
      "exit": { "preset": "fade", "durationS": 0.4 }
    }
  ]
}
```

Missing asset files fail `timeline compile` with `asset-missing`. On
`review push --timeline/--complete` referenced assets are uploaded to clipready
automatically (sha256-cached in `<job>/review/assets.json`).

## Audio beds (staged)

```json
{
  "audio": [
    { "id": "aud_1", "kind": "music", "asset": { "ref": "assets/bed.mp3" }, "anchor": { "type": "output", "at": "full" }, "gainDb": -18 }
  ]
}
```

## Grade (staged)

Declarative color grade for the composition render path — distinct from the legacy
ffmpeg grade string (which `settings.grade` still accepts).

```json
{ "settings": { "grade": { "preset": "teal-orange", "intensity": 0.6 } } }
```
