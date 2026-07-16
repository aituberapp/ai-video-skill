---
name: aituber-api
description: >
  AITuber API skill for AI video creation. Generate videos with AI narration, visuals, and
  captions for YouTube Shorts, TikTok, Instagram Reels, and long-form content. Supports
  AI-generated images, video clips, stock footage, and viral templates like skeleton and
  character styles. Handles the full pipeline: pick a voice, generate a video, poll for
  completion, export to MP4 and download, or publish to connected YouTube, TikTok, and
  Instagram channels.

  TRIGGER when the user wants to: create an AI video, generate a video from a script or
  idea, list or browse AI voices, export a video to MP4, download a rendered video, list
  connected channels, publish or schedule a video to YouTube/TikTok/Instagram, check their
  AITuber subscription or credit balance, or automate video creation.

  DO NOT TRIGGER for: editing existing video files, uploading user-provided video footage,
  live streaming, video transcription or captioning of external files, image generation
  without video context, or anything unrelated to the AITuber platform.
license: MIT
compatibility:
  - claude-code
allowed-tools: Bash(curl:*) WebFetch Read Write
metadata:
  openclaw:
    emoji: "\U0001F3AC"
    homepage: https://aituber.app
    primaryEnv: AITUBER_API_KEY
    requires:
      env:
        - AITUBER_API_KEY
      bins:
        - curl
---

# AITuber API

Create AI videos from a script or idea. From 15-second Shorts to 20-minute long-form content. The API handles voice narration (1,300+ voices, any language), AI-generated visuals, word-synced captions, MP4 export, and publishing to YouTube, TikTok, and Instagram.

**Base URL:** `https://app.aituber.app/api/v1`
**OpenAPI spec:** `https://app.aituber.app/api/v1/openapi.json`
**API docs:** `https://aituber.app/api`

> **Tip:** The endpoint reference and credit costs below are generated from the live API definition, so they match the current API. For interactive exploration and response schemas, use the OpenAPI spec or the docs page.

## Authentication

All endpoints except `GET /voices` require a Bearer token. Create an API key in the AITuber dashboard at https://app.aituber.app/dashboard/api-keys. Keys start with `ak_`.

```
Authorization: Bearer <AITUBER_API_KEY>
```

Store the key in the `AITUBER_API_KEY` environment variable.

## Complete Workflow

AITuber supports two common flows after generation:

- **Export flow:** pick voice, generate video, poll status, export, download
- **Publish flow:** pick voice, generate video, poll status, list channels, publish, poll publication status

### Step 1: Pick a voice

```bash
curl "https://app.aituber.app/api/v1/voices?gender=male&accent=American"
```

No auth required. Returns an array of voice objects. Use the `id` field as `voiceId` when generating, or omit `voiceId` to use the default voice.

### Step 2: Generate a video

From a script (you write the narration):

```bash
curl -X POST "https://app.aituber.app/api/v1/videos/generate" \
  -H "Authorization: Bearer $AITUBER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "script": "The human brain contains roughly 86 billion neurons. Every thought, memory, and emotion is the result of electrical signals racing through this incredible network.",
    "voiceId": "VOICE_ID_FROM_STEP_1",
    "imageStyleId": "cinematic",
    "aspectRatio": "9:16"
  }'
```

From an idea (AI writes the script):

```bash
curl -X POST "https://app.aituber.app/api/v1/videos/generate" \
  -H "Authorization: Bearer $AITUBER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "script": "5 mind-blowing facts about black holes",
    "inputType": "idea",
    "expectedDurationSeconds": 60
  }'
```

Returns `{ "videoId": "uuid", "status": "pending" }`.

**Video types (all use this same endpoint):**

1. **Faceless narration (images)** - Default. AI images with Ken Burns animation. Just send a script or idea.
2. **Faceless narration (video clips)** - AI video clips. Set `mediaType: "video"`.
3. **Stock footage** - Real stock footage matched to narration. Set `mediaType: "stock"`.
4. **Skeleton template** - Viral "what happens if..." X-ray style. Set `templateId: "skeleton"`. The template handles mediaType and style.
5. **Character template** - Character-driven stories. Set `templateId: "character"`, `inputType: "idea"`. Only idea mode (AI writes the script for character consistency).

For visual control in script mode, add instructions in brackets: `[A dark forest at night] The wind howled through the trees.` Each `[bracket]` tells the AI what to show for that scene.

### Step 3: Poll for completion

```bash
curl "https://app.aituber.app/api/v1/videos/VIDEO_ID" \
  -H "Authorization: Bearer $AITUBER_API_KEY"
```

Poll every 5-10 seconds. Generation typically takes 1-3 minutes. Wait until `status` is `completed` (or `failed`).

### Step 4: Export to MP4

```bash
curl -X POST "https://app.aituber.app/api/v1/exports" \
  -H "Authorization: Bearer $AITUBER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "videoId": "VIDEO_ID", "resolution": "1080p" }'
```

Exporting is free. Requires an active paid subscription. Poll `GET /videos/VIDEO_ID` and check `exportStatus` until it is `completed` (typically 30 seconds to 5 minutes).

### Step 5: Download the MP4

```bash
curl "https://app.aituber.app/api/v1/exports/download?videoId=VIDEO_ID" \
  -H "Authorization: Bearer $AITUBER_API_KEY"
```

Returns `{ "url": "https://...", "videoId": "..." }`. The URL is a signed temporary link that expires in 2 minutes. Download immediately.

### Step 6: Publish to connected channels

Publishing requires channels to already be connected in the AITuber dashboard and an active paid subscription with the Publish feature. If the video is not exported yet, the API starts the export automatically (no need to call `POST /exports` first).

List channels:

```bash
curl "https://app.aituber.app/api/v1/channels" \
  -H "Authorization: Bearer $AITUBER_API_KEY"
```

Publish to YouTube, TikTok, and Instagram in one call:

```bash
curl -X POST "https://app.aituber.app/api/v1/publications" \
  -H "Authorization: Bearer $AITUBER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "videoId": "VIDEO_ID",
    "caption": "New short is live",
    "publishNow": true,
    "channels": [
      {
        "channelId": "YOUTUBE_CHANNEL_ID",
        "title": "5 Mind-Blowing Facts",
        "tags": ["facts", "science"],
        "categoryId": "27"
      },
      {
        "channelId": "TIKTOK_CHANNEL_ID",
        "tiktokPrivacyStatus": "public",
        "isAiGenerated": true
      },
      {
        "channelId": "INSTAGRAM_CHANNEL_ID",
        "instagramPlacement": "reels"
      }
    ]
  }'
```

Per-platform channel settings:
- **YouTube:** `title`, `tags`, `categoryId`, `madeForKids`
- **TikTok:** `tiktokPrivacyStatus`, `allowComment`, `allowDuet`, `allowStitch`, `isAiGenerated`
- **Instagram:** `instagramPlacement`, `shareToFeed`

Poll publication status:

```bash
curl "https://app.aituber.app/api/v1/publications/PUBLICATION_ID" \
  -H "Authorization: Bearer $AITUBER_API_KEY"
```

Poll every 10-15 seconds until the upload reaches a stable state such as `published`, `scheduled`, or `failed`.

## Endpoints Reference

<!-- GENERATED:BEGIN endpoints -->
### GET /voices

Returns all available AI voices for video narration, sorted by popularity. **No auth required.**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| gender | string | No | Filter by voice gender. Values: "male", "female", "neutral". |
| accent | string | No | Filter by accent (case-insensitive). Examples: "American", "British", "Australian", "Indian". |
| age | string | No | Filter by age group. Values: "young", "middle_aged", "old". |
| useCase | string | No | Filter by recommended use case (case-insensitive). Examples: "narration", "conversational", "news", "audiobook", "social_media". |
| language | string | No | Filter to voices optimized for a specific language (ISO 639-1 code). Examples: "en", "es", "fr", "hi", "zh". |
| search | string | No | Search voices by name or description (case-insensitive). Examples: "roger", "energetic", "calm". |

### GET /voices/cloned

Returns the voices you cloned in the AITuber dashboard.

### POST /ideas

Generates a list of specific, viral-style video topic ideas for a niche or audience.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| prompt | string | Yes | The niche, audience, or theme to brainstorm for. Example: "space facts for a faceless YouTube Shorts channel". |
| language | string | No | Language for the ideas (ISO 639-1 code like "en", "es", "hi"). Default: "en". |
| count | number | No | How many ideas to generate (5-15). Default: 10. |
| source | `tool_page` \| `inline` | No | Where the request came from, for analytics only. Dashboard use; safe to omit. |
| templateId | string | No | Creation template the request came from, for analytics only. Dashboard use; safe to omit. |

### POST /scripts

Generates 2 distinct narration script variations for a topic, sized to your target duration.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| prompt | string | Yes | The topic or idea to write a script about. Example: "5 mind-blowing facts about the deep ocean". |
| duration | number | Yes | Target video duration in seconds (15-1200). The script length is sized so narration fits this duration. |
| language | string | No | Language for the script (ISO 639-1 code like "en", "es", "hi"). Default: English. |
| source | `tool_page` \| `inline` | No | Where the request came from, for analytics only. Dashboard use; safe to omit. |
| templateId | string | No | Creation template the request came from, for analytics only. Dashboard use; safe to omit. |

### GET /image-styles

Returns every image style you can use as `imageStyleId` in `POST /videos/generate` (when `mediaType` is `images`): the built-in styles plus any custom styles created in the AITuber dashboard.

### GET /caption-styles

Returns every caption style you can use as `captionStyleId` in `POST /videos/generate`: the built-in styles plus any custom styles created in the AITuber dashboard.

### POST /videos/generate

Starts generating a new AI video from a script or idea.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| script | string | Yes | The content for your video. How this field is used depends on `inputType`. **Script mode** (`inputType: "script"`, default): Provide the exact narration text. This is what the voice will speak word-for-word. Must be at least 5 words. Max 30,000 characters (enough for a 20-minute video at normal voice speed). The AI automatically splits it into visual segments and generates matching visuals. **Visual control:** By default, the AI decides what visuals to show for each part of your narration. For more control, add visual instructions in brackets before each narration segment: `[A dark forest at night] The wind howled through the trees. [Glowing eyes peering from shadows] Something was watching.` Each `[bracketed text]` tells the AI exactly what to show for that scene. The text after it is the voiceover. **Idea mode** (`inputType: "idea"`): Provide a short topic or concept. The AI writes a full narration script for you. Keep it under 800 characters. Pair with `expectedDurationSeconds` to control video length. Supports any language. The voice will speak naturally in whatever language the text is written in. |
| inputType | `script` \| `idea` | No | How to interpret the `script` field. - `script` (default): Your text is the exact narration. You control every word that is spoken. - `idea`: You provide a topic and the AI writes an engaging narration script for you. Use `expectedDurationSeconds` to control the target length. |
| mediaType | `images` \| `video` \| `stock` | No | The type of visuals for your video. Each produces a different look and feel. - `images` (default): AI generates a unique image for each segment, displayed with smooth Ken Burns pan/zoom animation. This is the classic "faceless narration video" style used by top YouTube channels. Most popular and cheapest option. Control the look with `imageQuality` and `imageStyleId`. - `video`: AI generates short video clips for each segment. More dynamic and cinematic than images, but costs more credits. Also used internally by the `skeleton` and `character` templates. - `stock`: Automatically finds and matches real stock footage to each segment. Great for news, educational, and documentary-style content. **For most use cases, leave this as default (`images`) unless you are using a template.** When using `templateId`, the template automatically selects the best media type for you, so you do not need to set `mediaType` separately. |
| voiceId | string | No | The voice ID for narration. Browse all 1,300+ available voices and listen to previews at `GET /voices`. If omitted, defaults to "Adam", a deep, natural American male voice. Filter voices by gender, accent, or use case using the `GET /voices` endpoint query parameters. Use the `previewUrl` from each voice to hear a sample before selecting. |
| voiceSpeed | number | No | Narration speed multiplier. Range: 0.7 to 1.2. - `0.7`: 30% slower. Great for educational, meditation, or non-native audiences. - `1.0` (default): Natural speed. - `1.2`: 20% faster. Great for energetic, hype, or fast-paced content. Most creators use values between 0.9 and 1.1. |
| aspectRatio | `9:16` \| `16:9` \| `1:1` | No | Video dimensions. Choose based on where you plan to publish. - `9:16` (default): Vertical/portrait. Best for YouTube Shorts, TikTok, and Instagram Reels. - `16:9`: Horizontal/landscape. Best for standard YouTube videos and presentations. - `1:1`: Square. Best for Instagram feed posts and LinkedIn. |
| expectedDurationSeconds | number | No | Target video duration in seconds. **Required when `inputType` is `"idea"`** so the AI knows how long a script to write. Examples: `30` for a 30-second Short, `60` for a 1-minute video, `180` for a 3-minute video, `600` for a 10-minute video, `1200` for a 20-minute video. **Max varies by template:** - Default faceless template (no `templateId`): **1200 seconds (20 minutes)** - `templateId: "skeleton"` or `templateId: "character"`: **420 seconds (7 minutes)** (these templates have different cost profiles and are not designed for long-form content) Passing a value above the template-specific cap returns a 400 error. Ignored when `inputType` is `"script"` because the duration is determined by the word count. |
| imageQuality | `basic` \| `good` \| `premium` \| `max` | No | Image generation quality tier. Only applies when `mediaType` is `"images"`. Higher quality produces more detailed, accurate images but costs more credits per image. - `basic` (default): 1 credit/image. Fast generation. Good for testing and drafts. - `good`: 6 credits/image. Better detail and accuracy. Good for most published content. - `premium`: 16 credits/image. High detail, very accurate to the script. Great for professional content. - `max`: 30 credits/image. Maximum quality. Best for high-production content. A typical 60-second video has 15-18 images (one every 3-5 seconds), so factor that into credit calculations. |
| imageStyleId | string | No | Visual art style for AI-generated images. Only applies when `mediaType` is `"images"`. Each style applies a consistent aesthetic across all images in your video. **Realistic:** - `photorealistic` (default): Hyperrealistic photography, DSLR quality. - `cinematic`: 35mm film look, dramatic lighting, movie still aesthetic. - `vintage-retro`: 1980s VHS aesthetic, neon colors, synthwave vibes. - `noir`: Classic black and white film noir, dramatic shadows. **Illustrated:** - `3d-pixar`: 3D Pixar-style cartoon, smooth rounded shapes. - `anime`: Japanese anime/manga style, cel-shaded, vibrant colors. - `digital-art`: Professional concept art, Artstation quality. - `comic-book`: American comic book, bold outlines, halftone shading. **Artistic:** - `pencil-sketch`: Detailed graphite pencil drawing on textured paper. - `oil-painting`: Classical oil painting with visible brushstrokes. - `watercolor`: Soft watercolor with translucent color washes. - `pop-art`: Andy Warhol style, bold primary colors. **Modern:** - `kurzgesagt`: Flat vector educational style (like the YouTube channel). - `pixel-art`: Retro 16-bit video game aesthetic. - `minimalist`: Clean, simple, lots of white space. - `claymation`: Stop-motion clay animation, Aardman-inspired. And 11 more styles. Combine with `imageStyleCustom` for fine-tuning. |
| captionStyleId | string | No | Caption (subtitle) visual style. Captions are rendered directly onto the video with word-by-word timing sync. **Built-in styles:** - `wrap-1` (default): Active word highlight with 2-word groups. Most popular style. - `hormozi`: Bold uppercase with yellow highlight on black pill. Alex Hormozi inspired. - `beast`: Bold Bangers font with letter-spacing bounce animation. MrBeast inspired. - `noah`: Bold italic Oswald with colored highlight. - `handwritten`: Organic casual style with handwriting font. Personal and authentic. - `subtitle`: Clean streaming-style subtitles on a dark bar. Professional and readable. - `impact`: Massive bold text, one word at a time. Maximum emphasis. - `pop`: Playful spring animation with bouncy words. Fun and energetic. - `chronicle`: Ancient serif for history, mythology, and epic stories. - `cyber`: Futuristic neon style for sci-fi, tech, and cyberpunk content. - `grit`: Raw marker style for true crime, street, and intense stories. - `luxe`: Elegant serif for luxury, fashion, and celebrity content. - `terminal`: Monospace style for tech, hacker, and AI content. You can also create custom caption styles with your own fonts, colors, and animations via the [AITuber dashboard](https://app.aituber.app/dashboard). Use the custom style ID here. |
| captionsEnabled | boolean | No | Whether to show captions (subtitles) on the video. Default: `true`. Captions are auto-synced word-by-word to the narration. We strongly recommend keeping captions on as they significantly boost engagement, accessibility, and watch time. Set to `false` only for music-only or ambient videos. |
| captionPosition | string | No | Vertical position of captions on the video. - `bottom` (default): Captions at the bottom of the screen. - `center`: Captions in the middle of the screen. - `top`: Captions at the top of the screen. |
| videoQuality | string | No | Video clip generation quality. Only applies when `mediaType` is `"video"`. - `basic`: Fastest generation, lower visual quality. - `good` (default): Good balance of quality and speed. - `premium`: Highest quality video clips. Slower generation. |
| templateId | string | No | Specialized video template that applies a specific visual format and style. Leave empty for standard faceless narration videos (the default). **Available templates:** - `skeleton`: "What happens if..." style educational videos with skeleton/X-ray visuals. Uses AI video generation internally. Popular viral format on YouTube Shorts. Example script: `"What happens if you eat only ice cream for 30 days"`. - `character`: Character-driven animated videos. AI generates a consistent character across all scenes and animates them. Uses AI video generation internally. Example script: `"A robot learns what friendship means on its first day at school"`. **Important:** When you set a template, it automatically handles `mediaType` and visual settings for you. You do not need to set `mediaType`, `imageQuality`, or `imageStyleId` separately. Just provide your `script` (or `inputType: "idea"` with a topic) and the template takes care of the rest. |

### GET /videos

Returns all videos for your organization, sorted newest first.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| limit | number | No | Maximum number of videos to return per page. Default: 50, max: 100. |
| cursor | string (uuid) | No | Pagination cursor: the `id` of the LAST video from the previous page. Returns videos older than that one. Omit for the first page. An empty array means there are no more videos. |

### GET /videos/{id}

Returns details for a single video.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| id | string (uuid) | Yes | The video ID returned from `POST /generate` or `GET /videos`. |

### DELETE /videos/{id}

Permanently deletes a video and its generated assets.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| id | string (uuid) | Yes | The video ID to delete. |

### GET /clip-models

Returns the AI video models available for standalone clip generation, with their capabilities (text-to-video, image-to-video, reference images), supported aspect ratios, resolutions, duration limits, and credit cost per second by resolution.

### POST /clips

Starts generating a single AI video clip (1-15 seconds) from a text prompt, an image, or both.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| title | string | No | Optional clip title. Defaults to the start of the prompt. |
| modelKey | string | Yes | The generation model to use. Get valid keys, capabilities, and per-second costs from `GET /clip-models`. |
| prompt | string | No | What the clip should show. Required for text-to-video models; optional when animating from images. |
| aspectRatio | `16:9` \| `9:16` \| `4:3` \| `3:4` \| `1:1` \| `21:9` | No | Clip dimensions. Check the model's `supportedAspectRatios` from `GET /clip-models`. Default: "16:9". |
| resolution | string | No | Output resolution (e.g. "720p", "1080p"). Check the model's `supportedResolutions`. Higher resolutions cost more credits per second. Default: "720p". |
| durationSeconds | integer | No | Clip length in seconds (1-15, model dependent; check `minDurationSeconds`/`maxDurationSeconds`). Default: 5. |
| firstFrameUrl | string | No | Public image URL to use as the first frame (image-to-video). Only for models with `supportsFirstFrame`. |
| lastFrameUrl | string | No | Public image URL to use as the last frame. Only for models with `supportsLastFrame`. |
| referenceImageUrls | array of string | No | Public image URLs used as style/subject references. Only for models with `supportsReferenceImages`; respect `maxReferenceImages`. |

### GET /clips

Returns your standalone AI clips, newest first.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| limit | integer | No | Maximum clips per page (1-50). Default: 20. |
| cursor | datetime (ISO 8601) | No | Pagination cursor: the `createdAt` of the last clip from the previous page. Omit for the first page. |

### GET /clips/{id}

Returns a clip with its generation status.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| id | string (uuid) | Yes | Clip ID from `POST /clips`. |

### POST /exports

Starts rendering a completed video into a downloadable MP4 file.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| videoId | string (uuid) | Yes | The ID of the video to export. The video must have `status: completed`. |
| resolution | `1080p` \| `4k` | No | Export resolution. - `1080p` (default): Full HD (1920x1080 or 1080x1920 for vertical). Fast rendering. - `4k`: Ultra HD (3840x2160 or 2160x3840 for vertical). Slower rendering, larger file. |

### GET /exports/download

Returns a temporary signed URL to download the rendered MP4 file for a video.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| videoId | string (uuid) | No | The video ID. Finds the latest completed export for this video. |
| exportId | string (uuid) | No | A specific export ID, returned from `POST /exports`. Usually not needed since `videoId` automatically finds the latest export. |

### GET /channels

Returns all connected social media channels (YouTube, TikTok, and Instagram) for your organization.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| platform | `youtube` \| `tiktok` \| `instagram` \| `all` | No | Filter by platform. Use "all" or omit to list all connected channels. |

### POST /publications

Publishes a completed video to one or more connected social media channels (YouTube, TikTok, and Instagram).

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| videoId | string (uuid) | Yes | The video to publish. Must have `status: completed`. |
| sceneExportId | string (uuid) | No | Optional export ID if the video has already been exported. If omitted, an export is triggered automatically. |
| caption | string | No | Description/caption shared across all platforms. Used as YouTube description and Instagram caption. Max 2200 characters. |
| addMadeWithCaption | boolean | No | Add "Made with AITuber, the AI video generator: aituber.app" at the end of the caption. Default: true. The caption is shortened if needed so the total stays within 2200 characters. |
| publishNow | boolean | No | Set to `true` (default) to publish immediately. Set to `false` and provide `scheduledAt` to schedule. |
| scheduledAt | datetime (ISO 8601) | No | ISO 8601 datetime to schedule publication. Must be in the future. Only used when `publishNow` is `false`. |
| channels | array of objects | Yes | One or more channels to publish to. Each entry can include platform-specific settings. |
| channels[].channelId | string (uuid) | Yes | Channel ID from `GET /channels`. Must have `status: connected`. |
| channels[].title | string | No | Video title (YouTube). Max 100 characters. Defaults to the video title from generation. |
| channels[].tags | array of string | No | YouTube tags for search discovery. Max 30 tags, each up to 100 characters. |
| channels[].categoryId | string | No | YouTube category ID. Default: "22" (People & Blogs). Common: "24" Entertainment, "27" Education, "26" Howto & Style, "28" Science & Technology, "20" Gaming, "10" Music, "17" Sports, "1" Film & Animation, "23" Comedy. |
| channels[].madeForKids | boolean | No | YouTube COPPA compliance flag. Default: false. |
| channels[].tiktokPrivacyStatus | `public` \| `friends` \| `private` | No | Privacy setting. Default: "public". |
| channels[].allowComment | boolean | No | Allow comments. Default: true. |
| channels[].allowDuet | boolean | No | Allow duets. Default: true. |
| channels[].allowStitch | boolean | No | Allow stitches. Default: true. |
| channels[].isAiGenerated | boolean | No | Label video as AI-generated on TikTok. Default: false. |
| channels[].instagramPlacement | `reels` \| `stories` \| `timeline` | No | Instagram: where to post. Default: "reels". |
| channels[].shareToFeed | boolean | No | Instagram: also share Reel to feed. Default: true. |

### GET /publications/{publicationId}

Returns the current status of a publication.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| publicationId | string (uuid) | Yes | Publication ID from `POST /publications`. |

### DELETE /publications/{publicationId}

Cancels a future scheduled publication before it goes live.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| publicationId | string (uuid) | Yes | Publication ID to cancel. |

### GET /subscription

Returns your current plan and credit balance.
<!-- GENERATED:END endpoints -->

## Credit Costs

<!-- GENERATED:BEGIN credits -->
| Operation | Cost |
|-----------|------|
| Image (basic) | 1 credit/image |
| Image (good) | 6 credits/image |
| Image (premium) | 16 credits/image |
| Image (max) | 30 credits/image |
| Audio narration | ~50 credits/minute |
| AI video clips (basic/good) | 4 credits/second (~20 per 5s clip) |
| AI video clips (premium) | 10 credits/second (~50 per 5s clip) |
| Stock footage | ~50 credits/minute |
| Export to MP4 | Free |
| Publishing | Free |

A typical 60-second video with basic image quality costs about 67 credits (narration + ~17 images). Before generating, check the balance with `GET /subscription`.
<!-- GENERATED:END credits -->

## Error Handling

| HTTP Status | Meaning |
|-------------|---------|
| 400 | Bad request (check error message for details) |
| 401 | Missing or invalid API key |
| 402 | Not enough credits. Response includes `creditsRequired` and `creditsAvailable` |
| 403 | Feature requires an active paid subscription |
| 404 | Resource not found |

## Full Example (bash)

```bash
# 1. Check credits
curl -s "https://app.aituber.app/api/v1/subscription" \
  -H "Authorization: Bearer $AITUBER_API_KEY" | jq .

# 2. Browse voices
curl -s "https://app.aituber.app/api/v1/voices?gender=female&useCase=narration" | jq '.[0:3]'

# 3. Generate video (omit voiceId to use the default voice)
VIDEO_ID=$(curl -s -X POST "https://app.aituber.app/api/v1/videos/generate" \
  -H "Authorization: Bearer $AITUBER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "script": "Did you know that octopuses have three hearts and blue blood?",
    "imageStyleId": "cinematic",
    "aspectRatio": "9:16"
  }' | jq -r '.videoId')

echo "Video ID: $VIDEO_ID"

# 4. Poll until completed
while true; do
  STATUS=$(curl -s "https://app.aituber.app/api/v1/videos/$VIDEO_ID" \
    -H "Authorization: Bearer $AITUBER_API_KEY" | jq -r '.status')
  echo "Status: $STATUS"
  if [ "$STATUS" = "completed" ] || [ "$STATUS" = "failed" ]; then break; fi
  sleep 10
done

# 5. Export to MP4
curl -s -X POST "https://app.aituber.app/api/v1/exports" \
  -H "Authorization: Bearer $AITUBER_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"videoId\": \"$VIDEO_ID\"}"

# 6. Poll export status
while true; do
  EXPORT_STATUS=$(curl -s "https://app.aituber.app/api/v1/videos/$VIDEO_ID" \
    -H "Authorization: Bearer $AITUBER_API_KEY" | jq -r '.exportStatus')
  echo "Export: $EXPORT_STATUS"
  if [ "$EXPORT_STATUS" = "completed" ] || [ "$EXPORT_STATUS" = "failed" ]; then break; fi
  sleep 10
done

# 7. Download
DOWNLOAD_URL=$(curl -s "https://app.aituber.app/api/v1/exports/download?videoId=$VIDEO_ID" \
  -H "Authorization: Bearer $AITUBER_API_KEY" | jq -r '.url')

curl -L -o video.mp4 "$DOWNLOAD_URL"
echo "Downloaded: video.mp4"
```
