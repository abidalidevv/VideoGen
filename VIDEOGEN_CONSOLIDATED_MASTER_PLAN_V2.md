# VideoGen Studio — Consolidated Master Implementation Plan v2

## 0. Document purpose

This document consolidates:

- the original product requirements;
- the reference video workflow;
- the existing ChatGPT plan in the repository;
- the existing Claude specification;
- the repository's master blueprint;
- additional architecture/performance recommendations;
- decisions needed before implementation starts.

Repository reviewed:

`https://github.com/abidalidevv/VideoGen`

Current repository documents reviewed:

- `AI_YouTube_Video_Generator_Detailed_Plan_Chatgpt.md`
- `youtube-video-generator-spec_claude.md`
- `VIDEOGEN_MASTER_BLUEPRINT.md`

The repository currently acts as a planning/specification repository rather than a finished production implementation.

---

# 1. Final product definition

## Product name

**VideoGen Studio**

## Primary goal

A Windows desktop application for a content-production team that turns:

1. **Voiceover audio**
2. **Niche / topic context**

into a finished YouTube-ready video with:

- voiceover-synchronized stock footage;
- animated captions;
- editable caption styles;
- scene replacement;
- preview before export;
- Full HD 16:9 output;
- fast, parallel stock retrieval;
- team-safe API rate limiting;
- project history and reproducibility.

## Core product rule

The application should feel simple to the team member even though the underlying pipeline is complex.

Normal workflow:

`Upload Audio → Select Niche → Generate → Preview/Edit → Render → Final MP4`

No complicated video-editor knowledge should be required for normal jobs.

---

# 2. Non-negotiable requirements

## Input

Normal generation screen exposes only:

- Voiceover upload
- Niche selector/input
- Generate button

Supported audio should include at minimum:

- MP3
- WAV
- M4A/AAC

## Output

Primary output:

- 1920×1080
- 16:9
- H.264
- AAC
- 30 FPS default
- optional 60 FPS
- exact voiceover duration within a very small technical tolerance
- MP4 container

The architecture should also support a later 9:16 mode without rewriting the core pipeline.

## Stock footage

Footage must be sourced through provider adapters, initially supporting at least:

- Pexels
- Pixabay
- custom provider adapter

More providers can be added without changing the scene planner or renderer.

## Caption editor

The user must be able to preview and modify:

- template
- font family
- font size
- normal text color
- active-word color
- stroke color
- stroke width
- shadow
- position
- line/word layout
- animation
- uppercase/lowercase behavior

## Scene editor

Every scene must support:

- preview
- replace clip
- search alternative clips
- trim/fit behavior
- lock scene
- regenerate selected scene only

## Preview

Preview must happen before final render.

Changing caption style must NOT trigger a new stock search or transcription pass.

---

# 3. Important decision: exact visual matching

“Whatever the voiceover says, show exactly that” is the desired UX, but it cannot be guaranteed literally with stock footage.

The production requirement should therefore be defined as:

> **Best available semantic visual match for each spoken segment, with AI-generated fallback strategies when literal footage is unavailable.**

Example:

Voiceover:

> “The company lost four billion dollars this quarter.”

Possible visual hierarchy:

1. exact company + finance clip;
2. company office / headquarters;
3. stock market / financial chart;
4. business loss / money concept;
5. generic finance cinematic footage.

This avoids empty scenes and unrealistic failures.

The system should also expose a visual-confidence value internally:

- 90–100: strong match
- 75–89: good match
- 50–74: approximate match
- below 50: weak match / fallback used

Low-confidence scenes can be flagged for manual review.

---

# 4. Final architecture decision

## Recommended production architecture

Use a **desktop client + centralized processing backend**.

```text
┌─────────────────────────────────────────────┐
│              VideoGen Desktop               │
│        Tauri 2 + React + TypeScript         │
│                                             │
│ Upload / Projects / Preview / Editor / UI   │
└──────────────────────┬──────────────────────┘
                       │ HTTPS / WebSocket
                       ▼
┌─────────────────────────────────────────────┐
│            Central VideoGen API              │
│        Python + FastAPI + PostgreSQL         │
├─────────────────────────────────────────────┤
│ Auth / Projects / Jobs / Provider Settings  │
│ Scene Planning / Asset Management / Events  │
└─────────────┬───────────────┬───────────────┘
              │               │
              ▼               ▼
       ┌────────────┐   ┌──────────────┐
       │ Redis      │   │ Object       │
       │ Queue/Cache│   │ Storage      │
       └─────┬──────┘   └──────────────┘
             │
             ▼
┌─────────────────────────────────────────────┐
│             Worker Pool                     │
│                                             │
│ ASR Worker                                  │
│ Scene/AI Worker                             │
│ Stock Search Worker                         │
│ Download Worker                             │
│ Preview/Proxy Worker                        │
│ Render Worker                               │
└──────────────────────┬──────────────────────┘
                       │
          ┌────────────┼─────────────┐
          ▼            ▼             ▼
       Pexels       Pixabay      Custom APIs

                       │
                       ▼
              FFmpeg / libass
```

### Why this is the final recommendation

The earlier documents correctly identify that multiple team members sharing API keys create rate-limit and concurrency problems. A centralized backend solves this cleanly.

The desktop app remains the team-facing application, but API keys, queues, download workers, and render workers are centralized.

This keeps the UI desktop-native while making team production manageable.

---

# 5. Tauri vs Electron decision

## Final recommendation: Tauri 2

Use:

- Tauri 2
- React
- TypeScript
- Vite

Reasons:

- smaller desktop footprint;
- strong Windows desktop fit;
- React frontend retained;
- native desktop shell without forcing the entire backend into JavaScript;
- easier path to a polished internal application.

Electron remains technically valid and was selected in the Claude specification, but the consolidated plan chooses Tauri because the backend is Python/FastAPI regardless of desktop shell.

The desktop shell should never contain core generation business logic.

---

# 6. Frontend stack

## Core

- React
- TypeScript
- Vite
- Tauri 2

## UI

- Tailwind CSS
- shadcn/ui / Radix primitives

## State

- Zustand for editor/project UI state
- TanStack Query for API/server state

## Validation/forms

- React Hook Form
- Zod

## Timeline/editor

A lightweight custom timeline is preferred over adopting a huge general-purpose video editor framework.

Required layers:

```text
Audio waveform
Video scenes
Captions
Optional BGM
Markers
```

## Preview rendering

HTML5 video for the video layer plus a synchronized caption overlay.

Recommended preview architecture:

```text
<video>
   +
Canvas / DOM caption layer
   +
Timeline state
```

The preview layer reads the same versioned composition JSON that the renderer consumes.

---

# 7. Backend stack

## Runtime

- Python 3.x
- FastAPI
- Uvicorn

## Database

Production:

- PostgreSQL

Desktop/offline development fallback:

- SQLite

## Queue

Production team mode:

- Redis
- Celery or RQ

For a single-machine development mode, the system should also be able to operate with an internal asyncio/bounded-worker queue so the codebase is not blocked by Redis during early development.

## Storage

Use an abstraction rather than hard-coding local folders everywhere.

Development:

- local filesystem

Production:

- S3-compatible object storage

Potential providers:

- Cloudflare R2
- AWS S3
- Backblaze B2

---

# 8. AI / speech stack

## ASR

Primary local option:

- faster-whisper

High-accuracy alignment option:

- WhisperX

Optional cloud acceleration:

- provider-based Whisper endpoint such as Groq/OpenAI when configured

The system should expose an `ASRProvider` abstraction.

```text
ASRProvider
 ├── LocalFasterWhisper
 ├── WhisperX
 ├── GroqWhisper
 └── OpenAIWhisper
```

The project should store the actual ASR provider/model used for reproducibility.

## Internal transcript representation

```json
{
  "language": "en",
  "duration": 163.2,
  "segments": [
    {
      "id": "seg_001",
      "start": 0.0,
      "end": 4.8,
      "text": "Dubai is becoming a major business hub.",
      "words": [
        {"word": "Dubai", "start": 0.0, "end": 0.62, "confidence": 0.98},
        {"word": "is", "start": 0.62, "end": 0.79, "confidence": 0.99}
      ]
    }
  ]
}
```

This representation powers both visual scene planning and animated captions.

---

# 9. Scene planning engine

This is the most important intelligence layer.

## Input

- transcript
- word timestamps
- niche
- style profile
- orientation
- target duration

## Output

Each scene contains:

- start time
- end time
- spoken text
- visual intent
- primary query
- alternate queries
- visual category
- mood
- preferred shot type
- orientation
- importance
- confidence requirement

Example:

```json
{
  "id": "scene_009",
  "start": 28.2,
  "end": 33.7,
  "spoken_text": "More investors are moving into the region.",
  "visual_intent": "business investors entering modern finance district",
  "queries": [
    "investors business meeting",
    "financial district professionals",
    "UAE business investment"
  ],
  "shot_type": "medium / wide",
  "mood": "premium",
  "orientation": "landscape",
  "importance": 0.91
}
```

## Scene duration

Do NOT blindly cut every 3 seconds.

Use semantic boundaries and voiceover rhythm.

General guidance:

- fast content: 2–4 sec
- normal content: 3–6 sec
- cinematic content: 5–9 sec

The final timeline must still equal the voiceover duration.

---

# 10. LLM layer

Use an LLM for semantic reasoning, not deterministic media operations.

Good LLM tasks:

- scene boundary suggestions;
- visual intent;
- stock queries;
- alternate stock queries;
- proper-noun recognition;
- niche context;
- mood/style classification;
- visual fallback strategy.

Bad LLM tasks:

- calculating media duration;
- trimming clips;
- encoding videos;
- muxing audio;
- file validation.

Recommended abstraction:

```text
LLMProvider
 ├── OpenAI
 ├── Anthropic
 ├── Google
 └── Local / OpenAI-compatible
```

The actual model should be configurable rather than hard-coded into business logic.

---

# 11. Stock provider architecture

Use a strict adapter pattern.

```text
StockProvider
 ├── PexelsProvider
 ├── PixabayProvider
 ├── CustomProvider
 └── FutureProvider
```

Normalized asset:

```json
{
  "provider": "pexels",
  "provider_asset_id": "12345",
  "source_url": "...",
  "download_url": "...",
  "width": 1920,
  "height": 1080,
  "duration": 14.8,
  "fps": 30,
  "orientation": "landscape",
  "author": "...",
  "license_url": "..."
}
```

The project database should preserve provider/license metadata.

---

# 12. Stock search pipeline

For every scene:

```text
Scene intent
   ↓
Primary query
Secondary query
Alternative query
Context query
   ↓
Search all enabled providers
   ↓
Normalize results
   ↓
Remove duplicates
   ↓
Score candidates
   ↓
Download top candidates
   ↓
Validate media
   ↓
Select winner
```

Do not download every search result.

Prefer downloading a small set of top candidates, then expanding only when confidence is low.

---

# 13. Clip ranking

Recommended starting score:

```text
Semantic relevance       35%
Niche/context relevance 15%
Quality/resolution      15%
Duration fit            10%
Orientation fit         10%
Motion/composition      10%
Variety                  5%
```

Later, make these weights profile-dependent.

Examples:

- luxury profile can increase composition/premium quality;
- news can favor subject accuracy;
- motivation can favor human-action relevance.

---

# 14. Duplicate avoidance

Avoid:

- same clip twice;
- visually near-identical clips;
- repetitive skyline shots;
- same provider repeated excessively;
- same shot category back-to-back.

Use:

- provider asset ID;
- perceptual hash;
- thumbnail embeddings/hashing where justified;
- category cooldown;
- recent scene history.

---

# 15. Worker architecture

## Production

```text
Job Queue
   ↓
Download workers
   ↓
Validation workers
   ↓
Proxy workers
   ↓
Render workers
```

Workers should have bounded concurrency.

Do not blindly set 20+ downloads just because the CPU can create threads.

Each provider needs independent limits.

```text
Pexels concurrency = N
Pixabay concurrency = M
Custom API concurrency = K
```

## Retry policy

Use exponential backoff.

Retry categories:

- network timeout;
- temporary HTTP error;
- rate limit;
- interrupted download.

Do NOT repeatedly retry permanent authentication failures.

---

# 16. API key management

Admin-only settings should support:

- provider name;
- API key;
- enable/disable;
- priority;
- concurrency limit;
- rate limit;
- test connection.

Secrets must be encrypted at rest.

Never put API keys into:

- project JSON;
- normal logs;
- frontend state sent to regular users;
- GitHub.

Recommended production architecture:

```text
Team Desktop
   ↓
Central API
   ↓
Secret store / encrypted DB
   ↓
Provider requests
```

---

# 17. API key pooling

The system should support multiple keys per provider.

Example:

```text
Pexels
 ├── Key A
 ├── Key B
 └── Key C
```

The scheduler can select an eligible key according to:

- remaining quota;
- cooldown;
- provider priority;
- failure rate.

Important: key pooling must not violate the provider's terms. It is a technical scheduler, not a mechanism to bypass provider restrictions.

---

# 18. Licensing and source tracking

Every downloaded stock asset should record:

- provider;
- asset ID;
- author/creator when available;
- source URL;
- license URL;
- retrieval timestamp;
- project usage.

Generate an optional `assets_manifest.json` for each project.

This is important for commercial YouTube workflows.

---

# 19. Asset caching

Cache:

- downloaded originals;
- thumbnails;
- proxy videos;
- search results;
- transcripts;
- scene plans.

Cache key examples:

```text
provider + asset_id
query + provider + filters
sha256(audio)
project_id + composition_version
```

Same stock asset should never be re-downloaded unnecessarily.

---

# 20. Proxy media

The editor should not attempt real-time playback of every 4K source.

Pipeline:

```text
Original stock
     ↓
Proxy generation
     ↓
720p / low bitrate
     ↓
Fast preview
```

Final export always uses original media.

This makes timeline scrubbing significantly smoother.

---

# 21. Composition model

The composition JSON is the single source of truth for the generated video.

Example:

```json
{
  "schema_version": 1,
  "project_id": "proj_001",
  "format": {
    "width": 1920,
    "height": 1080,
    "fps": 30
  },
  "audio": {
    "asset_id": "audio_001",
    "start": 0,
    "duration": 163.2
  },
  "scenes": [],
  "caption_style": {},
  "tracks": [],
  "render_settings": {}
}
```

Version this schema from the beginning.

Never assume old projects will always have the current structure.

---

# 22. Caption engine

## Rendering

Use:

- ASS/libass for final caption rendering;
- FFmpeg for final burn-in;
- HTML/canvas overlay for interactive preview.

ASS karaoke timing is well-suited to word highlighting.

## Caption templates

Start with 8–12 templates.

Suggested families:

1. Bold Highlight
2. Active Word Yellow
3. Active Word Green
4. Clean Minimal
5. Red Accent
6. Karaoke Fill
7. Pop Bounce
8. Pill Highlight
9. Typewriter
10. Cinematic Lower Third

Each template should be JSON-driven.

---

# 23. Caption style JSON

Example:

```json
{
  "name": "Bold Highlight",
  "fontFamily": "Montserrat ExtraBold",
  "fontSize": 58,
  "weight": 800,
  "fill": "#FFFFFF",
  "stroke": "#000000",
  "strokeWidth": 4,
  "shadow": {
    "enabled": true,
    "blur": 3,
    "offsetX": 0,
    "offsetY": 2
  },
  "activeWord": {
    "color": "#FFD400",
    "scale": 1.08,
    "animation": "pop"
  },
  "position": {
    "anchor": "bottom-center",
    "marginBottom": 100
  }
}
```

Global style + scene-level override should be supported.

---

# 24. Caption preview behavior

Changing:

- font;
- color;
- stroke;
- size;
- animation;
- position;

must update preview without:

- re-transcribing;
- re-searching stock;
- re-downloading footage.

This is a hard acceptance requirement.

---

# 25. Timeline editor

Minimum tracks:

```text
Track 1: Voiceover
Track 2: Video scenes
Track 3: Captions
Track 4: Optional BGM
Track 5: Optional SFX
```

Features:

- play/pause;
- scrub;
- zoom;
- scene selection;
- caption selection;
- split/trim later;
- lock scene;
- replace clip;
- regenerate scene.

---

# 26. Regenerate selected scene

Do not regenerate the full project when one scene is bad.

Workflow:

```text
Select Scene 12
      ↓
Regenerate
      ↓
Search fresh candidates
      ↓
Score
      ↓
Preview alternatives
      ↓
Apply
```

All other scenes remain unchanged.

---

# 27. Manual clip replacement

The scene panel should offer:

- current clip;
- candidate clips;
- search box;
- provider filter;
- duration filter;
- landscape filter;
- “use selected”.

This is the human override layer that protects production quality.

---

# 28. Confidence-driven editing

Show a visual warning on weak scenes.

Example:

```text
Scene 12
Visual match: 54%
⚠ Approximate match

[Find Better Clip]
```

This focuses human attention only where needed.

---

# 29. Niche profiles

The niche input should support reusable profiles.

Example:

```text
Motivation
Business
Finance
AI / Technology
Luxury
Real Estate
Fitness
Nature
Documentary
```

A niche profile may contain:

- preferred search language;
- visual mood;
- clip pace;
- caption template;
- preferred caption colors;
- preferred shot types;
- provider priority;
- fallback strategy.

The user still only needs to select the niche on normal jobs.

---

# 30. Visual pace settings

Profiles should support:

- Slow
- Normal
- Fast
- Very Fast

This affects scene duration and visual-change frequency.

Do not hard-code a universal “3–5 seconds”.

---

# 31. Branding system

Future-ready brand kit:

```text
Logo
Fonts
Primary colors
Caption presets
Intro
Outro
Watermark
Default export preset
```

A team member can select a brand profile and generate consistently branded output.

---

# 32. Optional BGM and audio ducking

Design the audio system so BGM can be added without architecture changes.

Later feature:

```text
Voiceover = primary
BGM = secondary
SFX = optional
```

Automatic ducking should reduce BGM while voiceover is active.

Do not make this a v1 blocker.

---

# 33. Optional subtitle SFX

Later feature:

- pop
- whoosh
- click
- hit

These should be template-driven and optional.

---

# 34. Smart video framing

For the primary YouTube format, filter for landscape footage before download whenever the provider supports orientation filtering.

For portrait/other source ratios:

- crop;
- scale;
- optional focal-point detection.

Never stretch footage.

---

# 35. Fair-use/copyright recommendation correction

Do NOT treat zooming/cropping as a guarantee of copyright safety or YouTube monetization eligibility.

A transform does not automatically make licensed stock footage legally safe or transform it into a guaranteed “fair use” case.

The project should instead rely on:

- legitimate stock licenses;
- provider terms;
- original voiceover;
- original edit/composition;
- appropriate transformations where needed;
- maintaining source/license records.

---

# 36. Render engine

FFmpeg remains the final media engine.

Responsibilities:

- trim;
- concatenate;
- scale;
- crop;
- overlays;
- subtitle burn-in;
- audio mix;
- mux;
- encode;
- validation.

Do not replace FFmpeg just for the sake of using a newer library.

The application should create deterministic FFmpeg filter graphs from the composition JSON.

---

# 37. Hardware encoder strategy

At startup detect:

- NVIDIA NVENC
- AMD AMF
- Intel QSV

Fallback:

- libx264

Default:

**Auto**

The application must still work when no supported GPU encoder exists.

---

# 38. Final render pipeline

```text
Composition JSON
       ↓
Validate assets
       ↓
Validate timeline
       ↓
Validate audio
       ↓
Normalize incompatible media
       ↓
Create scene filter graph
       ↓
Create caption ASS
       ↓
Apply captions
       ↓
Mix voiceover/BGM/SFX
       ↓
Encode H.264 + AAC
       ↓
Write temporary file
       ↓
Verify output
       ↓
Move to final output
```

Always render to a temporary file first.

Only mark a project “completed” after the output is validated.

---

# 39. Render presets

Provide:

- Draft Preview
- YouTube 1080p
- YouTube 1080p High Quality
- Custom

Default:

```text
1920x1080
30 fps
yuv420p
H.264
AAC 48 kHz
AAC 192 kbps
```

---

# 40. Team dashboard

The team should see:

```text
Jobs
────────────────────────────
#104  Downloading     74%
#105  Scene planning  ✓
#106  Rendering       33%
#107  Completed       ✓
```

Detailed status:

```text
Transcription      ✓
Scene planning      ✓
Stock search        ✓
Downloading         72%
Proxy generation    ✓
Preview ready       ✓
Rendering           0%
```

Use WebSocket/SSE events for live progress.

---

# 41. Project lifecycle

Recommended states:

```text
DRAFT
UPLOADING
TRANSCRIBING
PLANNING_SCENES
FETCHING_STOCK
DOWNLOADING
BUILDING_PREVIEW
READY_FOR_REVIEW
RENDERING
COMPLETED
FAILED
CANCELLED
```

Every state must be recoverable where practical.

---

# 42. Error handling

Errors should be actionable.

Bad:

`500 Internal Server Error`

Good:

`Pexels authentication failed. Check Settings → Stock Providers.`

Actions:

- Open Settings
- Retry
- Use another provider

---

# 43. Logging

Structured logs should include:

- project ID;
- job ID;
- provider;
- worker;
- scene ID;
- render step;
- duration;
- error category.

Never log:

- API secrets;
- unnecessary private audio content;
- full authentication tokens.

---

# 44. Observability

MVP:

- local/server logs;
- per-job log viewer;
- render timing.

Later:

- Sentry;
- aggregate performance metrics;
- worker health dashboard.

Do not add intrusive telemetry without a product need.

---

# 45. Database model

Minimum production entities:

```text
users
teams
team_members
projects
jobs
job_events
assets
project_assets
scenes
transcripts
caption_templates
caption_styles
niche_profiles
provider_accounts
provider_usage
render_outputs
```

Suggested relationships:

```text
Team
 ├── Members
 ├── Provider Accounts
 └── Projects
       ├── Transcript
       ├── Scenes
       │     └── Assets
       ├── Composition Versions
       └── Render Outputs
```

---

# 46. Composition versioning

Every save that materially changes the composition should create a revision.

Example:

```text
v1 = AI generated
v2 = Scene 4 replaced
v3 = Caption style changed
v4 = Scene 9 regenerated
v5 = Final approved
```

This is valuable for team workflows and rollback.

---

# 47. File storage layout

Logical project layout:

```text
projects/
  project_001/
    source/
      voiceover.mp3
    transcript/
      transcript.json
    scenes/
      scene_001.json
      scene_002.json
    assets/
      originals/
      proxies/
      thumbnails/
    captions/
      composition.json
      generated.ass
    renders/
      draft.mp4
      final.mp4
    manifests/
      assets_manifest.json
      generation_manifest.json
    logs/
      project.log
```

---

# 48. Security model

## Desktop users

- authenticate to central API;
- receive only permissions required by role;
- never receive provider master keys if not authorized.

## Admin

Can manage:

- provider credentials;
- AI provider credentials;
- worker limits;
- niche profiles;
- caption templates;
- render defaults.

## Encryption

Use encryption for credentials at rest.

Use HTTPS for all remote API communication.

---

# 49. Roles

Recommended roles:

### Admin

Everything.

### Editor

Create/edit/render projects.

### Reviewer

Preview/comment/approve.

This is optional for the first version but should not be architecturally impossible.

---

# 50. MVP scope

Do not attempt the entire product in one first sprint.

## MVP must include

### Phase 1

Audio upload → transcription → transcript screen.

### Phase 2

Niche → stock search → normalized candidate list.

### Phase 3

Stock clips → voiceover-length timeline → plain HD render.

### Phase 4

Static captions → word-level highlighting.

### Phase 5

Interactive preview → caption controls → scene replacement.

### Phase 6

Team backend → centralized queue → admin settings → production packaging.

---

# 51. Recommended implementation order

The final build order should be:

```text
1. Backend skeleton
2. Audio ingestion
3. ASR
4. Transcript data model
5. Scene planner
6. Stock provider abstraction
7. Stock candidate search
8. Download/cache engine
9. Timeline/composition model
10. Plain FFmpeg render
11. Caption engine
12. Preview player
13. Caption inspector
14. Scene replacement
15. Job queue
16. Team auth
17. Admin settings
18. Project history
19. Packaging/installer
20. QA + production hardening
```

---

# 52. Acceptance tests

The MVP is not production-ready until these pass.

## Test 1 — Duration

Input 60-second voiceover.

Expected:

Final duration approximately 60 seconds within a very small tolerance.

## Test 2 — Multiple topics

Input voiceover contains 3 clearly different topics.

Expected:

visuals change appropriately between subjects.

## Test 3 — Caption style

Change template.

Expected:

preview changes without new stock downloads.

## Test 4 — Typography

Change font/stroke/color.

Expected:

no AI or stock pipeline rerun.

## Test 5 — Scene replacement

Replace scene 7.

Expected:

only scene 7 changes.

## Test 6 — Provider failure

Disable one provider.

Expected:

another enabled provider is attempted.

## Test 7 — Hardware fallback

Disable hardware encoder.

Expected:

CPU export still works.

## Test 8 — Resume

Interrupt a download/render job.

Expected:

completed upstream work is not unnecessarily repeated.

## Test 9 — Cache

Generate another video with an already cached clip.

Expected:

no duplicate download.

## Test 10 — Multi-user

Run multiple jobs simultaneously.

Expected:

provider rate limits remain controlled and jobs do not corrupt one another.

---

# 53. Performance strategy

## Downloading

- async HTTP;
- bounded concurrency;
- connection reuse;
- retries with backoff.

## Transcription

Keep the ASR model warm for batches rather than loading it for every scene.

## AI

Batch compatible scene-analysis requests.

## Preview

Proxy media.

## Rendering

Hardware encode when available.

## Caching

Cache everything deterministic and reusable.

---

# 54. Important architecture rule: business logic location

The desktop UI must remain a UI layer.

Do NOT duplicate these rules in the frontend:

- scene scoring;
- provider selection;
- duration normalization;
- worker scheduling;
- render command construction;
- API-key handling.

Frontend responsibilities:

- display;
- user interaction;
- local editor state;
- sending user changes;
- rendering preview.

Backend responsibilities:

- all business logic;
- AI orchestration;
- asset retrieval;
- rendering orchestration;
- permissions;
- project persistence.

---

# 55. UI screen architecture

## Screen 1 — Dashboard

- recent projects;
- new project;
- active jobs;
- completed renders.

## Screen 2 — Generate

- audio dropzone;
- niche selector;
- advanced settings collapsed;
- Generate button.

## Screen 3 — Processing

- progress stages;
- scene count;
- download progress;
- live event log.

## Screen 4 — Preview Editor

Main areas:

```text
┌────────────────────────────────────┐
│ Video Preview                      │
│                                    │
│            16:9 Video              │
│             + Caption              │
│                                    │
├────────────────────────────────────┤
│ Timeline / waveform                │
├────────────────────────────────────┤
│ Scene strip                        │
└────────────────────────────────────┘
```

Right inspector:

- caption template;
- font;
- colors;
- stroke;
- animation;
- position;
- style controls.

## Screen 5 — Scene Director

- current scene;
- candidate clips;
- search alternatives;
- replace/lock/regenerate.

## Screen 6 — Settings

Tabs:

- AI Providers
- Stock Providers
- Workers
- Rendering
- Captions
- Storage
- Team/Admin

---

# 56. UI design direction

Recommended visual direction:

- professional dark studio interface;
- high contrast;
- restrained glass/blur usage;
- large video preview;
- clear timeline;
- compact inspector;
- keyboard shortcuts;
- obvious processing states.

The UI should be inspired by professional editors, but not attempt to clone CapCut's entire application.

The goal is a specialized “AI YouTube production studio”.

---

# 57. UI design tooling

For the UI design phase, use a dedicated UI design workflow before implementation where practical.

A Figma-based workflow is a good option for:

- screen layout;
- editor states;
- responsive desktop dimensions;
- component library;
- visual QA.

After the design is approved, implement in React/Tailwind/shadcn.

Do not allow a design tool to become the source of truth for business logic.

---

# 58. What should NOT be built in v1

Do not initially build:

- full manual video editor;
- advanced keyframing;
- multi-camera editing;
- complex transitions library;
- automatic YouTube publishing;
- generative video models as a dependency;
- full motion-graphics timeline;
- complicated collaboration comments.

These can be added later.

---

# 59. Future upgrades

Possible v2/v3 features:

- Shorts/Reels 9:16 one-click output;
- automatic thumbnails;
- BGM and audio ducking;
- subtitle SFX;
- brand kits;
- local asset library;
- semantic stock embeddings;
- automatic best-scene re-ranking;
- AI-generated missing B-roll;
- YouTube upload integration;
- team approval workflow;
- batch generation;
- render farm;
- automatic social variants.

---

# 60. Key strategic recommendation

The product should NOT try to compete by being “another CapCut”.

Its competitive advantage is:

```text
Voiceover
   ↓
Understands meaning
   ↓
Automatically finds relevant B-roll
   ↓
Synchronizes the timeline
   ↓
Creates captions
   ↓
Human reviews only where necessary
   ↓
Exports YouTube-ready video
```

The editor exists to correct AI decisions, not to make the user manually build the whole video.

---

# 61. Final technology recommendation

| Layer | Decision |
|---|---|
| Desktop | Tauri 2 |
| Frontend | React + TypeScript + Vite |
| UI | Tailwind + shadcn/Radix |
| State | Zustand + TanStack Query |
| Backend | Python + FastAPI |
| DB | PostgreSQL |
| Dev/local DB | SQLite |
| Queue | Redis + Celery/RQ |
| ASR | faster-whisper / WhisperX |
| LLM | Provider abstraction: OpenAI / Anthropic / Google / local |
| Stock | Provider abstraction: Pexels / Pixabay / custom |
| Preview | HTML5 video + synchronized overlay |
| Captions | JSON templates + ASS/libass |
| Video | FFmpeg |
| GPU | NVENC / AMF / QSV / libx264 fallback |
| Cache | Local + object storage |
| Auth | Central API authentication |
| Realtime | WebSocket/SSE |
| Testing | Pytest + Playwright + media fixtures |
| Monitoring | Structured logs + optional Sentry |

---

# 62. Final product flow

```text
TEAM MEMBER
   │
   ├── Upload Voiceover
   └── Select Niche
           │
           ▼
      CREATE PROJECT
           │
           ▼
      TRANSCRIBE AUDIO
           │
           ▼
      WORD TIMESTAMPS
           │
           ▼
      AI SCENE PLANNER
           │
           ▼
      VISUAL QUERIES
           │
           ▼
      MULTI-PROVIDER SEARCH
           │
           ▼
      CLIP RANKING
           │
           ▼
      PARALLEL DOWNLOAD
           │
           ▼
      CACHE + VALIDATE
           │
           ▼
      COMPOSITION JSON
           │
           ├───────────────┐
           ▼               ▼
      PROXY PREVIEW     CAPTION ENGINE
           │               │
           └───────┬───────┘
                   ▼
              REVIEW / EDIT
                   │
        ┌──────────┴──────────┐
        │                     │
   Replace clip          Change captions
        │                     │
        └──────────┬──────────┘
                   ▼
               APPROVE
                   │
                   ▼
              FFmpeg RENDER
                   │
                   ▼
              OUTPUT VERIFY
                   │
                   ▼
           FINAL YOUTUBE MP4
```

---

# 63. Final decisions summary

## Decision 1

**Tauri 2 + React** for the desktop app.

## Decision 2

**Python + FastAPI** for core backend/media orchestration.

## Decision 3

**Central backend** for production team use.

## Decision 4

**Redis + queue workers** for multi-user concurrency; local asyncio mode can be used during early development.

## Decision 5

**Whisper/faster-whisper/WhisperX** for word timestamps.

## Decision 6

**LLM provider abstraction** for scene planning.

## Decision 7

**Stock provider abstraction** for Pexels, Pixabay and future APIs.

## Decision 8

**Composition JSON** is the shared source of truth for preview and final render.

## Decision 9

**ASS/libass + FFmpeg** for final animated captions.

## Decision 10

**Proxy preview + original-media final render**.

## Decision 11

Manual override is mandatory: replace, lock and regenerate individual scenes.

## Decision 12

Do not promise literal 1:1 visual matching; optimize for semantic relevance and make weak matches visible.

## Decision 13

Track provider/license metadata for every stock asset.

## Decision 14

Keep the normal UX extremely simple: **Audio + Niche → Generate**.

---

# 64. Recommended first coding milestone

Do not start by building the beautiful editor.

Start with this thin vertical slice:

```text
Upload audio
    ↓
Transcribe
    ↓
Generate 5–10 scene plan entries
    ↓
Search Pexels + Pixabay
    ↓
Download top clip per scene
    ↓
Build timeline
    ↓
Render 1080p without captions
    ↓
Show result
```

Once this reliably works, add:

```text
word timestamps
→ captions
→ preview
→ caption inspector
→ scene replacement
→ queue
→ team backend
```

This minimizes the risk of spending weeks building UI before proving the core video-generation pipeline.

---

# 65. Engineering rule for the entire project

**Backend owns the truth. Frontend owns the experience. FFmpeg owns the final pixels. AI proposes decisions. Humans can override them.**

That separation should remain intact throughout the project.
