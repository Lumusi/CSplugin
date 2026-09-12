# NTV Live — CloudStream Plugin Plan

Goal: client-side CloudStream plugin for NTV live content (24/7 channels + events).
All stream resolution runs on the phone (like CineStream/CSX). No `/seg` relay, no server bandwidth.

## 1. Why this shape

- CineStream (`SaurabhKaperwan/CSX`) proves the VOD client-side pattern but has no live path.
- `Kraptor123/Cs-Karma/Streamed` proves the live path: `TvType.Live` + `loadLinks` fan-out + WebView-sniff extractor that sidesteps the GOAT/wasm unlock entirely.
- NTV's kobra/viper/dlhd `(source, id)` pairs are the same `embed.st` slots Streamed already handles. Reuse its extractor approach; swap the listing API from `streamed.pk` to `ntv.st`.

## 2. Architecture

```
getMainPage / search          load()                loadLinks()                 Extractor
ntv.st get-matches/get-       find fixture,         for each (source,id):       WebView loads
channels -> LiveSearch-       pass dataUrl          build embed.st URL ->       embed.st/embed/..
Response(title,               through               loadExtractor()             sniff .m3u8 -> callback
  "ntv:<srv>:<src>:<id>:<n>")
```

- `supportedTypes = setOf(TvType.Live)`, `hasMainPage = true`, `hasQuickSearch = true`.
- `load()` does no unlock — just fixture lookup, returns Live load with `dataUrl = url`.
- `loadLinks()` resolves fresh per tap (correct for short-lived tokens).
- Extractor (`NtvEmbedExtractor`, port of `EmbedStreams`) opens embed URL in Android WebView, autoplays (jwplayer/clappr/`video.play()` click script), intercepts first `.m3u8`/`.mpd` in `shouldInterceptRequest`, returns `newExtractorLink(M3U8)` with Referer/Origin/UA headers.

## 3. Data sources (from server.py chain ref, Sep 2026)

| Content | API | loadLinks action |
|---|---|---|
| Events (all servers) | `GET ntv.st/api/get-matches?server=<srv>&type=both` → `{live, all}` with `sources[]` | direct-URL sources (most servers): callback immediately; `(source,id)` pairs (kobra/viper/dlhd): build `https://embed.st/embed/<src>/<id>/<n>` → extractor |
| 24/7 channels | `GET ntv.st/api/get-channels?limit=100&offset=N` (dlhd entries only) → dlhd iframe→atob chain | plain `app.get` + regex in `loadLinks`, no WebView needed |
| zlive events/24-7 | `cast.zlive.st/*.json` → `iptv.zlive.st/<slug>` → 302 signed m3u8 | follow redirect in `loadLinks`, callback directly |
| golf detour | `embed.st/embed/golf/<id>/<n>` → iframe `embedhd.st` → base64 → real GOAT slot | let WebView follow it automatically (no special code) |

Main page sections = one `mainPageOf` entry per server (`kobra`, `dlhd`, `raptor`, `falcon`, `phoenix`, `titan`, `viper`, `zlive`) + one for 24/7. Reuse Streamed's per-sport `mainPageOf` pattern.

## 4. File layout (new repo dir, e.g. `D:\Projects\NTV-Cloudstream\`)

```
settings.gradle.kts  (include :NTVLive)
build.gradle.kts     (root, cloudstream gradle plugin)
NTVLive/
  build.gradle.kts   (version, cloudstream { tvTypes = ["Live"], ... })
  icon.png
  src/main/AndroidManifest.xml
  src/main/kotlin/com/ntv/
    NtvLive.kt          // MainAPI: mainPage/search/load/loadLinks + data classes
    NtvEmbedExtractor.kt// ExtractorApi: WebView sniff (port of EmbedStreams)
    NtvPlugin.kt        // @CloudstreamPlugin registerMainAPI + registerExtractorAPI
```

Reference files to copy first: `Streamed.kt`, `EmbedStreamsExtractor.kt`, `StreamedPlugin.kt`, `build.gradle.kts` from Cs-Karma/Streamed.

## 5. Build tasks

1. Scaffold repo + gradle (CloudStream template), `tvTypes = ["Live"]`, confirm empty plugin compiles and loads in app.
2. Port `Streamed.kt` listing → `NtvLive.kt`: replace `streamed.pk` discovery with `ntv.st get-matches` per server; keep `Matches/Sources/Stream` data classes reshaped to ntv.st JSON (`live[]`, `all[]`, `sources[]`).
3. Port `load()` fixture lookup (by `ntv:<srv>:<src>:<id>:<n>` id, fallback to first source).
4. Port `loadLinks()` fan-out: direct urls → callback; `(source,id)` → `loadExtractor("https://embed.st/embed/...")`; sort live-first; dedupe embed URLs.
5. Port `EmbedStreams` → `NtvEmbedExtractor` verbatim, retarget `mainUrl = "https://embed.st"`, keep autoplay JS + `.m3u8/.mpd` intercept + 15s timeout + alpha/bravo/… naming (adapt to ntv source names).
6. Add dlhd atob + zlive 302 fast paths in `loadLinks` (skip WebView where direct resolve works).
7. Headers: pass `Referer/Origin/UA` on every `ExtractorLink`; poster headers with Firefox UA as Streamed does.
8. Test matrix: one event per server type, 24/7 channel, golf-detour event, expired/token-refresh (re-tap), airplane→retry.

## 6. Pitfalls

- WebView resolve is 5–15s per source; run sources concurrently, return first hits immediately, don't await all.
- Embed page redesigns break the autoplay/sniff — keep the JS generic (jwplayer + clappr + `video` tags), never depend on one selector.
- Mobile-IP Cloudflare differs from server IP; keep `VPNStatus.MightBeNeeded`.
- Never put the wasm unlock in the plugin — the whole point is the page does it for you.
- Server (`server.py` + `goat-unlock.mjs`) stays untouched; this plugin is additive (CloudStream app viewers) not a migration (TiviMate/VLC still use `:8899`).

## 7. Done when

- Plugin loads, home shows per-server sections, search finds fixtures.
- Tapping a live event yields ≥1 playable M3U8 within ~15s on mobile data.
- 24/7 dlhd channel plays without WebView (fast path).
- No NTV server traffic during CloudStream playback (verify via `/stats`).
