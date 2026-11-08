# Mato Podcast Generator

A mini podcast episode generator that takes a topic, generates a two-host dialogue script using an LLM, converts it to audio via Gemini TTS, and lets users play back the final episode — all orchestrated through Inngest background jobs.
## Architecture
```
User enters topic
        │
        ▼
  POST /api/episodes
  ├─ Creates episode record in Supabase (status: pending)
  └─ Sends "episode/generate.requested" event to Inngest updated also st
        │
        ▼
  ┌─────────────────────────────────────────────┐
  │           Inngest Pipeline                   │
  │                                              │
  │  Step 1: generate-script                     │
  │  ├─ Calls Gemini 2.0 Flash for dialogue     │
  │  ├─ Validates JSON structure                 │
  │  └─ Stores script in Supabase               │
  │                                              │
  │  Step 2: generate-tts-segments               │
  │  ├─ Iterates each script line                │
  │  ├─ Calls Gemini TTS (Kore/Puck voices)     │
  │  ├─ Retries up to 3x per segment            │
  │  └─ Uploads segments to Supabase Storage     │
  │                                              │
  │  Step 3: stitch-audio                        │
  │  ├─ Downloads all segments                   │
  │  ├─ Concatenates PCM data with silence gaps  │
  │  ├─ Wraps in WAV container                   │
  │  └─ Uploads final file to Supabase Storage   │
  └─────────────────────────────────────────────┘
        │
        ▼
  Frontend (Supabase Realtime subscription)
  ├─ Live pipeline status updates
  ├─ Script display with edit capability
  └─ Custom audio player with host indicator
```

## Tech Stack

| Component | Choice | Reason |
|-----------|--------|--------|
| Framework | Next.js 15 (App Router) | Required by spec |
| Database | Supabase (Postgres + Storage) | Required by spec |
| Background Jobs | Inngest | Required by spec — step functions, retries, failure handling |
| LLM | Gemini 2.0 Flash | Free tier, JSON mode, fast generation |
| TTS | Gemini TTS (2.5 Flash Preview) | Free tier, multiple voices, matches spec preference |
| Styling | Tailwind CSS v4 | Utility-first, fast iteration |

## Setup Instructions

### Prerequisites

- Node.js 18+
- A [Supabase](https://supabase.com) project
- A [Google AI Studio](https://aistudio.google.com) API key (for Gemini)
- An [Inngest](https://inngest.com) account (free tier)

### 1. Clone and Install

```bash
git clone <repo-url>
cd mato-podcast-generator
npm install
```

### 2. Set Up Supabase

1. Create a new project at [supabase.com](https://supabase.com)
2. Run the schema in `supabase/schema.sql` via the SQL Editor
3. Create a storage bucket named `episode-audio` with **public access** enabled
4. Copy your project URL, anon key, and service role key

### 3. Set Up Gemini

1. Go to [Google AI Studio](https://aistudio.google.com)
2. Create an API key
3. The free tier supports both text generation and TTS

### 4. Environment Variables

Copy `.env.local.example` to `.env.local` and fill in the values:

```bash
cp .env.local.example .env.local
```

### 5. Run Development Server

```bash
# Terminal 1: Next.js dev server
npm run dev

# Terminal 2: Inngest dev server
npm run inngest-dev
```

Open [http://localhost:3000](http://localhost:3000) and the Inngest dashboard at [http://localhost:8288](http://localhost:8288).

## Error Handling Strategy

### Handled Well

1. **TTS Segment Failure (Partial Recovery)**: Each TTS segment has its own retry loop (3 attempts with exponential backoff). If one segment fails after all retries, the entire pipeline fails gracefully with a descriptive error stored in Supabase.

2. **Malformed LLM Output**: The script generation includes JSON parsing with fallback regex extraction, plus structural validation (correct host IDs, minimum exchange count, non-empty text). Gemini's `responseMimeType: "application/json"` forces structured output.

3. **Pipeline-Level Failure**: Inngest's `onFailure` callback catches any unhandled error and updates the episode status to "failed" with the error message, so the user always sees what went wrong.

4. **User Navigation During Generation**: The Inngest pipeline runs independently of the browser session. Users can navigate away and return — the episode will continue generating. Supabase Realtime delivers status updates when they come back.

### Would Handle in Production

- **Rate limiting**: Queue TTS requests to avoid hitting Gemini rate limits during concurrent episode generation. Could use Inngest's concurrency controls.
- **Audio quality validation**: Check segment duration and amplitude before concatenation; re-generate segments that are too short or silent.
- **Idempotent segment generation**: Track which segments succeeded so we can resume from the last failure instead of re-generating everything.
- **Storage cleanup**: Delete orphaned segment files after successful concatenation to save storage costs.

## Bonus Features

- **Supabase Realtime**: Live status updates via Postgres changes subscription (with polling fallback)
- **Script Editing**: Users can edit the generated script before triggering audio generation
- **Host Indicator**: Audio player shows which host is currently speaking during playback

## Trade-offs

1. **WAV vs MP3**: Using WAV for the final audio because it's trivial to concatenate PCM data without external dependencies (no ffmpeg needed on Vercel). MP3 would reduce file size ~10x but requires a native binary or WASM encoder.

2. **Sequential TTS**: Generating audio segments sequentially rather than in parallel to stay within Gemini's rate limits on the free tier. In production, parallel generation with concurrency controls would cut latency significantly.

3. **Time-based host detection**: The audio player estimates which host is speaking based on elapsed time / number of lines. In production, we'd store exact segment timestamps during concatenation for precise tracking.

4. **Single Inngest step for TTS**: All TTS segments are generated in one Inngest step rather than individual steps per segment. This reduces Inngest step overhead but means a failure in segment 8/10 restarts all TTS generation. Per-segment steps would enable surgical retries.

## Project Structure

```
src/
├── app/
│   ├── api/
│   │   ├── episodes/        # Episode CRUD endpoints
│   │   │   ├── route.ts     # GET (list) + POST (create)
│   │   │   └── [id]/route.ts # GET (single) + PATCH (edit script)
│   │   └── inngest/route.ts # Inngest webhook endpoint
│   ├── globals.css
│   ├── layout.tsx
│   └── page.tsx             # Main page (client component)
├── components/
│   ├── AudioPlayer.tsx      # Custom playback with host indicator
│   ├── EpisodeCard.tsx      # Single episode with expandable details
│   ├── EpisodeForm.tsx      # Topic input + suggestion chips
│   ├── EpisodeList.tsx      # Episode listing
│   ├── PipelineStatus.tsx   # Visual pipeline stage tracker
│   └── ScriptDisplay.tsx    # Script viewer/editor
├── hooks/
│   └── useEpisodes.ts       # Realtime subscription + polling
└── lib/
    ├── audio.ts             # WAV concatenation utilities
    ├── inngest/
    │   ├── client.ts        # Inngest client instance
    │   └── functions.ts     # Episode generation pipeline
    ├── llm.ts               # Gemini script generation
    ├── supabase/
    │   ├── client.ts        # Browser client
    │   └── server.ts        # Service role client
    ├── tts.ts               # Gemini TTS conversion
    └── types.ts             # Shared TypeScript types
```

## Deployment

Deploy to Vercel:

```bash
npm i -g vercel
vercel
```

Set the environment variables in the Vercel dashboard. The Inngest integration can be added via Vercel's marketplace or by configuring the signing key manually.
