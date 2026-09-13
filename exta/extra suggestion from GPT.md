# AI YouTube Video Generator — Advanced Recommendations & Production Architecture Addendum

## Purpose

This document is an additional engineering/product specification that complements the main product plan.

It captures the additional recommendations that should be considered before coding the production version of the desktop application, with special focus on:

- semantic voice-to-visual matching;
- fallback visual selection;
- multi-provider stock APIs;
- caching and download performance;
- GPU-aware rendering;
- proxy preview;
- scene-level editing;
- caption template architecture;
- audio waveform editing;
- production/team workflows;
- project reproducibility;
- extensibility and long-term maintainability.

The goal is not simply to create an automated video generator, but a reliable internal production tool for a YouTube content team.

---

# 1. Core Product Differentiator: AI Visual Matching

The most important feature should be **semantic visual matching**.

Do not implement the core workflow as:

```text
Transcript
→ keyword extraction
→ stock search
→ random clip
```

Instead use:

```text
Audio
↓
Speech-to-text
↓
Word-level timestamps
↓
Sentence / phrase segmentation
↓
Scene planning
↓
Visual intent generation
↓
Multi-provider stock search
↓
Candidate ranking
↓
Best clip selection
↓
Timeline placement
```

Example voiceover:

> "Dubai is becoming one of the world's biggest business hubs."

The AI should produce structured visual intent such as:

```json
{
  "subject": ["Dubai", "business district"],
  "visual_concepts": [
    "Dubai skyline",
    "modern skyscrapers",
    "business district",
    "corporate offices"
  ],
  "preferred_shot": "wide cinematic city shot",
  "duration_target": 5.2
}
```

This visual intent is then used to search all configured stock providers.

---

# 2. Scene Planning Must Be Semantic, Not Sentence-by-Sentence

The application should not force a video change at every sentence.

Example:

```text
Voiceover:
"Dubai has become one of the world's most attractive business destinations,
attracting thousands of investors every year."
```

A poor system might create one clip for the entire paragraph.

A better system could create:

```text
00:00–00:04.5 → Dubai skyline
00:04.5–00:08.8 → business district / offices
00:08.8–00:13.2 → investors / business meeting
```

The number of scenes should depend on:

- semantic topic changes;
- sentence boundaries;
- speaking pace;
- available quality footage;
- desired B-roll pace;
- niche profile;
- visual repetition.

The total timeline must still match the audio exactly.

---

# 3. Visual Pace Profiles

Create configurable visual pacing profiles.

```text
Slow
Normal
Fast
Very Fast
```

Suggested defaults:

| Content type | Suggested pace |
|---|---|
| Documentary | Slow |
| Luxury | Slow |
| Finance | Normal |
| Business | Normal |
| Technology | Normal/Fast |
| News | Fast |
| Motivation | Fast |
| Social-style content | Very Fast |

Each profile can influence:

- minimum clip duration;
- preferred clip duration;
- maximum clip duration;
- maximum number of visual changes per minute;
- transition frequency.

---

# 4. Stock Provider Abstraction

All stock APIs should appear to the application as a single provider system.

Recommended architecture:

```text
StockProvider
├── PexelsProvider
├── PixabayProvider
├── CustomProvider
└── FutureProvider
```

Common interface:

```text
search(query, filters)
get_asset(asset_id)
download(asset)
get_metadata(asset)
```

The AI engine should never know provider-specific request formats.

It should simply call:

```text
StockProviderRegistry.search()
```

The registry searches configured providers and returns normalized results.

---

# 5. Provider Result Normalization

Normalize all provider responses into one internal asset structure.

Example:

```json
{
  "provider": "pexels",
  "asset_id": "123456",
  "source_url": "...",
  "download_url": "...",
  "width": 1920,
  "height": 1080,
  "duration": 12.4,
  "fps": 30,
  "creator": "...",
  "license_info": "..."
}
```

This makes future providers easy to add.

---

# 6. Save License and Attribution Metadata

Every downloaded stock asset should retain its source information.

Example:

```json
{
  "provider": "pexels",
  "asset_id": "12345",
  "creator": "Creator Name",
  "source_url": "...",
  "downloaded_at": "...",
  "license_note": "..."
}
```

Do not assume provider rules are identical.

Store enough metadata to support:

- internal auditing;
- attribution generation where needed;
- future content audits;
- re-linking to source assets.

---

# 7. Candidate Scoring System

Do not select the first search result.

Every stock candidate should receive a score.

Suggested scoring model:

```text
Semantic relevance        35%
Niche relevance           15%
Visual quality            15%
Orientation / framing     10%
Duration suitability      10%
Motion quality             5%
Duplicate penalty          5%
Source/provider preference 5%
```

The exact weights should remain configurable.

Example:

```text
Candidate A → 92
Candidate B → 84
Candidate C → 73
```

Candidate A becomes the default selection.

---

# 8. Avoid Visual Repetition

A video should not repeatedly use:

- the same clip;
- almost identical clips;
- the same shot from the same source;
- the same camera angle repeatedly.

Maintain a project-level asset history:

```text
used_assets[]
used_creators[]
used_queries[]
```

Apply penalties when the same or similar assets appear too frequently.

Future improvement: perceptual hashing / embedding similarity for duplicate detection.

---

# 9. Fallback Visual Hierarchy

Exact footage will not always exist.

Use a controlled fallback hierarchy:

```text
1. Exact semantic visual
↓
2. Strongly related visual
↓
3. Niche-related visual
↓
4. Generic contextual visual
↓
5. Compatible cinematic filler
```

Example:

Voiceover:

> "The company lost $4.2 billion this quarter."

Exact stock footage for that sentence may not exist.

Reasonable fallback visuals:

```text
company headquarters
↓
financial market footage
↓
stock market screens
↓
finance / accounting visuals
```

The application must never leave an empty timeline because an exact clip was unavailable.

---

# 10. Low-Confidence Visual Warnings

The visual planner should store a confidence score.

Example:

```text
Visual relevance: 94%
Niche relevance: 90%
Quality: 96%
Overall confidence: 93%
```

If confidence is low:

```text
⚠ Low visual confidence
```

The editor can then offer alternate clips.

This is preferable to silently inserting obviously wrong footage.

---

# 11. Asset Cache

Implement a persistent local asset cache.

Example:

```text
cache/
├── pexels/
├── pixabay/
├── custom/
├── thumbnails/
├── proxies/
└── metadata/
```

Use asset IDs and/or content hashes as cache keys.

If an asset was already downloaded:

```text
API request          ❌
download             ❌
local cache          ✅
```

Benefits:

- lower bandwidth;
- faster project generation;
- fewer API calls;
- reduced provider rate-limit pressure;
- faster regeneration.

---

# 12. Parallel Download Workers

Stock downloading should be concurrent.

Example:

```text
               Job Queue
                   │
       ┌───────────┼───────────┐
       ↓           ↓           ↓
   Worker 1    Worker 2    Worker 3
       ↓           ↓           ↓
   Worker 4    Worker 5    Worker 6
```

However, concurrency must be provider-aware.

Example configuration:

```text
Global workers: 6
Pexels concurrency: 3
Pixabay concurrency: 2
Custom provider: 1
```

Each provider should have its own rate-limit and retry policy.

---

# 13. Retry and Backoff

Network failures should not fail the complete video.

Recommended retry strategy:

```text
Attempt 1 → immediate retry
Attempt 2 → short delay
Attempt 3 → exponential backoff
```

After maximum retries:

```text
Asset download failed
↓
Search next candidate
↓
Continue job
```

The entire generation job should not collapse because one stock URL failed.

---

# 14. Proxy Preview System

Do not preview original 4K stock footage directly for every editing operation.

Generate proxy media.

```text
Original 4K / Full HD
        ↓
    720p proxy
```

Preview uses:

```text
Proxy assets
```

Final export uses:

```text
Original assets
```

Benefits:

- smoother timeline playback;
- lower GPU/CPU load;
- faster seeking;
- faster preview generation.

---

# 15. Hardware-Aware Rendering

The application should detect supported encoders automatically.

Suggested priority:

```text
NVIDIA → NVENC
Intel  → QSV
AMD    → AMF
Fallback → libx264 CPU
```

The user should ideally see:

```text
Encoding: Automatic
Detected: NVIDIA NVENC
```

rather than having to understand FFmpeg encoder flags.

---

# 16. Render Validation

Before final export, run a render validation pass.

Checks:

```text
✓ Audio exists
✓ Audio duration valid
✓ Video duration matches target
✓ No missing scene assets
✓ No timeline gaps
✓ No corrupted files
✓ Caption timings valid
✓ Caption safe-zone valid
✓ Resolution = 1920×1080
✓ Audio track exists
✓ Encoder available
```

If validation fails, show a clear issue instead of starting a long render that will later fail.

---

# 17. Scene-Level Editing

Every scene should be an editable timeline object.

Example:

```text
Scene 01   00:00–00:04
Scene 02   00:04–00:09
Scene 03   00:09–00:14
```

Scene actions:

```text
Replace
Search
Trim
Extend
Delete
Lock
```

---

# 18. Scene Locking

A particularly useful feature for production teams is **Lock Scene**.

Example:

```text
Scene 07 → LOCKED
```

Any later AI regeneration should not alter locked scenes.

This means a team member can manually approve a good visual and protect it from future automatic regeneration.

---

# 19. Regenerate Selected Scenes

Never force users to regenerate the whole project for one bad clip.

Example:

```text
Scenes 1–15  ✓
Scene 16     ✗
Scenes 17–28 ✓
```

Button:

```text
Regenerate Scene 16
```

Only the selected scene should go through visual search and replacement.

---

# 20. Caption Template Engine

Caption styles must be data-driven, not hard-coded.

Example:

```json
{
  "name": "Bold Highlight",
  "fontFamily": "Montserrat ExtraBold",
  "fontSize": 58,
  "fill": "#FFFFFF",
  "stroke": "#000000",
  "strokeWidth": 4,
  "activeWordColor": "#FFD400",
  "position": "bottom-center",
  "animation": "pop"
}
```

Possible template library:

```text
Bold Highlight
Karaoke
Minimal
Word Pop
Bounce
Typewriter
All Caps
Lower Third
Box Highlight
Social Punch
```

Store templates as JSON files or database records so new templates can be added without rewriting UI logic.

---

# 21. Caption Global Settings + Local Overrides

Provide global settings:

```text
Font
Size
Color
Stroke
Stroke width
Shadow
Position
Animation
Active-word color
```

Allow scene-level overrides.

Example:

```text
Global font size = 60

Scene 8 override = 72
```

This gives the team professional control without making the editor complex.

---

# 22. Word-Level Caption Timing

The internal transcript should preserve word timestamps.

Example:

```text
YOU    00.00–00.25
ARE    00.25–00.47
NOT    00.47–00.76
TIRED  00.76–01.20
```

This enables:

- active-word highlighting;
- pop animation;
- karaoke effects;
- word-by-word scaling;
- precise timing;
- better caption animation.

---

# 23. Audio Waveform Timeline

Add a waveform to the editor.

Example concept:

```text
VOICEOVER
▁▂▅▇▆▂▃▆▇▅▂▁▃▆▇
```

Show synchronized caption words below/above it.

This makes it easier to understand speaking rhythm and manually edit scene/caption boundaries.

---

# 24. YouTube 16:9 Composition Rules

Primary output:

```text
1920 × 1080
16:9
```

Source aspect ratio handling:

```text
16:9 → direct
4:3  → crop
9:16 → intelligent crop
1:1  → crop/scale
```

Never stretch footage.

Use smart crop / center-of-interest strategies when possible.

---

# 25. Caption Safe Zones

Captions should not be placed too close to:

- bottom edge;
- sides;
- YouTube UI overlays;
- important faces/subjects.

Add configurable safe-zone visualization in the editor.

Example:

```text
┌────────────────────────────┐
│                            │
│       safe content         │
│                            │
│       [ CAPTION ]          │
│                            │
└────────────────────────────┘
```

---

# 26. Niche Profiles

Create reusable niche profiles.

Examples:

```text
Luxury Real Estate
Finance News
AI News
Motivation
Business
Technology
```

A niche profile can store:

```text
preferred keywords
visual style
visual pace
caption template
font
colors
transition style
fallback preferences
```

Then the ordinary workflow remains:

```text
Upload audio
Select niche
Generate
```

---

# 27. Brand Kit

A future but strongly recommended feature.

Example:

```text
Brand Kit
├── Logo
├── Fonts
├── Caption presets
├── Color palette
├── Intro
├── Outro
└── Watermark
```

The team can create multiple brand profiles if several YouTube channels are managed.

---

# 28. Project Manifest

Every generated project should have a reproducible manifest.

Example:

```text
project.json
```

Store:

```text
project ID
voiceover file
transcript
word timestamps
scene plan
stock assets
provider information
source metadata
caption template
caption overrides
timeline
render settings
brand settings
software/model versions
```

This makes projects reproducible and debuggable.

---

# 29. API Keys and Security

API keys must never be embedded in project JSON or plaintext logs.

UI should show:

```text
••••••••••••••••
```

Use encrypted local storage / OS credential facilities where possible.

Project files should only contain provider identifiers, not secrets.

Example:

```json
{
  "provider": "pexels"
}
```

not:

```json
{
  "api_key": "SECRET"
}
```

---

# 30. Production Job Dashboard

The tool should support multiple simultaneous jobs.

Example:

```text
Video #101   Rendering       44%
Video #102   Downloading    72%
Video #103   Waiting
Video #104   Completed
```

Each job can show:

```text
Transcription       ✓
Scene planning      ✓
Stock search        ✓
Downloading         72%
Caption generation ✓
Preview             ✓
Rendering           0%
```

---

# 31. Job State Machine

Use explicit job states.

Suggested states:

```text
CREATED
QUEUED
TRANSCRIBING
PLANNING_SCENES
SEARCHING_ASSETS
DOWNLOADING_ASSETS
BUILDING_TIMELINE
GENERATING_CAPTIONS
READY_FOR_REVIEW
RENDERING
COMPLETED
FAILED
CANCELLED
```

This is important for reliable recovery and UI progress.

---

# 32. Cancellation and Recovery

A production tool needs:

```text
Pause
Resume
Cancel
Retry
Restart failed step
```

If the application closes during rendering, the project should remain recoverable.

Do not force the team to restart an entire video because one step failed.

---

# 33. Keep AI Providers Replaceable

Do not hard-code the whole AI engine to one vendor.

Recommended abstraction:

```text
AIProvider
├── OpenAIProvider
├── LocalLLMProvider
└── FutureProvider
```

Possible future tasks:

- visual intent generation;
- scene planning;
- query expansion;
- relevance scoring;
- title/description generation;
- quality checks.

This keeps the product portable.

---

# 34. MVP Architecture vs Scaled Architecture

Do not over-engineer the first desktop release.

## MVP

Use:

```text
Tauri
React
TypeScript
Python
FastAPI
SQLite
Local job queue
Async worker threads/processes
FFmpeg
faster-whisper
```

## Scaling phase

When multiple machines or centralized workers are needed:

```text
Redis
Celery / RQ
Central job server
Shared storage
PostgreSQL
```

The MVP should be designed so the worker layer can later be replaced by distributed workers.

---

# 35. Recommended Layered Architecture

```text
                  ┌──────────────────────┐
                  │      Tauri App       │
                  │ React + TypeScript   │
                  └──────────┬───────────┘
                             │
                  ┌──────────▼───────────┐
                  │   Local API Layer    │
                  │       FastAPI        │
                  └──────────┬───────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
   AI Pipeline         Asset Pipeline       Project DB
        │                    │                    │
        ▼              ┌─────┼─────┐              ▼
  Whisper / STT        ▼     ▼     ▼           SQLite
        │            Pexels Pixabay Custom
        ▼
 Scene Planner
        │
        ▼
 Visual Ranking
        │
        ▼
 Timeline Builder
        │
        ▼
 Caption Engine
        │
        ▼
 Preview Composition
        │
        ▼
 FFmpeg Renderer
        │
        ▼
 1920×1080 MP4
```

---

# 36. Recommended Project Folder Structure

Suggested local structure:

```text
app-data/
├── projects/
│   └── PROJECT_ID/
│       ├── project.json
│       ├── audio/
│       ├── transcript/
│       ├── scenes/
│       ├── assets/
│       ├── proxies/
│       ├── captions/
│       ├── previews/
│       ├── renders/
│       ├── logs/
│       └── metadata/
│
├── cache/
│   ├── stock/
│   ├── thumbnails/
│   └── proxies/
│
├── templates/
│   ├── captions/
│   ├── niches/
│   └── brands/
│
└── settings/
```

---

# 37. Composition JSON as Source of Truth

The composition should describe the final video.

Example:

```json
{
  "project_id": "abc123",
  "resolution": "1920x1080",
  "fps": 30,
  "audio": "audio/voiceover.wav",
  "scenes": [
    {
      "start": 0,
      "end": 4.2,
      "asset": "assets/scene-001.mp4"
    },
    {
      "start": 4.2,
      "end": 8.7,
      "asset": "assets/scene-002.mp4"
    }
  ],
  "captions": {
    "template": "bold-highlight",
    "font": "Montserrat ExtraBold",
    "size": 58,
    "color": "#FFFFFF",
    "stroke": "#000000",
    "strokeWidth": 4,
    "animation": "pop"
  }
}
```

The preview system and final renderer should both interpret this same structure.

---

# 38. Preview Architecture

The preview should be fast and responsive.

Recommended concept:

```text
Project Composition JSON
        ↓
Proxy assets
        ↓
Preview player / timeline
        ↓
Live caption overlay
```

Do not require a full final-quality render after every caption setting change.

The editor should update styling in real time as much as practical.

---

# 39. Final Render Pipeline

Final export:

```text
Composition JSON
↓
Original video assets
↓
Scale / crop
↓
Timeline composition
↓
Caption render
↓
Audio normalization
↓
Audio/video mux
↓
H.264 encoding
↓
1920×1080 MP4
```

Recommended default profile:

```text
Container: MP4
Video: H.264
Audio: AAC
Resolution: 1920×1080
FPS: 30
```

Bitrate / CRF should be configurable, but sensible defaults should hide unnecessary complexity from ordinary users.

---

# 40. Audio Handling

Normalize voiceover before composition where necessary.

Recommended pipeline:

```text
Input audio
↓
Decode
↓
Normalize / loudness adjustment
↓
Transcription source
↓
Final mix source
```

Future music layer:

```text
Voice
+
Music
+
SFX
```

with automatic ducking.

Example:

```text
Voice active → music 12%
Voice pause  → music 30%
```

---

# 41. Future Music and SFX Architecture

Keep audio tracks composable:

```text
AudioTrack 1 → Voiceover
AudioTrack 2 → Background music
AudioTrack 3 → Sound effects
```

This allows future expansion without restructuring the timeline model.

---

# 42. Team-Oriented UX

The main dashboard should prioritize production tasks.

Suggested screens:

```text
Dashboard
Projects
New Video
Preview / Editor
Render Queue
Settings
Caption Templates
Niche Profiles
Brand Kits
```

The default generation page should remain extremely simple.

---

# 43. Suggested Dashboard Layout

```text
┌──────────────────────────────────────────────┐
│ AI Video Generator                           │
├──────────────┬───────────────────────────────┤
│ Projects     │ Recent Projects               │
│ New Video    │                               │
│ Queue        │ Video #104    Completed       │
│ Templates    │ Video #103    Rendering       │
│ Brands       │ Video #102    Ready           │
│ Settings     │                               │
└──────────────┴───────────────────────────────┘
```

---

# 44. Error Handling Philosophy

Errors should be task-specific.

Bad:

```text
FFmpeg error code 1
```

Better:

```text
Unable to render Scene 12.
The source video could not be decoded.
[Replace Clip] [Retry]
```

Logs can still contain technical details for debugging.

---

# 45. Observability / Logging

Save structured logs per project.

Example:

```text
logs/generation.log
logs/render.log
logs/download.log
logs/errors.log
```

Useful metrics:

- generation duration;
- number of API searches;
- cache hit rate;
- download failures;
- average scene confidence;
- render time;
- hardware encoder used.

This will help optimize production throughput later.

---

# 46. Performance Targets

Reasonable initial targets:

```text
UI launch: fast
Preview: near-real-time for proxies
Parallel downloads: configurable
Scene generation: async
Final render: hardware accelerated when available
```

The exact performance will depend heavily on:

- audio length;
- number of scenes;
- stock API latency;
- number of worker threads;
- disk speed;
- CPU/GPU;
- encoder.

Do not hard-code unrealistic timing promises.

---

# 47. Testing Strategy

## Unit tests

Test:

- timeline math;
- duration calculations;
- scene splitting;
- provider adapters;
- score calculations;
- caption timing;
- composition validation.

## Integration tests

Test:

```text
Audio
→ Transcription
→ Scene planning
→ Asset selection
→ Composition
→ Render
```

## Golden-media tests

Keep a small set of known audio/video fixtures.

Every major renderer change can render them and compare:

- duration;
- resolution;
- audio presence;
- frame samples;
- caption presence.

---

# 48. Versioning

Record software/model versions in every project.

Example:

```json
{
  "app_version": "1.0.0",
  "planner_version": "1.2",
  "caption_engine_version": "1.0",
  "transcription_model": "..."
}
```

This is useful if the same project must later be reproduced.

---

# 49. Do Not Over-Engineer the First Release

Avoid putting these into the first MVP unless they are required:

- full cloud infrastructure;
- multi-user authentication;
- distributed Kubernetes workers;
- complex collaborative editing;
- heavy cloud rendering;
- dozens of AI agents.

The first production desktop release should optimize for:

```text
Reliable generation
Fast downloads
Good visual matching
Excellent captions
Fast preview
Fast render
Easy team usage
```

---

# 50. Recommended MVP Roadmap

## Phase 1 — Foundation

```text
Tauri
React
TypeScript
FastAPI
SQLite
FFmpeg
Project system
Settings system
```

## Phase 2 — Audio Intelligence

```text
Whisper / faster-whisper
Word timestamps
Transcript viewer
Scene segmentation
```

## Phase 3 — Stock Pipeline

```text
Pexels adapter
Pixabay adapter
Provider registry
Search normalization
Download workers
Caching
Retries
```

## Phase 4 — Visual Intelligence

```text
Visual intent generation
Query expansion
Candidate scoring
Fallback selection
Duplicate avoidance
Confidence score
```

## Phase 5 — Timeline

```text
Scene timeline
Proxy media
Audio waveform
Composition JSON
```

## Phase 6 — Captions

```text
Caption templates
Word highlighting
Animations
Font/style controls
Global/local overrides
```

## Phase 7 — Preview

```text
Live player
Timeline scrubbing
Scene replacement
Caption preview
Lock scene
Regenerate scene
```

## Phase 8 — Final Rendering

```text
FFmpeg pipeline
GPU detection
H.264/AAC
Render queue
Validation
Export
```

## Phase 9 — Production Features

```text
Niche profiles
Brand kits
Job dashboard
Recovery
Logs
Analytics
```

---

# 51. Features Worth Adding Later

## Optional future features

```text
YouTube title generation
Description generation
Tags/keywords
Thumbnail generation
Automatic intro/outro
Background music
SFX
Auto chapter generation
Subtitle SRT/VTT export
9:16 Shorts output
1:1 social output
4K rendering
Cloud worker mode
Centralized team dashboard
```

Do not let these features delay the core 16:9 workflow.

---

# 52. Strong Recommendation: Support Multiple Output Ratios Later

Keep the composition engine resolution-aware.

Initially:

```text
1920×1080
```

Later:

```text
1920×1080 → YouTube
1080×1920 → Shorts / Reels / TikTok
1080×1080 → Square social
```

This should be an output-profile change rather than a complete engine rewrite.

---

# 53. Strong Recommendation: Separate Business Logic from UI

The UI should never contain logic such as:

```text
Which stock provider to call
How to score a clip
How to split scenes
How to build FFmpeg commands
```

The UI should request actions:

```text
Generate Project
Search Assets
Replace Scene
Render Project
```

The backend/engine owns the business logic.

This makes future UI changes much easier.

---

# 54. Strong Recommendation: Keep the Composition Engine Independent

The composition model should not depend on React.

It should be a backend/domain model that could theoretically be consumed by:

```text
Desktop UI
Web UI
CLI
Remote worker
Cloud renderer
```

This becomes a major advantage if the product is later converted into a SaaS application.

---

# 55. Final Recommended Architecture

```text
                        ┌──────────────────────┐
                        │       Tauri          │
                        │ React + TypeScript   │
                        └──────────┬───────────┘
                                   │
                        ┌──────────▼───────────┐
                        │      FastAPI         │
                        │ Local Application API│
                        └──────────┬───────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
        ▼                          ▼                          ▼
  Project Service            Generation Engine          Settings Service
        │                          │
        ▼                          ├───────────────┐
     SQLite                       │               │
                                  ▼               ▼
                           Speech / STT      AI Planner
                                  │               │
                                  └───────┬───────┘
                                          ▼
                                   Visual Intent
                                          │
                                          ▼
                                  Stock Provider
                                    Registry
                                    /   |   \
                                   /    |    \
                               Pexels Pixabay Custom
                                    \    |    /
                                     \   |   /
                                      Candidate Pool
                                           │
                                           ▼
                                      Clip Ranking
                                           │
                                           ▼
                                    Asset Download
                                      Worker Pool
                                           │
                                           ▼
                                       Asset Cache
                                           │
                                           ▼
                                    Timeline Builder
                                           │
                         ┌─────────────────┴─────────────────┐
                         ▼                                   ▼
                   Caption Engine                      Proxy Builder
                         │                                   │
                         └─────────────────┬─────────────────┘
                                           ▼
                                     Preview Editor
                                           │
                                           ▼
                                   Composition JSON
                                           │
                                           ▼
                                  Render Validation
                                           │
                                           ▼
                                    FFmpeg Renderer
                                           │
                                           ▼
                                      GPU Encoder
                                           │
                                           ▼
                                  Final 1920×1080 MP4
```

---

# 56. Three Features That Must Not Be Compromised

## 1. Voice → Visual Semantic Matching

This is the product's central differentiator.

## 2. Excellent Caption Preview / Editor

The team must see exactly how the caption system behaves before rendering.

## 3. Fast Production Pipeline

Parallel search + parallel downloads + caching + proxies + hardware rendering should be built into the architecture from the beginning.

---

# 57. Final Engineering Direction

The correct objective is not:

> Build an FFmpeg tool that puts random stock videos behind audio.

The objective is:

> Build a production-grade desktop video generation system where a voiceover becomes a structured, semantically matched, editable video composition that a team member can preview, correct, and export to YouTube with minimal manual work.

The core abstraction should therefore be:

```text
Audio
→ Understanding
→ Scene Plan
→ Visual Candidates
→ Ranked Assets
→ Timeline
→ Captions
→ Preview
→ Human Approval
→ Render
```

This architecture keeps the system efficient today while leaving a clean path toward a larger SaaS / cloud-rendering platform later.
