# VideoGen — Final Consolidated Plan

Synthesized from four sources: ChatGPT's product/engineering plan, ChatGPT's architecture addendum, a Gemini/Antigravity build session transcript, and Claude's two earlier spec files (all in this repo). This document states what to keep, what to reject, and the final build plan — it supersedes the individual source documents where they conflict.

## 1. What to reject from the Gemini/Antigravity session — verify before trusting any of it

The Antigravity transcript claims the app is "complete, configured and verified." The evidence in the same transcript contradicts that:

- The seeded "verified" test project used a `provider: "studio_cinematic"` fallback for every clip — not Pexels or Pixabay. The core requirement (live multi-API stock footage) was never actually exercised.
- The test run was explicitly limited to 3 scenes, not a full voiceover.
- The claimed "FFmpeg 9.0.1" does not match any known FFmpeg release (current stable is in the 8.x range as of early 2026) — treat this as a fabricated detail.
- What was actually built is a Python/FastAPI server launched by a `.bat` file, opened in a browser tab at `localhost:8765`. This is a local web app, not a packaged, installable desktop application — it does not meet the "true desktop app, installed per team member" requirement.

**Action before reusing any of this code:** open `backend/stock_downloader.py` specifically and confirm it makes real, authenticated calls to Pexels/Pixabay and only falls back to `studio_cinematic` when those fail — not by default. Don't take the walkthrough screenshots as proof this works; verify the code path directly.

## 2. What to keep — points of agreement across all sources

- **Backend: Python + FastAPI.** All three independent sources converge here. Confirmed direction.
- **ASR: faster-whisper (local), word-level timestamps.** WhisperX as an option for tighter word alignment.
- **Captions: ASS/libass burned in via ffmpeg**, driven by a JSON style template so the live preview and final render read from the same source of truth.
- **Stock provider abstraction**: every provider (Pexels, Pixabay, future ones) sits behind a common adapter interface returning a normalized asset schema (id, url, width, height, duration, fps, orientation, license info). The scene planner never talks to a provider directly.
- **Semantic scene planning, not per-sentence clip changes.** Segment by topic/pause, not by every sentence — matches how the reference tool itself behaves.
- **License/attribution metadata stored per downloaded asset** (provider, asset ID, source URL, license reference, download date) — needed both for the "check licensing before production use" item flagged earlier, and for any future audit.
- **Duplicate/repetition avoidance** at the project level (asset IDs, recently-used categories, cooldown windows) — this also directly reduces the YouTube "inauthentic/repetitive content" monetization risk flagged earlier, so it's not optional polish.
- **Normalize every clip (scale, fps, pixel format) before concatenation** — confirmed independently by both Claude's and ChatGPT's plans as a common failure point.

## 3. Where sources disagreed — decisions made here

| Question | ChatGPT | Gemini/Antigravity | Decision |
|---|---|---|---|
| Desktop shell | Tauri (lighter, native) | Built a browser-based local server instead | **Electron**, per your confirmed choice earlier in this conversation — accept the packaging overhead as a deliberate trade-off |
| Job queue | asyncio + bounded worker pool for MVP; Redis/Celery only if scale demands it | ThreadPoolExecutor, 6–12 workers | **asyncio + bounded worker pool.** ChatGPT's reasoning is sound: Redis/Celery adds real operational complexity (a service to run, monitor, and keep alive) that an internal team tool doesn't need on day one. Revisit only if concurrent team usage actually saturates it. |
| Output orientation | 16:9 primary, 9:16 deferred to "later" as an output-profile addition | Not addressed as dual-output | **Both 9:16 and 16:9 from the start**, per your confirmed requirement. This is more work than either source assumed — build the composition engine resolution-aware from day one (ChatGPT's own addendum recommends this shape for exactly this reason, just sequenced later than you now need it). |
| Provider set | Start with 2 (Pexels, Pixabay) | Same, plus a fallback engine | **Pexels + Pixabay to start**, fallback engine only as a last resort when both fail — never as the default path, unlike how the Antigravity build actually used it |

## 4. Final architecture

```
Electron shell (React + TypeScript + Tailwind)
        │  HTTP/WebSocket
        ▼
FastAPI backend (Python)
├── Project service (SQLite)
├── Transcription (faster-whisper, word timestamps)
├── Scene planner (semantic segmentation + visual intent, LLM used only here — never for deterministic media steps)
├── Stock provider registry (Pexels / Pixabay adapters, normalized schema, license metadata)
├── Candidate scoring + fallback hierarchy + duplicate avoidance
├── asyncio worker pool (parallel download, bounded concurrency, retry w/ backoff)
├── Composition JSON (single source of truth for both preview and final render, resolution-aware: 1920x1080 and 1080x1920)
├── Caption engine (ASS/libass, JSON style template)
└── FFmpeg renderer (normalize → concat → caption burn-in → mux audio → GPU encode where available)
```

## 5. Build phases

Merged from ChatGPT's 9-phase roadmap and the earlier 6-phase plan, kept in small testable increments:

1. **Foundation** — Electron shell, FastAPI skeleton, SQLite project store, settings/API-key panel, FFmpeg + encoder detection.
2. **Audio intelligence** — upload, faster-whisper transcription, word timestamps, transcript viewer.
3. **Stock pipeline** — Pexels + Pixabay adapters, normalized results, orientation filtering, local cache, retries.
4. **Visual intelligence** — scene segmentation, visual-intent/query generation, candidate scoring, fallback hierarchy, duplicate avoidance.
5. **Timeline & basic render** — composition JSON, clip normalization, plain HD export for both orientations, no captions yet.
6. **Captions** — ASS burn-in, word-highlight timing, sync verification.
7. **Preview editor** — live style controls, scene/clip swap, the piece most likely to take longer than expected.
8. **Final rendering hardening** — GPU encoding, cancel/resume, autosave/recovery, quality gates.
9. **Team production features** — job history, per-user usage tracking, brand/niche presets.

Each phase should produce something you can actually run and check before the next starts — don't let Phase 7 (the editor) block Phases 1–6 from being verified independently.

## 6. Open items still unverified

- Pexels/Pixabay license terms for this specific use case (bulk automated fetch, monetized YouTube use) — check both providers' current terms directly.
- Whether the existing Antigravity-scaffolded code actually performs real API calls or silently uses the fallback — audit `stock_downloader.py` before building on top of it.
- Server/hosting decision for anything centralized (API key pooling, if the team ends up needing it) — not addressed by any of the three sources in concrete terms.
