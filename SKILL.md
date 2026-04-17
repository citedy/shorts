---
name: citedy-video-shorts
title: "AI Video Shorts"
description: >
  Generate and optionally publish branded AI avatar lip-sync video shorts for
  Instagram Reels and YouTube Shorts using Citedy's live shorts API contract.
  Includes setup, pricing, prompt guidance, API reference, and worked examples.
version: "3.1.1"
author: Citedy
tags:
  - video
  - ai-avatar
  - shorts
  - reels
  - youtube-shorts
  - content-creation
  - lip-sync
metadata:
  openclaw:
    requires:
      env:
        - CITEDY_API_KEY
    primaryEnv: CITEDY_API_KEY
  compatible_with: "citedy-seo-agent@3.5.0"
privacy_policy_url: https://www.citedy.com/privacy
security_notes: |
  The site issues real agent keys prefixed citedy_agent_. Store those values in
  the CITEDY_API_KEY environment variable. CITEDY_API_KEY is only the local env
  variable name used for storage. Keys authenticate only against Citedy API
  endpoints (www.citedy.com/api/agent/*). All traffic is TLS-encrypted.
---

# AI Video Shorts

Use this skill to generate branded talking-head UGC shorts with Citedy and
optionally publish them to connected Instagram Reels or YouTube Shorts accounts.

Base URL: `https://www.citedy.com`

## Overview

This skill covers the full pipeline:

1. Register and approve an agent key
2. Generate a speech script
3. Generate an avatar
4. Generate the video
5. Poll for the final asset
6. Merge only when multiple clips or custom subtitle styling are needed
7. Publish the final asset to connected social accounts

The shortest valid path is important: if polling returns a completed video with
`subtitles_applied: true`, treat that Step 3 asset as the final publish-ready
result and skip merge.

## Operating Rules

This package should behave like a producer, not just an API reference.

- Prefer execution over explanation. Translate the request into the shortest
  path to a finished short instead of dumping the entire pipeline on the user.
- Use a guided flow: `setup -> create -> publish`.
- Ask only for missing high-leverage inputs such as whether to publish now,
  whether to reuse or change avatar identity, and any hard brand constraints.
- If a detail can be safely defaulted, default it.
- Reuse known brand tone and avatar preferences when you are confident instead
  of re-asking every time.
- Poll generation silently and only surface meaningful progress.

## Producer Defaults

Unless the user says otherwise:

- `duration`: `10`
- `style`: `hook`
- `language`: `en`
- `aspect_ratio`: `9:16`
- `resolution`: `480p`
- `publish`: `false`

## Source Of Truth

This skill must follow the live backend contract in:

- `app/api/agent/register/route.ts`
- `app/api/agent/me/route.ts`
- `app/api/agent/shorts/avatar/route.ts`
- `app/api/agent/shorts/route.ts`
- `app/api/agent/shorts/[id]/route.ts`
- `app/api/agent/shorts/merge/route.ts`
- `app/api/agent/shorts/publish/route.ts`

If any example below drifts, follow the backend routes instead.

## Live Contract Notes

- The site returns real API keys prefixed `citedy_agent_...`.
- `CITEDY_API_KEY` is only the env variable name where that real key is stored.
- `POST /api/agent/register` returns `pending_id`, `approval_url`, and
  `expires_in`. It does not return the API key itself.
- Avatar enums are strict:
  - `gender`: `male | female`
  - `origin`: `european | asian | african | latin | middle_eastern | south_asian`
  - `age_range`: `18-25 | 26-35 | 36-50`
  - `type`: `tech_founder | vibe_coder | student | executive`
  - `location`: `coffee_shop | dev_cave | street | car | home_office | podcast_studio | glass_office | rooftop | bedroom | park | gym`
- `POST /api/agent/shorts` requires `prompt`, `avatar_url`, and `duration`,
  supports `resolution: 480p | 720p`, `aspect_ratio: 9:16 | 16:9 | 1:1`, and
  optional `speech_text` from 5 to 1000 characters.
- `avatar_url` for generation must be hosted on `download.citedy.com` or
  Supabase storage.
- `GET /api/agent/shorts/{id}` may already return the final video with
  `subtitles_applied: true` and may also include `subtitle_warning`.
- `POST /api/agent/shorts/merge` is optional, accepts 2-4 `video_urls`, and
  requires `phrases.length === video_urls.length`.
- `POST /api/agent/shorts/publish` accepts `video_url`, `speech_text`,
  `targets`, and optional `privacy_status`.
- Supported publish targets are `instagram_reels` and `youtube_shorts`, at most
  one target per platform in a single request.
- Publish `video_url` must be HTTPS on `download.citedy.com` or Supabase
  storage.
- Identical publish retries should reuse the exact same payload and are expected
  to replay the prior successful result instead of creating a duplicate post.

## When To Use

Activate this skill for requests like:

- "Create a Reel about my product"
- "Make a YouTube Short"
- "Generate a talking-head promo video"
- "Create a short with subtitles"
- "Turn this topic into a 15-second UGC clip"
- "Publish this short to Instagram"
- "Publish this short to YouTube Shorts"
- "Make a video shorts series about SEO"

## Setup

If you do not already have a saved `citedy_agent_...` API key in
`CITEDY_API_KEY`, use this flow once.

### 1. Register

If this copy ships with `scripts/register.mjs`, you can use the helper:

```bash
node scripts/register.mjs [agent_name]
```

Otherwise call the API directly:

```http
POST https://www.citedy.com/api/agent/register
Content-Type: application/json

{"agent_name": "<your_agent_name>"}
```

Expected response:

```json
{
  "pending_id": "pend_123",
  "approval_url": "https://www.citedy.com/approve-agent/...",
  "expires_in": 3600
}
```

### 2. Approve

Open the returned `approval_url`, approve the agent, then copy the generated
key shown on the site. The key itself starts with `citedy_agent_...`.

### 3. Save Key

Store that real generated key value in the `CITEDY_API_KEY` environment
variable.

Important:

- The site issues a key formatted like `citedy_agent_...`
- `CITEDY_API_KEY` is only the local env variable name used to store it

### 4. Authenticate

Send the key on every authenticated request:

```http
Authorization: Bearer <CITEDY_API_KEY>
```

### 5. Inspect Account Metadata

After setup, call `GET /api/agent/me` to inspect credits, connected platforms,
and referral data.

Example response:

```json
{
  "tenant_balance": {
    "credits": 1500,
    "status": "active"
  },
  "referral": {
    "code": "ABC123XZ",
    "url": "https://www.citedy.com/register?ref=ABC123XZ"
  },
  "connected_platforms": [
    {
      "platform": "instagram_reels",
      "connected": true,
      "id": "cdfaf220-0000-0000-0000-000000000000",
      "account_name": "@brand"
    },
    {
      "platform": "youtube_shorts",
      "connected": false
    }
  ]
}
```

## Default Operating Flow

Canonical execution order:

1. Check auth and connected accounts with `/api/agent/me`
2. Generate script
3. Generate or reuse avatar
4. Generate video
5. Poll status
6. Skip merge if the render already returned a subtitled final asset
7. Publish only if requested

### Step 0 - Discover Connected Accounts

Before publish decisions, inspect available connected platforms:

```http
GET https://www.citedy.com/api/agent/me
Authorization: Bearer <CITEDY_API_KEY>
```

Use the returned connected account `id` values for publish targets.

Guidance:

- Save connected `id` values before publish
- If no platforms are connected, the user can still generate the video and use
  the returned download URL manually
- If publish is requested, point users to
  `https://www.citedy.com/dashboard/settings` to connect Instagram or YouTube

### Step 1 - Generate Script

`POST /api/agent/shorts/script` costs 1 credit.

```http
POST https://www.citedy.com/api/agent/shorts/script
Content-Type: application/json
Authorization: Bearer <CITEDY_API_KEY>

{
  "topic": "Why founders should automate SEO content",
  "duration": "short",
  "style": "hook",
  "language": "en"
}
```

Example response:

```json
{
  "script": "Stop writing SEO content manually. Use Citedy AI to turn your research into ranking articles in seconds.",
  "word_count": 17,
  "estimated_seconds": 8
}
```

### Step 2 - Generate Avatar

`POST /api/agent/shorts/avatar` costs 3 credits.

```http
POST https://www.citedy.com/api/agent/shorts/avatar
Content-Type: application/json
Authorization: Bearer <CITEDY_API_KEY>

{
  "gender": "female",
  "origin": "latin",
  "age_range": "26-35",
  "type": "tech_founder",
  "location": "coffee_shop"
}
```

Example response:

```json
{
  "avatar_url": "https://download.citedy.com/agent/avatars/example.png"
}
```

### Step 3 - Generate Video

`POST /api/agent/shorts` is asynchronous and costs:

- 5 seconds: 60 credits
- 10 seconds: 130 credits
- 15 seconds: 185 credits

```http
POST https://www.citedy.com/api/agent/shorts
Content-Type: application/json
Authorization: Bearer <CITEDY_API_KEY>

{
  "prompt": "Female tech founder in a coffee shop workspace. Camera: medium close-up, vertical framing, steady shot. Style: polished startup aesthetic. Motion: subtle head nods and natural hand gestures. Audio: no background music.",
  "avatar_url": "https://download.citedy.com/agent/avatars/example.png",
  "duration": 10,
  "resolution": "720p",
  "aspect_ratio": "9:16",
  "speech_text": "Stop writing SEO content manually. Use Citedy AI to turn your research into ranking articles in seconds."
}
```

Immediate response:

```json
{
  "id": "job_123",
  "status": "processing"
}
```

### Step 4 - Poll Status

```http
GET https://www.citedy.com/api/agent/shorts/{id}
Authorization: Bearer <CITEDY_API_KEY>
```

Typical completed response:

```json
{
  "id": "job_123",
  "status": "completed",
  "video_url": "https://download.citedy.com/agent/shorts/final.mp4",
  "subtitles_applied": true,
  "subtitle_warning": null
}
```

Important:

- If `status` is `completed` and `subtitles_applied` is `true`, that
  `video_url` is already publish-ready
- If you have one finished clip, do not force a merge step
- If `subtitle_warning` is present, surface it clearly to the user

### Step 5 - Optional Merge

Use merge only for 2-4 clip assembly or custom subtitle restyling.

`POST /api/agent/shorts/merge` costs 5 credits.

```http
POST https://www.citedy.com/api/agent/shorts/merge
Content-Type: application/json
Authorization: Bearer <CITEDY_API_KEY>

{
  "video_urls": [
    "https://download.citedy.com/agent/shorts/part-1.mp4",
    "https://download.citedy.com/agent/shorts/part-2.mp4"
  ],
  "phrases": [
    { "text": "First subtitle phrase." },
    { "text": "Second subtitle phrase." }
  ],
  "config": {
    "words_per_phrase": 4,
    "font_size": 48,
    "text_color": "#FFFFFF",
    "stroke_color": "#000000",
    "stroke_width": 2
  }
}
```

Example response:

```json
{
  "final_video_url": "https://download.citedy.com/agent/shorts/final-merged.mp4"
}
```

### Step 6 - Optional Publish

Use the Step 3 `video_url` directly when the video is already finished with
subtitles. Only use the merged URL when you actually ran Step 5.

```http
POST https://www.citedy.com/api/agent/shorts/publish
Content-Type: application/json
Authorization: Bearer <CITEDY_API_KEY>

{
  "video_url": "https://download.citedy.com/agent/shorts/final.mp4",
  "speech_text": "Stop writing SEO content manually. Use Citedy AI to turn your research into ranking articles in seconds.",
  "targets": [
    { "platform": "instagram_reels", "account_id": "cdfaf220-0000-0000-0000-000000000000" }
  ],
  "privacy_status": "public"
}
```

Example response:

```json
{
  "results": [
    {
      "platform": "instagram_reels",
      "success": true,
      "post_id": "18099473800954601"
    }
  ],
  "credits_charged": 5,
  "metadata": {
    "title": "Auto-generated title",
    "description": "Auto-generated description with hashtags"
  },
  "metadata_provider": "gemini",
  "metadata_degraded": false,
  "total_time_ms": 8500
}
```

Publish notes:

- `instagram_reels` costs 5 credits
- `youtube_shorts` costs 0 credits
- `speech_text` is required and is used to derive title, description, and
  hashtags
- On a client timeout or retry, resend the exact same payload

## Worked Examples

### Example 1 - Product Promo Reel

User request:

> Make a 10-second Instagram Reel promoting my skincare serum.

Recommended response:

> I'll create a 10-second product Reel for your skincare serum.
>
> Estimated cost:
> - Script: 1 credit ($0.01)
> - Avatar: 3 credits ($0.03)
> - Video generation (10s): 130 credits ($1.30)
> - Merge: 0 credits if the generated clip already comes back with subtitles
> - Instagram publish: 5 credits ($0.05) if you want auto-publishing
>
> Total:
> - Generate only: 134 credits ($1.34)
> - Generate + Instagram publish: 139 credits ($1.39)

Example execution summary:

> Script ready.
> Avatar ready.
> Generating video now. This usually takes about 60-120 seconds.
> Polling for completion...
> Video ready: `https://download.citedy.com/agent/shorts/final.mp4`
> Your Instagram account is connected. Want me to publish it now?

### Example 2 - Educational YouTube Short

User request:

> Create a YouTube Short explaining what SEO is for beginners.

Recommended response:

> I'll create a 15-second educational YouTube Short.
>
> Estimated cost:
> - Script: 1 credit
> - Avatar: 3 credits
> - Video generation (15s): 185 credits
> - YouTube publish: 0 credits
>
> Total: 189 credits ($1.89) before any optional merge.

Example delivery:

> Script: "SEO stands for Search Engine Optimization. It's how your website shows up when people search. Better SEO means more qualified traffic without paying for every click."
>
> Video ready and subtitled:
> `https://download.citedy.com/agent/shorts/final.mp4`

### Example 3 - Quick Hook Video

User request:

> I need a 5-second hook video for Instagram.

Recommended response:

> A 5-second hook video costs:
> - Script: 1 credit
> - Avatar: 3 credits
> - Video generation (5s): 60 credits
>
> Total: 64 credits ($0.64) before optional merge or publish.

## API Reference

All authenticated endpoints require:

```http
Authorization: Bearer <CITEDY_API_KEY>
```

### POST /api/agent/register

Public registration endpoint used before the agent has a key.

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `agent_name` | string | yes | Human-readable agent name |

Response includes `pending_id`, `approval_url`, and `expires_in`.

### GET /api/agent/me

Returns tenant balance, rate limits, referral data, and connected platforms.

Useful fields:

- `tenant_balance.credits`
- `referral.code`
- `referral.url`
- `connected_platforms[].id`
- `connected_platforms[].platform`

### POST /api/agent/shorts/script

Generate a speech script for the avatar.

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `topic` | string | yes | What the video is about |
| `duration` | `"short"` \| `"long"` | no | `short` is typically 5-10s, `long` is typically around 15s |
| `style` | `"hook"` \| `"educational"` \| `"cta"` | no | Tone of the script |
| `language` | string | no | ISO 639-1 language code |
| `product_id` | string | no | Product context when available |

Cost: 1 credit.

### POST /api/agent/shorts/avatar

Generate an AI avatar image.

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `gender` | `"male"` \| `"female"` | no | Avatar gender |
| `origin` | enum | no | Ethnic origin preset |
| `age_range` | enum | no | `18-25`, `26-35`, or `36-50` |
| `type` | enum | no | `tech_founder`, `vibe_coder`, `student`, or `executive` |
| `location` | enum | no | One of the supported scene presets |

Cost: 3 credits.

### POST /api/agent/shorts

Submit a shorts generation job.

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `prompt` | string | yes | Visual prompt using the 5-layer structure |
| `avatar_url` | string | yes | URL on `download.citedy.com` or Supabase storage |
| `duration` | `5` \| `10` \| `15` | yes | Video duration in seconds |
| `resolution` | `"480p"` \| `"720p"` | no | Defaults to `480p` |
| `aspect_ratio` | `"9:16"` \| `"16:9"` \| `"1:1"` | no | Defaults to `9:16` |
| `speech_text` | string | no | Exact spoken words, 5-1000 chars |

Costs:

- 5 seconds: 60 credits
- 10 seconds: 130 credits
- 15 seconds: 185 credits

### GET /api/agent/shorts/{id}

Poll generation status.

Returned fields may include:

- `status`
- `video_url`
- `subtitles_applied`
- `subtitle_warning`

### POST /api/agent/shorts/merge

Merge multiple generated clips and add styled subtitles.

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `video_urls` | string[] | yes | 2-4 URLs, all on `download.citedy.com` |
| `phrases` | `{ text: string }[]` | yes | One phrase object per video |
| `config.words_per_phrase` | number | no | Integer from 2 to 8 |
| `config.font_size` | number | no | Integer from 16 to 72 |
| `config.position_from_bottom` | number | no | Integer from 50 to 300 |
| `config.text_color` | string | no | Hex or named color |
| `config.stroke_color` | string | no | Hex or named color |
| `config.stroke_width` | number | no | Integer from 0 to 5 |

Cost: 5 credits.

### POST /api/agent/shorts/publish

Publish a finished short to connected accounts.

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `video_url` | string | yes | HTTPS URL on `download.citedy.com` or Supabase storage |
| `speech_text` | string | yes | 5-2000 chars, used for publish metadata |
| `targets` | array | yes | 1-2 targets, max one per platform |
| `targets[].platform` | `"instagram_reels"` \| `"youtube_shorts"` | yes | Target platform |
| `targets[].account_id` | uuid | yes | Connected account ID from `/api/agent/me` |
| `privacy_status` | `"public"` \| `"unlisted"` \| `"private"` | no | Defaults to `public` |

Costs:

- Instagram Reels publish: 5 credits
- YouTube Shorts publish: 0 credits

## Utility Endpoints

These endpoints are useful for setup and diagnostics:

| Endpoint | Method | Cost | Description |
| --- | --- | --- | --- |
| `/api/agent/health` | GET | 0 credits | Check API availability |
| `/api/agent/me` | GET | 0 credits | Credits, referral info, connected platforms |
| `/api/agent/products` | GET | 0 credits | List registered products |
| `/api/agent/products/search` | POST | 0 credits | Search products for script context |

## Pricing

| Step | Cost (credits) | Cost (USD) |
| --- | --- | --- |
| Script generation | 1 | $0.01 |
| Avatar generation | 3 | $0.03 |
| Video generation (5s) | 60 | $0.60 |
| Video generation (10s) | 130 | $1.30 |
| Video generation (15s) | 185 | $1.85 |
| Merge | 5 | $0.05 |
| Publish to Instagram Reels | 5 | $0.05 |
| Publish to YouTube Shorts | 0 | $0.00 |
| Full 10s + Instagram publish | 139 | $1.39 |
| Full 15s + Instagram publish | 194 | $1.94 |

`1 credit = $0.01 USD`

## Prompt Best Practices

Use a clean 5-layer structure for `prompt`.

1. Scene: who is on screen and where they are
2. Camera: framing and movement
3. Style: visual tone and lighting
4. Motion: gestures, posture, head movement
5. Audio: usually `no background music`

Example:

```text
Professional founder in a modern startup office with a city backdrop.
Camera: medium close-up, vertical framing, steady shot.
Style: polished brand aesthetic, warm natural lighting.
Motion: subtle head nods, relaxed posture, natural hand gestures.
Audio: no background music.
```

Speech text rules:

- Put spoken words in `speech_text`, not in `prompt`
- Keep `speech_text` exact and quote-safe
- Keep speech concise enough for the selected duration
- If you are using a generated script, pass that script verbatim

## Limits And Operational Notes

- Registration is rate limited to 10 requests per hour per IP
- General agent routes are rate limited to 60 requests per minute
- Shorts generation allows only 1 concurrent video job per agent
- Merge is for 2-4 clips only
- Publish accepts at most 2 targets and only one per platform
- Prefer replaying identical publish payloads instead of improvising retries

## Error Handling

| Status | Meaning | Suggested Action |
| --- | --- | --- |
| `400` | Invalid JSON or schema validation failed | Fix the payload before retrying |
| `401` | Missing or invalid API key | Check `CITEDY_API_KEY` |
| `402` | Insufficient credits | Ask the user to top up |
| `409` | Concurrent generation already running | Poll the existing job and retry later |
| `429` | Rate limit exceeded | Respect `retry_after` or `Retry-After` |
| `500` | Server-side failure | Retry once carefully; for publish use the exact same payload |

## Response Guidelines

- Reply in the user's language
- Show the estimated credits before any paid operation
- Behave like a guided producer, not a REST tutorial
- Prefer the shortest valid path to the finished video
- Use defaults aggressively unless the user asks for something else
- Poll automatically after submitting generation
- Surface progress clearly: script ready, avatar ready, generating, polling,
  final URL ready
- Do not force a merge step when the render already produced a final subtitled
  asset
- Return the final direct download URL
- Offer publish only after the video is ready
- Confirm before calling publish

## Want More?

This skill covers video shorts only. For the broader content suite including
blog articles, social adaptations, competitor SEO analysis, lead magnets, and
keyword workflows, use the full `citedy-seo-agent` skill.
