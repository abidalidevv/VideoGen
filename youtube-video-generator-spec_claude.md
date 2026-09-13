# YouTube Faceless Video Generator — Technical Spec & Build Plan

## 1. Goal

A desktop application (Electron) that takes two inputs — a voiceover audio file and a niche — and generates a finished, captioned YouTube video by automatically sourcing matching stock footage from multiple stock-video APIs, syncing it to the voiceover, and burning in animated CapCut-style captions. Built for team use, not a single-user tool.

## 2. Confirmed Requirements

- **Input:** voiceover audio + niche selection only. No manual clip picking on input.
- **Output length:** matches the audio's duration exactly.
- **Output formats:** both vertical 9:16 (Shorts/Reels) and landscape 16:9 (long-form), minimum full HD.
- **Footage source:** live-fetched stock video, pulled from multiple stock APIs. API keys are added/managed by the admin in a settings panel — not hardcoded per user.
- **Footage relevance:** the video on screen should track what the voiceover is saying at that moment (see Section 4 for the realistic version of this).
- **Captions:** CapCut-style animated captions with a template/animation system — adjustable color, stroke, font size, style, and animation.
- **Preview step:** required before final merge/export, so caption style can be adjusted without re-running the whole pipeline.
- **Platform:** true desktop application (Electron), installed per team member — not a web app, even though the reference tool that inspired this request is itself a web app.
- **Concurrency:** videos are downloaded via multiple parallel workers; multiple team members will run the tool at once.
- **Merge/export:** ffmpeg is the known baseline; open to a more advanced stack for the rest of the pipeline.

## 3. Key Decisions & Trade-offs

### 3.1 Desktop (Electron) vs. web app

Confirmed: **Electron desktop app.**

Trade-off worth keeping in mind: because API keys, rate-limit pooling, and the worker queue need to be centralized for team use (otherwise every team member's local instance fights over the same keys), the backend that actually does the work should live on a central server the Electron app talks to. In that shape, the Electron app is functionally a wrapper around a web service — the team-facing experience is close to the reference web app, plus installer/auto-update/code-signing overhead on top. That overhead is justified if there's a reason to want a native app specifically (offline use, later productizing/licensing the tool) — otherwise it's added engineering cost for limited user-facing gain on an internal tool. Noted here as a decision that was made deliberately, not a default.

### 3.2 Orientation: 9:16 and 16:9

Confirmed: **both formats supported**, not just one.

Implication: stock footage needs to be queried per-orientation (Pexels and Pixabay both support an orientation filter in their search APIs — use that instead of building a smart-crop/reframe system). Caption templates need orientation-aware layout presets, since safe margins and font scale differ between a tall vertical frame and a wide landscape frame.

## 4. Known Risks / Things to Verify Before Building

- **"Exact" voiceover-to-video matching is not fully achievable.** The reference video itself shows thematic/loose matches ("something in you is reaching" → a person raising an arm to the sky), not literal 1:1 matches. Realistic approach: extract 1–3 keywords per transcript chunk and pick the best-scoring available clip. Some chunks (abstract phrases) will get an approximate match — that's the ceiling of what any stock-footage pipeline can do, including the reference tool.
- **Stock API licensing terms differ per provider.** Some providers restrict bulk/automated redistribution or impose conditions on monetized use. Check each API's terms before wiring it in — this hasn't been verified yet and is a real channel-strike/takedown risk if skipped.
- **Shared API keys will hit rate limits under team-scale concurrent use.** Free/cheap stock APIs enforce per-key hourly limits. A key-pool or rotation strategy is needed once more than one or two people are running jobs at once.
- **The animated, style-editable caption preview is a project in itself.** This is effectively a lightweight video editor (canvas rendering synced to audio, live style controls), not a small feature bolt-on. It is also the piece most likely to take longer than expected, since it leans on the Node/React/canvas side of the stack rather than the WordPress/PHP/Vue side this developer is strongest in.

## 5. Architecture Overview

```
Voiceover + niche + orientation
        │
        ▼
Transcribe & analyze  (ASR + word-level timestamps, keyword extraction per chunk)
        │
        ▼
Fetch stock clips     (multi-API, orientation-filtered, parallel workers, queued)
        │
        ▼
Assemble & caption    (trim/sequence clips to audio length; live style-editable preview)
        │
        ▼
Render & export       (ffmpeg burn-in + merge, per-orientation output, full HD)
```

### Components

| Component | Role |
|---|---|
| Electron frontend | Project setup (audio upload, niche, orientation), live preview screen, caption style controls, admin/settings panel |
| Backend service | Centralized processing — recommend Python (FastAPI), not Node, since the ASR/video-processing ecosystem (faster-whisper, moviepy, ffmpeg-python, Celery) is far more mature there, and it avoids leaning on the weaker Node/backend skill area |
| Job queue | Redis + Celery (or RQ) — manages concurrent jobs across team members, prevents API rate-limit collisions |
| ASR | faster-whisper, run locally — no per-call cost, gives word-level timestamps used for both keyword extraction and caption word-highlight timing |
| Stock API layer | Abstraction over each provider (Pexels, Pixabay, etc., as added via admin settings) returning normalized clip metadata: url, duration, orientation, tags |
| Assembly | Per transcript chunk, pick best-matching clip, trim/loop to chunk duration, sequence |
| Captions | ASS subtitle format (libass) — its `\k` karaoke tag natively produces the CapCut-style word-highlight animation; burned in via ffmpeg. Live preview uses an HTML5 canvas overlay synced to audio, driven by a JSON style template (font, size, color, stroke, highlight color, animation type) so style edits don't require a re-render |
| Export | ffmpeg, GPU-accelerated (NVENC) where available, one output per confirmed orientation |
| Admin panel | API key management (encrypted at rest), per-user usage tracking, job history |

## 6. Build Phases

Sequenced so each phase produces something testable before the next begins — the goal is a working (if crude) pipeline early, rather than a long single build with nothing runnable until the end.

| Phase | Deliverable | What it validates |
|---|---|---|
| 1 | Audio upload → transcription → transcript shown on screen | Whisper's accuracy on the actual voiceovers being used |
| 2 | Niche/keyword → stock API fetch, orientation filter, clip list shown (no video yet) | API integration works; licensing terms checked per provider |
| 3 | Clips stitched to audio length, exported as plain HD video, no captions | Core assembly/export pipeline end-to-end |
| 4 | Static caption burn-in (ASS, no animation yet) | Caption timing/sync against the voiceover |
| 5 | Electron live preview UI with style editor | The hardest UI piece, built once the pipeline underneath is proven |
| 6 | Animation templates, admin panel, worker queue for team-scale concurrency | Full team-ready version |

## 7. Reference

The original idea reference was a screen-recorded demo of an existing web-based tool: "Upload Audio" input, a "Pipeline" selector (Main/Nature), a niche dropdown (e.g. "Motivation Psychology"), producing vertical 9:16 output with animated word-highlight captions over stock/motivational b-roll. This spec adapts that concept into a desktop app supporting both orientations, per the decisions above.
