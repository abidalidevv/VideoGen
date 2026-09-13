# Additional Risks & Recommendations — Video Generator Tool

Companion to `youtube-video-generator-spec.md`. Covers points raised after the initial spec that affect whether the tool's output can actually be monetized, plus a few build-level gaps and gotchas.

## 1. YouTube Monetization Policy Risk

YouTube updated its channel monetization policy on July 15, 2025: the "repetitious content" policy was renamed to "inauthentic content" and clarified. (Source: support.google.com/youtube/answer/1311392 — check this page directly for the current wording before making decisions based on this summary.)

**What the policy says, in short:**
- Inauthentic content = mass-produced or repetitive content — content that looks templated with little to no variation across videos, or that's easily replicable at scale.
- This applies to the channel as a whole, not just individual videos — if enough videos on a channel violate it, monetization can be pulled from the entire channel.
- The bar for staying eligible: the average viewer should be able to clearly tell that content differs from video to video. A consistent format/template is fine; what matters is that the *substance* of each video is materially varied.
- This is separate from copyright/licensing — a video can be fully licensed and still fail this policy if it's too repetitive/templated.

**Why this matters for this tool specifically:** voiceover + generic stock footage + a fixed caption template is close to the exact profile this policy targets, especially if a team is producing many videos per niche that end up structurally near-identical.

**Recommendation:** design variety into the pipeline itself, not as an afterthought:
- Randomize/vary b-roll selection and transitions per video, even for the same niche and similar scripts.
- Keep the voiceover scripts themselves substantively original — the tool should automate assembly, not compensate for templated/thin scripts.
- Avoid having multiple team members generate near-identical videos for the same niche in the same run.

## 2. Backend Hosting Gap (not yet decided in the spec)

The centralized backend (job queue, ASR, rendering) needs to actually run somewhere — a VPS or a dedicated machine, not the Electron app itself. For team-scale concurrent transcription and rendering, plan for a machine with solid CPU and, ideally, a GPU (speeds up both Whisper transcription and ffmpeg encoding). Hosting/running cost for this machine should be factored into the project's budget, separate from any stock-API costs.

## 3. Technical Gotcha: Normalize Clips Before Concatenation

Stock clips pulled from different APIs will come in with different frame rates, resolutions, and codecs. Feeding them straight into an ffmpeg concat will produce artifacts or outright failures. Every downloaded clip needs to be normalized (scaled to the target resolution, converted to a consistent fps and pixel format) before assembly. Worth calling out explicitly to whoever builds this — including AI coding tools — since it's a common, easy-to-miss failure point.

## 4. MVP Scope Recommendation: Curated Niches Before Generic NLP

Rather than building full generic keyword-extraction from transcripts on day one, start with a fixed, curated list of 3–4 niches, each with a manually mapped set of search keywords (similar to the reference tool's fixed "Main / Nature" pipeline selector). This keeps Phases 1–3 of the build plan quickly testable, and defers the harder, more fragile generic-matching problem until the core pipeline is already proven to work end to end.
