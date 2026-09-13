# AI YouTube Video Generator — Detailed Product & Engineering Plan

## 1. Product Vision

Build a Windows desktop application for a content-production team that turns only two primary inputs into a finished YouTube video:

1. **Voiceover / Audio**
2. **Niche / Topic context**

The application automatically:

- transcribes the voiceover;
- understands what is being said;
- creates time-aligned visual scenes;
- searches multiple stock-video APIs;
- downloads multiple candidate clips in parallel;
- selects the most relevant clips;
- builds an exact-duration 16:9 timeline;
- generates animated captions;
- provides a live/editable preview before rendering;
- exports a YouTube-ready Full HD video.

### Primary target output

- Resolution: **1920 × 1080 (16:9)**
- Codec: H.264
- Audio: AAC
- Typical FPS: 30 fps
- Audio duration = final video duration
- Stock footage changes are synchronized to the voiceover meaning, not randomly placed.

The supplied reference video is approximately 90.4 seconds and uses a vertical 9:16 composition. The new product should preserve the useful workflow ideas from the reference but use **16:9 Full HD as the primary YouTube format**.

---

# 2. Core User Experience

The team member should not need to operate a complicated video editor for ordinary jobs.

## Simple workflow

```text
Create Project
    ↓
Upload Voiceover
    ↓
Enter / Select Niche
    ↓
Generate
    ↓
Automatic transcription
    ↓
Automatic scene planning
    ↓
Parallel stock-video search/download
    ↓
Automatic visual matching
    ↓
Caption generation
    ↓
Preview Editor
    ↓
Adjust captions / replace clips if needed
    ↓
Render Full HD
    ↓
Save final MP4
```

## User input

The normal generation screen should contain only:

- Voiceover upload
- Niche input/select
- Generate button

Optional advanced controls can remain collapsed.

Example:

```text
Voiceover
[ upload_mp3.wav ]

Niche
[ Luxury Real Estate ]

                    [ Generate Video ]
```

---

# 3. Product Principles

## Principle 1 — Voiceover is the timeline source of truth

The final visual timeline must be based on the actual voiceover duration and timestamps.

If audio is 03:42, final video must be 03:42 unless the user explicitly changes the project.

## Principle 2 — Visuals must match meaning

Do not simply search the niche and place random stock footage.

The application should understand phrases such as:

> "Dubai is becoming one of the world's biggest business hubs."

and generate visual intent such as:

```text
Dubai skyline
business district
modern skyscrapers
corporate offices
business people
```

## Principle 3 — AI decides; human can override

Automatic generation should be the default, but every scene should be replaceable manually.

## Principle 4 — Preview and final render must use the same composition model

A composition JSON should define the project. Both the preview and final renderer should consume the same model so that the exported result closely matches the preview.

## Principle 5 — Stock providers must be pluggable

Pexels, Pixabay, and future providers must be adapters behind a common interface. Provider-specific code must not be spread throughout the application.

---

# 4. Recommended Technology Stack

## Desktop shell

### Tauri 2

Recommended over Electron for the first production version.

Why:

- smaller application footprint;
- native Windows integration;
- good security model;
- React frontend can be retained;
- easy access to a local backend/process;
- suitable for a team desktop tool.

Electron remains a valid alternative if the project later requires a fully JavaScript-centric runtime or Electron-specific integrations.

### Frontend

- React
- TypeScript
- Vite
- Tailwind CSS
- shadcn/ui or Radix primitives
- Zustand for local UI/editor state
- TanStack Query for backend/query state
- React Hook Form + Zod for forms and validation

---

# 5. Backend / Engine Stack

## Python

Use Python for the AI/media orchestration layer because the video-analysis ecosystem is strong and it integrates naturally with speech-to-text and machine-learning tooling.

### API/service layer

**FastAPI**

Responsibilities:

- project commands;
- generation jobs;
- settings;
- provider configuration;
- preview composition generation;
- render requests;
- status/events;
- worker coordination.

The FastAPI service can run locally with the desktop application.

---

# 6. Video Processing Stack

## FFmpeg

FFmpeg remains the core media engine.

Use it for:

- audio normalization;
- video trimming;
- scaling;
- cropping;
- concatenation;
- transitions;
- overlays;
- caption rendering;
- audio/video muxing;
- final encoding;
- thumbnail extraction;
- waveform generation where useful.

## Hardware acceleration

Use hardware encoders when available.

Priority:

```text
NVIDIA → NVENC
AMD    → AMF
Intel  → QSV
Fallback → libx264
```

The application should detect available encoders at startup.

Do not make hardware acceleration a hard dependency. The tool must still work on CPU-only machines.

---

# 7. Captions / Subtitles Technology

Use a timestamped word/segment representation internally.

Recommended rendering strategy:

### ASS/libass + FFmpeg

ASS is useful for:

- font styling;
- stroke/outline;
- shadow;
- positioning;
- karaoke effects;
- word highlighting;
- timing;
- animated subtitle effects.

For more advanced motion graphics, introduce a second composition layer later rather than forcing every visual effect through subtitles.

---

# 8. Speech-to-Text

## Recommended starting point

**faster-whisper** for local transcription.

For higher-quality alignment, consider:

**WhisperX**

Pipeline:

```text
Audio
 ↓
Voice Activity Detection
 ↓
Whisper transcription
 ↓
Word alignment
 ↓
Word timestamps
 ↓
Sentence / phrase segmentation
```

The final internal representation should retain both segment-level and word-level timestamps.

Example:

```json
{
  "text": "Dubai is becoming a major business hub",
  "start": 12.40,
  "end": 16.90,
  "words": [
    {"word": "Dubai", "start": 12.40, "end": 12.92},
    {"word": "is", "start": 12.92, "end": 13.12}
  ]
}
```

---

# 9. AI / Language Model Layer

Use an LLM only where semantic reasoning adds value.

Recommended jobs for the LLM:

- identify scene boundaries;
- summarize each scene;
- generate stock-video search queries;
- generate alternate search queries;
- detect proper nouns;
- interpret niche context;
- classify visual intent;
- flag scenes where stock footage is likely to be weak.

Do NOT use the LLM for deterministic media processing such as trimming, rendering, muxing, or duration calculation.

A recommended architecture is:

```text
Whisper → timestamps
        ↓
LLM → visual scene plan
        ↓
Stock provider adapters
        ↓
Deterministic media engine
```

---

# 10. Scene Planning System

This is the most important intelligence layer.

## Input

```text
Voiceover transcript
Word timestamps
Niche
Project format
```

## Output

A structured scene plan.

Example:

```json
{
  "scenes": [
    {
      "id": "scene_001",
      "start": 0.0,
      "end": 4.8,
      "spoken_text": "Dubai is becoming a global business hub.",
      "visual_intent": "Dubai modern skyline business district",
      "queries": [
        "Dubai skyline",
        "Dubai business district",
        "modern Dubai skyscrapers"
      ],
      "preferred_orientation": "landscape",
      "importance": 0.94
    }
  ]
}
```

---

# 11. Scene Duration Logic

Do not change footage every few words unless the content clearly demands it.

A better approach is semantic segmentation.

Example:

```text
Voiceover:
"Dubai has become one of the world's most attractive business destinations, attracting thousands of investors every year."
```

Possible timeline:

```text
0.0 - 3.8  → Dubai skyline
3.8 - 7.5  → business district
7.5 - 11.3 → investors / business meeting
11.3 - 14.1 → modern offices
```

The exact scene boundaries should be AI-generated and then normalized to the voiceover's real duration.

---

# 12. Stock Video Provider Architecture

Use a provider abstraction.

```text
StockProvider
 ├── PexelsProvider
 ├── PixabayProvider
 ├── CustomProvider
 ├── FutureProvider
 └── LocalLibraryProvider
```

Common interface:

```python
class StockProvider(Protocol):
    async def search(query: str, options: SearchOptions) -> list[StockAsset]: ...
    async def download(asset: StockAsset, destination: Path) -> Path: ...
```

Every provider should return a normalized asset object.

Example:

```json
{
  "provider": "pexels",
  "asset_id": "12345",
  "url": "...",
  "width": 1920,
  "height": 1080,
  "duration": 15.2,
  "fps": 30,
  "orientation": "landscape",
  "license_url": "..."
}
```

---

# 13. Stock Search Strategy

For each scene, generate several queries rather than one.

Example:

```text
Primary:
Dubai business district

Secondary:
Dubai corporate skyline

Alternative:
UAE business towers

Context:
Middle East investment
```

Search providers in parallel.

Then merge candidates into a common pool.

---

# 14. Stock Clip Scoring

Do not select the first result.

Use a ranking function approximately like:

```text
Final Score =
    Semantic Relevance
  + Niche Relevance
  + Resolution Score
  + Landscape Score
  + Duration Fit
  + Motion Quality
  + Freshness / Variety
  - Duplicate Penalty
  - Low Quality Penalty
```

Example weighted model:

```text
Semantic relevance     35%
Niche/context          15%
Quality/resolution     15%
Duration fit           10%
Landscape suitability  10%
Motion/composition     10%
Variety                 5%
```

The weights should be configurable later.

---

# 15. Avoiding Repetitive Footage

The engine should remember selected assets at project level.

Avoid:

- same clip twice;
- nearly identical clips repeatedly;
- repeated skyline shots within a short period;
- identical stock-provider results in consecutive scenes.

Use:

- asset IDs;
- perceptual hashes/thumbnails;
- scene category memory;
- cooldown windows.

---

# 16. Parallel Download Worker System

The application should never download every asset sequentially.

Recommended design:

```text
Generation Job
      ↓
Task Queue
      ↓
 ┌────┼────┬────┬────┐
 ↓    ↓    ↓    ↓    ↓
W1   W2   W3   W4   W5
 ↓    ↓    ↓    ↓    ↓
Downloads / validation / caching
```

Recommended first implementation:

- asyncio for API requests;
- bounded concurrency;
- provider-specific rate limits;
- retry with exponential backoff;
- local download cache.

A heavier Redis + Celery/RQ architecture should be introduced only if the workload actually requires it.

For a local desktop team tool, **asyncio + internal job queue is the better MVP** because it avoids unnecessary operational complexity.

---

# 17. Worker Recommendation

## MVP

```text
FastAPI
 + asyncio
 + bounded worker pool
 + local job queue
```

## Scale-up option

```text
FastAPI
 + Redis
 + Celery/RQ
 + dedicated render/download workers
```

Do not start with Redis unless the product genuinely needs distributed machines or very high concurrent workloads.

---

# 18. Local Cache

Every downloaded stock file should be cached.

Suggested structure:

```text
cache/
  stock/
    pexels/
    pixabay/
  previews/
  thumbnails/
  transcripts/
  renders/
```

If the same source asset is requested again, the application should not download it again.

Store metadata with cache entries.

---

# 19. Video Normalization Pipeline

Source footage can have many different resolutions, FPS values, codecs, and aspect ratios.

Normalize before final composition.

```text
Source clip
 ↓
Probe metadata
 ↓
Decode
 ↓
Scale/crop to 16:9 safe frame
 ↓
Normalize FPS if required
 ↓
Trim
 ↓
Place on timeline
```

Never stretch a 9:16 clip to 16:9.

Preferred behavior:

- crop;
- intelligently reposition;
- scale to fill;
- optionally use background treatment when crop would destroy important content.

---

# 20. Smart Crop

Start with center crop for MVP.

Later upgrade to:

- face detection;
- object detection;
- saliency detection;
- shot-aware positioning.

For example, if a person is on the right side of the source frame, the crop should preserve them instead of blindly cropping from the center.

---

# 21. Caption System

Caption templates should be data-driven rather than hard-coded UI designs.

Example model:

```json
{
  "id": "bold-highlight",
  "name": "Bold Highlight",
  "fontFamily": "Montserrat ExtraBold",
  "fontSize": 60,
  "fillColor": "#FFFFFF",
  "strokeColor": "#000000",
  "strokeWidth": 5,
  "shadow": true,
  "activeWord": {
    "enabled": true,
    "color": "#FFD400"
  },
  "animation": "pop"
}
```

---

# 22. Caption Editor Controls

At minimum:

### Typography

- font family;
- font weight;
- size;
- case;
- line height;
- letter spacing.

### Colors

- text color;
- active-word color;
- stroke color;
- background color;
- shadow color.

### Stroke

- enabled;
- width;
- opacity.

### Position

- top;
- center;
- bottom;
- X/Y adjustment.

### Animation

- none;
- pop;
- bounce;
- fade;
- slide up;
- slide down;
- typewriter;
- karaoke/highlight.

---

# 23. Word-Level Caption Timing

Internal caption representation must retain word-level timing.

Example:

```json
{
  "words": [
    {"text": "THIS", "start": 3.20, "end": 3.55},
    {"text": "IS", "start": 3.55, "end": 3.70},
    {"text": "THE", "start": 3.70, "end": 3.90},
    {"text": "FUTURE", "start": 3.90, "end": 4.55}
  ]
}
```

This enables active-word animation without manually editing every caption.

---

# 24. Preview Editor

The preview is mandatory before final rendering.

## Main UI concept

```text
┌──────────────────────────────────────────────┐
│ Project                                     │
├──────────────────────────────────────────────┤
│                                              │
│             ┌───────────────────┐            │
│             │                   │            │
│             │       VIDEO       │            │
│             │                   │            │
│             │    CAPTIONS       │            │
│             │                   │            │
│             └───────────────────┘            │
│                                              │
├──────────────────────────────────────────────┤
│ Timeline                                     │
│                                              │
│ VIDEO     ████ █████ ██ █████ ████           │
│ AUDIO     ███████████████████████████████    │
│ CAPTION   ███  █████ ███ ███████ ███         │
└──────────────────────────────────────────────┘
```

Right-side inspector:

```text
Caption Template
Font
Size
Text Color
Stroke
Stroke Width
Active Word Color
Animation
Position
```

---

# 25. Manual Scene Replacement

Every scene should provide:

```text
Replace Clip
```

When clicked:

```text
Current query
[ Dubai business district ]

[ Search ]

Candidate A
Candidate B
Candidate C
Candidate D
```

User selects a replacement without rebuilding the entire project.

---

# 26. Preview Rendering Strategy

Do not render the entire final video every time the user changes a caption setting.

Use a hybrid preview strategy.

### Fast preview

- low-resolution proxy video;
- browser/native video playback;
- captions composited live where possible;
- scene switching represented by the composition timeline.

### Final preview/render

FFmpeg render.

This keeps UI interaction responsive.

---

# 27. Composition JSON

The entire project should be represented by a versioned composition model.

Example:

```json
{
  "version": 1,
  "format": {
    "width": 1920,
    "height": 1080,
    "fps": 30
  },
  "audio": {
    "path": "audio/voiceover.mp3",
    "duration": 163.2
  },
  "scenes": [
    {
      "id": "scene_001",
      "start": 0,
      "end": 4.8,
      "asset": "cache/pexels/123.mp4",
      "crop": {
        "x": 0.12,
        "y": 0.0,
        "scale": 1.0
      }
    }
  ],
  "captions": {
    "template": "bold-highlight",
    "font": "Montserrat ExtraBold",
    "size": 60,
    "color": "#FFFFFF",
    "stroke": {
      "color": "#000000",
      "width": 5
    },
    "animation": "pop"
  }
}
```

The composition model must be versioned so that future releases can migrate old projects safely.

---

# 28. Final Render Pipeline

```text
Project JSON
    ↓
Validate all assets
    ↓
Validate audio
    ↓
Normalize media if needed
    ↓
Construct FFmpeg filter graph
    ↓
Place video scenes
    ↓
Apply crop/scale
    ↓
Apply captions
    ↓
Mix/mux original voiceover
    ↓
Encode H.264 + AAC
    ↓
Write temporary output
    ↓
Verify output
    ↓
Move to final location
```

Always render to a temporary file first.

Only mark the final output as successful after integrity checks pass.

---

# 29. Export Settings

Recommended default:

```text
Container: MP4
Video codec: H.264
Encoder: NVENC / QSV / AMF / libx264
Resolution: 1920x1080
FPS: 30
Pixel format: yuv420p
Audio codec: AAC
Audio sample rate: 48 kHz
Audio bitrate: 192 kbps
```

For YouTube, provide presets such as:

```text
YouTube 1080p
YouTube 1080p High Quality
Draft Preview
Custom
```

---

# 30. Audio Handling

Voiceover should remain the master timing source.

Audio pipeline:

```text
Input audio
 ↓
Probe
 ↓
Normalize if necessary
 ↓
Optional loudness normalization
 ↓
Use as final master audio
```

Do not repeatedly re-encode the voiceover during intermediate processing if avoidable.

---

# 31. Project Folder Structure

Suggested project structure:

```text
projects/
  project-id/
    project.json
    composition.json
    audio/
      voiceover.mp3
    transcript/
      transcript.json
    scenes/
      scene_001.json
      scene_002.json
    assets/
      scene_001.mp4
      scene_002.mp4
    previews/
    renders/
      final.mp4
    thumbnails/
    logs/
```

---

# 32. Database

## Recommended MVP

**SQLite**

No external database server is needed for a desktop-only application.

Store:

- projects;
- settings metadata;
- generation jobs;
- scene metadata;
- assets;
- caption templates;
- render history;
- provider configuration references;
- error logs.

File paths and media remain on disk rather than being stored as database blobs.

---

# 33. Secrets / API Keys

Do not store provider/API secrets in plain JSON.

Preferred approach on Windows:

- Windows Credential Manager / secure OS key storage;
- encryption at rest where applicable;
- never log secrets;
- mask secrets in UI.

Example UI:

```text
Pexels API Key
[ ************** ]
                         [ Test Connection ]
```

---

# 34. Admin / Settings

Settings should include:

## AI

- LLM provider
- model
- transcription model
- API key

## Stock

- provider enable/disable
- API keys
- concurrency limits
- search preferences

## Render

- encoder
- quality
- bitrate
- FPS
- default resolution
- output folder

## Captions

- default template
- default font
- default size
- default colors

## Workers

- download concurrency
- retry count
- timeout
- cache settings

---

# 35. License / Compliance Layer

This should be designed from day one.

For every downloaded stock asset, store:

```text
Provider
Asset ID
Source URL
License/reference URL if supplied
Download date
Project ID
```

This provides internal traceability.

The application should not imply that every provider has identical usage rights. Each provider's current API terms and stock-license requirements must be checked before production use.

---

# 36. Reliability / Failure Handling

Generation should not fail completely because one stock provider failed.

Example:

```text
Pexels unavailable
   ↓
Try Pixabay
   ↓
Try another provider
   ↓
Try another query
   ↓
Use cached/local asset if available
```

Per scene, maintain state:

```text
queued
searching
downloading
validating
selected
ready
failed
```

---

# 37. Retry Strategy

API/network errors should use exponential backoff.

Example:

```text
Attempt 1 → immediate
Attempt 2 → 1 sec
Attempt 3 → 2 sec
Attempt 4 → 5 sec
```

Do not endlessly retry a permanently invalid API key.

Differentiate:

- network error;
- timeout;
- rate limit;
- authentication failure;
- no search result;
- invalid media;
- provider outage.

---

# 38. Quality Gates

Before a scene is accepted:

- file exists;
- file can be decoded;
- duration > minimum threshold;
- resolution meets configured minimum;
- orientation is acceptable;
- codec is decodable;
- no obvious corruption.

Before final export:

- all scene assets exist;
- audio exists;
- composition timeline contains no gaps;
- final duration matches expected tolerance;
- final file decodes successfully.

---

# 39. Timeline Gap Protection

The renderer must guarantee continuous video coverage.

If one clip is shorter than its scene duration:

```text
Option A → select another clip
Option B → extend using another compatible clip
Option C → loop only when visually acceptable
```

Avoid frozen frames by default.

---

# 40. Scene Transition Strategy

MVP:

- hard cuts;
- very short crossfade option.

Later:

- fade;
- whip/push;
- zoom;
- motion transitions.

Do not overload the first release with dozens of transitions. Relevance and synchronization are more important.

---

# 41. Caption Template Library

Create templates as JSON/config, not code.

Example initial pack:

1. Clean White
2. Bold Highlight
3. Yellow Keyword
4. Black Box
5. Minimal Lower Third
6. Karaoke
7. Pop Words
8. Bounce Words
9. Typewriter
10. High Contrast

Later users can save custom presets.

---

# 42. Team Workflow

Potential team workflow:

```text
Team Member
   ↓
Uploads audio
   ↓
Selects niche
   ↓
Generate
   ↓
AI processing
   ↓
Preview
   ↓
Quick corrections
   ↓
Export
   ↓
Upload final MP4 to YouTube workflow
```

For team consistency, use global default caption templates and rendering presets.

---

# 43. Job Progress UI

Generation should show meaningful stages instead of a single spinner.

Example:

```text
✓ Audio analyzed
✓ Transcript generated
✓ 18 scenes planned
✓ 54 stock candidates found
✓ 37 clips downloaded
✓ Scenes matched
● Building preview
○ Final render
```

This is especially important because stock search/download/render can take noticeable time.

---

# 44. Error UI

Errors should be actionable.

Bad:

```text
Error 500
```

Good:

```text
Pexels authentication failed.
Check the API key in Settings → Stock Providers.

[ Open Settings ]
[ Retry ]
```

---

# 45. Logging

Maintain structured logs.

Log categories:

- project;
- AI;
- stock provider;
- downloader;
- renderer;
- preview;
- system.

Never log API secrets or full private user content unnecessarily.

---

# 46. Observability

MVP:

- local log files;
- per-job log viewer.

Later:

- Sentry for crash/error reporting;
- anonymous performance metrics if needed;
- render timing metrics.

Do not add remote telemetry without a clear product requirement.

---

# 47. Performance Recommendations

### Downloading

Use async HTTP and concurrent workers.

### Transcription

Reuse models instead of loading Whisper for every scene.

### Rendering

Use hardware encoding when available.

### Preview

Use proxy media instead of repeatedly encoding full-resolution videos.

### Caching

Cache transcripts, search results, thumbnails and downloaded stock assets.

### AI

Batch compatible scene-analysis requests when the model/provider supports it.

---

# 48. Recommended MVP Scope

Do not build everything in version 1.

## MVP v1

### Input

- audio upload;
- niche input.

### AI

- Whisper/faster-whisper;
- sentence segmentation;
- LLM visual query generation.

### Stock

- two providers initially;
- parallel search/download;
- asset cache.

### Video

- 16:9 1080p;
- smart center crop;
- hard cuts;
- voiceover sync.

### Captions

- 5–10 templates;
- word timestamps;
- font/color/stroke/size controls;
- 3–5 animations.

### Editor

- video preview;
- timeline;
- caption inspector;
- replace clip.

### Export

- MP4;
- H.264;
- AAC;
- hardware acceleration if available.

---

# 49. Version 2 Features

After MVP is stable:

- better semantic clip scoring;
- face/object-aware crop;
- more caption templates;
- custom template editor;
- richer transitions;
- background music layer;
- sound effects;
- automatic intro/outro;
- thumbnail generation;
- brand presets;
- project duplication;
- batch generation.

---

# 50. Version 3 / Scale Features

Potential future productization:

- remote render workers;
- cloud storage;
- team accounts;
- permissions;
- shared template library;
- usage/billing controls;
- API access;
- browser dashboard;
- distributed queue;
- GPU render server;
- automatic YouTube publishing.

---

# 51. Batch Generation

For a content team, this is likely one of the highest-value future features.

Example:

```text
10 voiceovers
10 niches

[ Generate All ]
```

Queue:

```text
Video 01 → processing
Video 02 → processing
Video 03 → queued
...
```

The worker system should be designed so this feature can be added without rewriting the core generation pipeline.

---

# 52. Batch Rendering Architecture

For future scale:

```text
Desktop App
   ↓
Job Manager
   ↓
Render Queue
   ├── Worker 1
   ├── Worker 2
   ├── Worker 3
   └── Worker N
```

Each worker can run a render job independently.

---

# 53. Suggested Repository Structure

```text
ai-video-generator/
│
├── apps/
│   ├── desktop/
│   │   ├── src/
│   │   └── src-tauri/
│   │
│   └── engine/
│       ├── app/
│       ├── api/
│       ├── services/
│       ├── workers/
│       ├── providers/
│       ├── ai/
│       ├── media/
│       ├── captions/
│       ├── projects/
│       └── db/
│
├── presets/
│   └── captions/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
│
├── docs/
│
└── scripts/
```

---

# 54. Frontend Modules

```text
src/
  components/
  features/
    projects/
    generator/
    preview/
    timeline/
    captions/
    settings/
    render/
  stores/
  services/
  types/
  utils/
```

Keep UI modules feature-oriented rather than putting the entire application into one giant component tree.

---

# 55. Backend Modules

Suggested:

```text
engine/app/

  api/
    projects.py
    generation.py
    preview.py
    rendering.py
    settings.py

  ai/
    transcription.py
    scene_planner.py
    query_generator.py

  providers/
    base.py
    pexels.py
    pixabay.py

  media/
    probe.py
    normalize.py
    crop.py
    timeline.py
    render.py

  captions/
    parser.py
    templates.py
    ass_renderer.py

  workers/
    queue.py
    downloader.py
    generation.py

  projects/
    manager.py
    models.py

  db/
    sqlite.py
```

---

# 56. API Contract Example

Generation request:

```http
POST /api/projects/{project_id}/generate
```

Body:

```json
{
  "niche": "Luxury Real Estate",
  "voiceover": "audio/voiceover.mp3"
}
```

Response:

```json
{
  "job_id": "job_abc123",
  "status": "queued"
}
```

Progress:

```http
GET /api/jobs/{job_id}
```

or WebSocket:

```text
/ws/jobs/{job_id}
```

Use WebSocket/SSE-style event updates for the desktop UI.

---

# 57. Job Event Model

Example events:

```json
{
  "type": "scene_planning_progress",
  "completed": 8,
  "total": 18
}
```

```json
{
  "type": "asset_downloaded",
  "scene_id": "scene_008",
  "progress": 0.72
}
```

The frontend should render these events into the progress UI.

---

# 58. Why Not Make FFmpeg Do Everything?

FFmpeg is excellent at deterministic media operations, but it should not become the application architecture.

Do not put:

- business logic;
- AI reasoning;
- provider logic;
- scene planning;
- project management

inside giant FFmpeg command strings.

Instead:

```text
AI / application logic
        ↓
Composition model
        ↓
FFmpeg execution layer
```

This makes the project maintainable.

---

# 59. Why Not Use a Browser-Only Video Editor?

A browser-only editor can work, but this use case is better suited to a desktop application because the application needs:

- local stock-video caching;
- large temporary video files;
- FFmpeg processes;
- GPU encoding;
- local filesystem access;
- predictable performance;
- background generation jobs.

A future cloud edition can reuse much of the backend architecture.

---

# 60. Why Tauri Instead of a Pure Python Desktop GUI?

A Python desktop GUI could work, but React gives a much stronger environment for:

- timeline UI;
- video preview;
- caption inspector;
- settings pages;
- reusable component system;
- responsive editor interactions.

Tauri keeps the desktop shell comparatively lightweight.

---

# 61. Security Considerations

Protect:

- API keys;
- project media;
- voiceover files;
- provider credentials.

Rules:

- never put secrets in frontend source;
- never commit API keys;
- mask secrets in logs;
- use secure OS credential storage;
- validate paths to prevent arbitrary file writes;
- sanitize filenames;
- validate downloaded URLs before writing files;
- use strict timeouts.

---

# 62. Testing Strategy

## Unit tests

Test:

- duration calculations;
- scene segmentation;
- scoring;
- caption timing;
- crop calculations;
- project serialization;
- settings validation.

## Integration tests

Test:

```text
Audio
 ↓
Transcript
 ↓
Scene Plan
 ↓
Mock Stock Provider
 ↓
Composition
 ↓
Render
```

## Media tests

Use known short fixture videos to verify:

- codec support;
- scaling;
- crop;
- audio mux;
- captions;
- duration accuracy.

---

# 63. Important Acceptance Tests

The MVP should pass these before being considered production-ready.

### Test 1

Upload 60-second audio.

Expected:

Final duration ≈ 60 seconds within a very small tolerance.

### Test 2

Use a voiceover containing three clearly different topics.

Expected:

Visual scenes should reflect each topic rather than use one generic stock clip repeatedly.

### Test 3

Switch caption template.

Expected:

Preview updates without regenerating the stock footage.

### Test 4

Change font/stroke/color.

Expected:

No new stock searches required.

### Test 5

Replace one scene clip.

Expected:

Only that scene changes.

### Test 6

A stock provider becomes unavailable.

Expected:

Generation tries another enabled provider or reports a useful recoverable error.

### Test 7

Hardware encoder unavailable.

Expected:

CPU rendering still works.

---

# 64. Performance Targets

These are engineering targets rather than guarantees and should be measured with real hardware.

### Generation

Aim for:

- concurrent stock searching;
- concurrent downloads;
- cached reuse;
- no unnecessary re-transcoding.

### Preview

Aim for near-immediate caption-property updates whenever proxy media is already prepared.

### Export

Use hardware encoding where available to significantly reduce render time.

---

# 65. Cost Control

LLM and API costs can become significant when many videos are generated.

Reduce cost by:

- caching transcription;
- caching scene plans;
- caching search results;
- caching downloaded stock assets;
- using local Whisper where practical;
- using the LLM only for semantic tasks;
- batching scene planning requests when possible.

Do not call an LLM for every individual word or every frame.

---

# 66. Search Optimization

Do not send overly long transcript paragraphs directly as stock searches.

Transform them into concise visual queries.

Bad:

```text
Dubai has become one of the most attractive business destinations because many investors...
```

Better:

```text
Dubai business district
Dubai investors
Dubai corporate skyline
```

---

# 67. Niche-Aware Prompting

The niche should influence visual selection globally.

For example:

```text
Niche: Personal Finance
```

The generic phrase:

```text
"people planning for the future"
```

can be transformed into context-aware searches such as:

```text
financial planning
investment discussion
money management
professional finance meeting
```

This increases semantic consistency.

---

# 68. Proper Noun Handling

The scene planner should detect named entities.

Example:

```text
Elon Musk
Tesla
Dubai
New York
Apple
```

Do not over-generalize these into unrelated generic stock queries.

The system can mark them as:

```text
entity_type = person/company/place
```

and generate specific searches where provider availability allows.

---

# 69. No-Stock Fallback

Some concepts will not have good stock footage.

Fallback options:

1. alternate stock query;
2. second provider;
3. previous/next contextually related clip;
4. visual background with stronger caption treatment;
5. optional generated graphic/visual in a future version.

Never block the entire video because a single scene has weak stock availability.

---

# 70. Future AI Visual Generation

Do not make image/video generation a dependency of the MVP.

However, design the scene asset layer so a scene can eventually be sourced from:

```text
Stock Video
Local Video
Generated Image
Generated Video
Motion Graphic
```

This allows the product to evolve beyond stock footage later.

---

# 71. YouTube-Oriented Features for Later

Useful future additions:

- automatic thumbnail generation;
- title ideas;
- description generation;
- chapter generation;
- tags;
- subtitle file export;
- YouTube upload integration.

These should remain separate from the core renderer.

---

# 72. Suggested Development Order

## Phase 1 — Foundation

- Tauri + React project;
- Python/FastAPI engine;
- SQLite;
- project manager;
- basic settings;
- FFmpeg detection;
- hardware encoder detection.

## Phase 2 — Audio Intelligence

- audio import;
- transcription;
- word timestamps;
- transcript viewer;
- scene segmentation.

## Phase 3 — Stock Pipeline

- Pexels adapter;
- Pixabay adapter;
- normalized asset model;
- async search;
- parallel download;
- cache;
- clip scoring.

## Phase 4 — Timeline Engine

- scene duration management;
- normalization;
- crop;
- composition JSON;
- basic FFmpeg renderer.

## Phase 5 — Captions

- caption data model;
- templates;
- ASS generator;
- word highlighting;
- animations;
- preview controls.

## Phase 6 — Preview Editor

- video player;
- timeline;
- inspector;
- clip replacement;
- project save/load.

## Phase 7 — Production Hardening

- retries;
- logging;
- recovery;
- validation;
- error states;
- installer;
- test suite.

---

# 73. Recommended MVP Screen Map

```text
Dashboard
   ├── New Project
   ├── Recent Projects
   └── Settings

New Project
   ├── Voiceover
   ├── Niche
   └── Generate

Generation
   ├── Progress
   ├── Scene Status
   └── Logs

Editor
   ├── Preview
   ├── Timeline
   ├── Scenes
   └── Caption Inspector

Export
   ├── Preset
   ├── Output Location
   └── Render

Settings
   ├── AI
   ├── Stock Providers
   ├── Captions
   ├── Rendering
   └── Workers
```

---

# 74. UX Recommendation

Do not expose the complexity of the backend to the team member.

The front page should feel close to:

```text
┌─────────────────────────────────┐
│      AI YouTube Video Maker     │
│                                 │
│ Voiceover                       │
│ [ Drop audio here ]             │
│                                 │
│ Niche                           │
│ [ Luxury Real Estate       ]    │
│                                 │
│       [ GENERATE VIDEO ]        │
│                                 │
└─────────────────────────────────┘
```

Advanced settings should be separated.

---

# 75. Important Design Decision — Keep Backend Logic Centralized

The React/Tauri frontend must remain the UI layer.

Do not duplicate:

- stock selection logic;
- AI prompt generation;
- rendering logic;
- project business rules

in the frontend.

The backend/engine remains the single source of truth.

Frontend communicates through typed APIs/events.

---

# 76. Recommended Internal Data Flow

```text
                ┌───────────────┐
                │   Voiceover   │
                └───────┬───────┘
                        ↓
                ┌───────────────┐
                │ Transcription │
                └───────┬───────┘
                        ↓
               ┌────────────────┐
               │ Scene Planner  │◄──── Niche
               └───────┬────────┘
                       ↓
            ┌──────────────────────┐
            │ Stock Query Generator│
            └──────────┬───────────┘
                       ↓
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
     Pexels         Pixabay       Custom API
        │              │              │
        └──────────────┼──────────────┘
                       ↓
                Candidate Assets
                       ↓
                 Quality Scoring
                       ↓
                  Scene Matcher
                       ↓
                 Composition JSON
                       │
             ┌─────────┴─────────┐
             ↓                   ↓
         Preview             FFmpeg Render
             │                   │
             └─────────┬─────────┘
                       ↓
                  Final MP4
```

---

# 77. Recommended Initial Provider Set

Start with **two stock providers**, not five or ten.

Recommended first targets:

- Pexels
- Pixabay

Then add other providers through the same adapter contract.

The important capability is not the number of APIs; it is the abstraction that allows additional providers to be added without changing scene planning or rendering.

---

# 78. Recommended Caption Editor Priority

Build these controls first:

```text
1. Template
2. Font
3. Size
4. Text color
5. Stroke color
6. Stroke width
7. Active word color
8. Animation
9. Position
```

Do not start with 100 typography settings.

Once the core editor works, add advanced controls.

---

# 79. Important Rendering Rule

Do not concatenate incompatible source clips blindly.

Each source must be normalized or handled in a controlled filter graph.

A common rendering strategy is:

```text
Per-scene clip
 ↓
scale/crop
 ↓
trim
 ↓
setpts
 ↓
format
 ↓
concat/composition
```

Then:

```text
Final video
+
Original voiceover
+
Caption layer
```

---

# 80. FFmpeg Filter Graph Strategy

Keep filter graph generation in a dedicated Python module.

Example conceptual pipeline:

```text
[scene0] → scale/crop → trim → setpts → [v0]
[scene1] → scale/crop → trim → setpts → [v1]
[scene2] → scale/crop → trim → setpts → [v2]

[v0][v1][v2] → concat → [basevideo]

[basevideo] + ASS captions → [captionvideo]

[captionvideo] + voiceover → final.mp4
```

For many scenes, dynamically generate this graph rather than writing scene-specific commands manually.

---

# 81. Preview vs Export Quality

Preview can use:

```text
1280x720 proxy
```

or another lower-resolution proxy to improve editor responsiveness.

Final export remains:

```text
1920x1080
```

Preview quality must still be sufficient to judge:

- visual timing;
- caption placement;
- font;
- stroke;
- animation.

---

# 82. Project Recovery

A crash should not destroy the project.

Autosave:

- composition JSON;
- transcript;
- caption settings;
- scene selections.

Use atomic writes / temporary files when saving project state.

The user should be able to reopen a partially generated project.

---

# 83. Cancel / Resume

Long generation tasks should support:

- cancel;
- resume;
- retry failed scene;
- skip scene;
- regenerate scene.

Example:

```text
Generation stopped at Scene 14/22

[ Resume ]
[ Restart Failed Scenes ]
[ Cancel ]
```

---

# 84. Scene Regeneration

One of the highest-value controls:

```text
Regenerate Scene
```

It should regenerate only the selected scene's search/selection, not the entire project.

This saves:

- time;
- API usage;
- download bandwidth;
- render time.

---

# 85. Recommended Future Brand Presets

Allow a preset like:

```text
Channel: Finance Daily

Caption template: Bold Highlight
Font: Montserrat ExtraBold
Active word: Yellow
Stroke: Black 5px
Position: Bottom Center
Default intro: 2.5 sec
Default outro: 2 sec
```

Then a team member can generate consistently branded videos without manually configuring each one.

---

# 86. What Should NOT Be in the First Version

Avoid initially building:

- full CapCut-style nonlinear editing suite;
- complex audio mixing workstation;
- advanced color grading;
- dozens of transitions;
- cloud collaboration;
- user billing;
- social publishing integrations;
- remote render farm.

These increase complexity without improving the central workflow enough.

The core product is:

**Voiceover → relevant visuals → captions → preview → Full HD render**

---

# 87. Recommended Final Architecture

```text
┌────────────────────────────────────────────┐
│                Tauri Desktop               │
│                                            │
│ React + TypeScript + Tailwind + Zustand    │
│ Preview + Timeline + Caption Inspector     │
└──────────────────┬─────────────────────────┘
                   │ HTTP/WebSocket
                   ↓
┌────────────────────────────────────────────┐
│                FastAPI Engine               │
├────────────────────────────────────────────┤
│ Project Service                            │
│ Generation Orchestrator                    │
│ Scene Planner                              │
│ Provider Manager                           │
│ Caption Engine                             │
│ Render Engine                              │
│ Settings / Secrets                         │
└───────┬───────────────┬────────────────────┘
        │               │
        ↓               ↓
   AI Services      Stock Providers
        │               │
        ↓               ↓
 Whisper/LLM       Pexels/Pixabay/etc.
        │
        └───────────┬───────────┘
                    ↓
             Composition JSON
                    ↓
                 FFmpeg
                    ↓
             Hardware Encoder
                    ↓
                Final MP4
```

---

# 88. Final Recommendation

The strongest approach is **not** to build a generic video editor first.

Build a specialized AI video-production pipeline with a lightweight editor around it.

The key product advantage should be:

```text
Minimal input
      ↓
Strong semantic understanding
      ↓
Relevant stock footage
      ↓
Automatic timing
      ↓
Professional captions
      ↓
Human preview/correction
      ↓
Fast YouTube-ready export
```

The most important engineering investment should go into:

1. **Voiceover-to-scene semantic matching**
2. **Fast parallel stock retrieval and caching**
3. **Word-level caption timing and templates**
4. **Accurate preview/export consistency**
5. **Fast hardware-accelerated rendering**
6. **Recovery/retry so the team can generate many videos reliably**

---

# 89. Suggested Build Milestones

## Milestone A

Desktop shell + project system + FFmpeg detection.

## Milestone B

Audio import + Whisper transcript + timestamps.

## Milestone C

AI scene planner + stock provider adapters.

## Milestone D

Parallel downloads + cache + clip scoring.

## Milestone E

Automatic 16:9 composition + basic rendering.

## Milestone F

Caption templates + word highlighting.

## Milestone G

Preview editor + manual clip replacement.

## Milestone H

Hardware encoding + retries + autosave + production hardening.

## Milestone I

Batch generation and team-oriented features.

---

# 90. One-Sentence Product Definition

> A Windows desktop AI video-production tool that takes a voiceover and niche, automatically creates semantically matched 16:9 stock footage with synchronized animated captions, lets the user preview and adjust the presentation, and exports a YouTube-ready Full HD MP4.

