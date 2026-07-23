# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Branching / development flow

This is the `ipevolabs` fork. Branches have distinct roles:

- **`main`** — mirrors upstream. Only ever fast-forwarded with upstream changes; never commit our own work here.
- **`dev`** — the default branch and our working branch. All our own work lands here (directly or via feature branches off `dev`).

To pull in upstream changes: update `main`, then merge `main` → `dev`. Open new feature/fix branches off `dev` and merge them back into `dev`.

## Commands

```bash
npm run dev      # Next.js dev server on http://localhost:3000
npm run build    # Production build (output: "standalone", for Docker/Cloud Run)
npm start        # Serve the production build
npm run lint     # ESLint (flat config, eslint-config-next core-web-vitals + typescript)
```

There is no test suite. Verify changes with `npm run lint` and by exercising the app.

Running locally requires a LiveKit server (see README for the `livekit/livekit-server --dev` Docker command) and a paid-tier Gemini API key. Env vars live in `.env.local`: `GEMINI_API_KEY`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `LIVEKIT_URL`, and optional `BROADCAST_PASSWORD`.

## Architecture

Real-time broadcast translation: one organizer speaks; attendees each pick a language and hear an AI translation. The core insight is that **exactly one Gemini Live session runs per language per room, shared across all listeners** of that language.

### The translation flow

```
Organizer mic → LiveKit room ← TranslationBridge (one per language, joins as bot "translator-{lang}")
                                     ↓ subscribes to organizer audio, pipes PCM → Gemini Live WS
                                     ↓ Gemini translates (translationConfig.targetLanguageCode)
                                     ↓ publishes translated audio back as track "translated-audio-{lang}"
Attendee ← subscribes to the translator track for their chosen language
```

### Server-side (`src/lib/`)

- **`translation-session-manager.ts`** — A **singleton** (`getInstance()`, cached on `global`) holding all in-memory state: `sessions` and `translations` (`Map<sessionId, Map<lang, TranslationBridge>>`). `getOrCreate` reuses an active bridge and bumps `subscriberCount`, or spins up a new one; `unsubscribe` decrements and tears the bridge down when the count hits zero. **Because state is in-memory, the deployment is locked to a single instance (`--max-instances 1`)** — horizontal scaling would create duplicate bots. See the README's scaling section.

- **`translation-bridge.ts`** — One instance per active language. Joins the LiveKit room as a bot, subscribes to the organizer's audio (`autoSubscribe: false`, subscribes manually), streams PCM to the Gemini Live WebSocket, and republishes translated audio (24 kHz mono). Also emits interim/final transcriptions over LiveKit data channels (topic `"transcription"`), targeted only at attendees whose `language` participant attribute matches. Key resilience behavior: handles Gemini `goAway`/close by reconnecting via `sessionResumption` handles (`reconnectGemini`) without dropping the LiveKit room. Stops itself when the organizer disconnects. Input 48 kHz → Gemini; output 24 kHz.

- **`languages.ts`** — `SUPPORTED_LANGUAGES` list + lookup helpers. The Gemini model id and language codes (e.g. `zh-Hans`) live here and in the bridge.

### API routes (`src/app/api/`)

All routes go through the session-manager singleton. `"original"` is a sentinel language meaning "no translation / raw organizer audio" — handled specially in several routes.

- `sessions/` — POST creates a session (sanitizes `eventId` into a slug or generates a short uuid; accepts an optional `allowedLanguages` allowlist; enforces `BROADCAST_PASSWORD`). GET lists sessions. `sessions/[sessionId]/` GETs one / DELETEs (tears down all bridges).
- `token/` — Mints LiveKit JWTs. `role=organizer` grants publish; attendees only get a token if the session exists. Password-gated for organizers.
- `translate/` — POST = subscribe to a language (validates against `allowedLanguages`, unsubscribes `previousLanguage`); DELETE = unsubscribe. `translate/status/` lists active translations (the broadcast page polls this every ~3s, which also keeps the Cloud Run container warm). `translate/unsubscribe/` for beacon-style teardown on page unload.
- `auth/status/` — Reports whether `BROADCAST_PASSWORD` is configured.

### Client (`src/app/`)

- `page.tsx` — Landing / create-session form.
- `session/[id]/broadcast/page.tsx` — Organizer view: publishes mic + optional tab audio through a custom mixer, shows a QR code, listener count, and active translations. Holds a screen wake lock during broadcast.
- `session/[id]/watch/page.tsx` — Attendee view: picks a language (sets the `language` participant attribute so the bridge targets transcriptions correctly), renders translated audio + a live transcript split into paragraphs.

## Deployment notes

Designed for **Google Cloud Run** (bridges are long-lived WebSocket processes needing a persistent container). Critical config, all explained in the README: `--max-instances 1` (singleton state), `--no-cpu-throttling`, `--timeout 3600`, high `--concurrency`, and enough CPU/memory to avoid OOM (`@livekit/rtc-node` runs native WebRTC — budget ~20–30 MiB + ~10% vCPU per active language). `next.config.ts` marks `@livekit/rtc-node` and `ws` as `serverExternalPackages` and builds `output: "standalone"` for the Docker image.

Further docs live in `docs/` (`authentication.md` — IAP + password setup, `tutorial_cloud_run.md`, `diagnostics.md`, `dev_to_developer_guide.md`).
