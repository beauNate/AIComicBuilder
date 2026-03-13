# Google Veo Video Model Integration Design

**Date**: 2026-03-13
**Status**: Approved

## Overview

Integrate Google's Veo video generation models (via Gemini API) into AIComicBuilder as a new `VideoProvider`, reusing the existing `gemini` protocol. Users configure a Gemini provider with video capability and select a Veo model to generate shot videos from first/last frame pairs.

## Architecture

### New File: `src/lib/ai/providers/veo.ts`

A standalone `VeoProvider` class implementing the `VideoProvider` interface, parallel to the existing `SeedanceProvider`.

```
VeoProvider implements VideoProvider
  constructor({ apiKey, baseUrl?, model?, uploadDir? })
  generateVideo({ firstFrame, lastFrame, prompt, duration, ratio }) → Promise<string>
```

**Internal flow:**
1. `clampDuration(duration)` — maps any integer to nearest of `[4, 6, 8]`
2. Read `firstFrame` and `lastFrame` files → base64 `{ imageBytes, mimeType }`
3. `ai.models.generateVideos({ model, prompt, image: firstFrameData, config: { lastFrame: lastFrameData, durationSeconds, aspectRatio } })`
4. Poll `ai.operations.getVideosOperation()` every 10s, max 60 attempts (10 min timeout)
5. `ai.files.download(generatedVideos[0].video, { downloadPath })` → save to `uploads/videos/<ulid>.mp4`
6. Return local file path

### Modified File: `src/lib/ai/provider-factory.ts`

Add `gemini` case to `createVideoProvider`:

```typescript
case "gemini":
  return new VeoProvider({ apiKey, baseUrl, model });
```

No other files require changes.

## Supported Models

| Model ID | Notes |
|---|---|
| `veo-2.0-generate-001` | Default, stable |
| `veo-3.1-generate-preview` | Latest, supports audio, reference images |
| `veo-3.1-fast-generate-preview` | Speed-optimized |

## Parameters

| Parameter | Handling |
|---|---|
| `duration` | Clamped to nearest of 4/6/8 seconds |
| `ratio` | `"16:9"` / `"9:16"` passed through; anything else defaults to `"16:9"` |
| `firstFrame` / `lastFrame` | Read from local filesystem, sent as `{ imageBytes, mimeType }` |

## Error Handling

- **Timeout**: 60 × 10s poll attempts → throws `"Veo generation timed out after 10 minutes"`
- **API failure**: Non-ok responses throw with status + body
- **Generation failure**: Operation state `FAILED` throws with error detail
- **Missing video**: No `generatedVideos[0]` throws `"No video returned from Veo"`

## User Configuration

No UI changes required. Users add a Provider in Settings with:
- Protocol: `gemini`
- Capability: `video` (plus optionally `text`, `image`)
- API Key: Gemini API Key
- Model: one of the Veo model IDs above

The existing model selection UI and `resolveVideoProvider` plumbing handle the rest.

## Out of Scope

- Veo 3.1 audio generation (no audio pipeline in the project)
- Video extension (appending clips)
- Reference images beyond first/last frame
- UI changes to model settings
