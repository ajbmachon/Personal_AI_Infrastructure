---
name: PAI Voice System
pack-id: danielmiessler-pai-voice-system-v1.1.0
version: 1.1.0
author: danielmiessler
description: Voice notification system with dual TTS backends (free edge-tts + premium ElevenLabs), prosody enhancement, and agent personality-driven voice delivery
type: feature
purpose-type: [notifications, accessibility, automation]
platform: macos
dependencies:
  - pai-hook-system (required) - Hooks trigger voice notifications
  - pai-core-install (required) - Skills, identity, and response format drive voice output
keywords: [voice, tts, elevenlabs, notifications, prosody, speech, agents, personalities, accessibility]
---

<p align="center">
  <img src="../icons/pai-voice-system-v2.png" alt="PAI Voice System" width="256">
</p>

# PAI Voice System (pai-voice-system)

> Voice notification system with natural speech synthesis and personality-driven delivery

> **Installation:** This pack is designed for AI-assisted installation. Give this directory to your AI and ask it to install using the wizard in `INSTALL.md`. The installation dynamically adapts to your system state. See [AI-First Installation Philosophy](../../README.md#ai-first-installation-philosophy) for details.

---

## Platform Requirements

| Platform | Status | Notes |
|----------|--------|-------|
| **macOS** | ✅ Fully Supported | Uses `afplay` (built-in) for audio playback |
| **Linux** | ⚠️ Experimental | Requires audio player modification |
| **Windows** | ❌ Not Supported | No current implementation |

---

## What This Pack Provides

- **Spoken Notifications**: Hear task completions via text-to-speech
- **Dual TTS Backends**: Free edge-tts (Microsoft neural voices) or premium ElevenLabs
- **Automatic Fallback**: ElevenLabs failures (401/429/quota) seamlessly fall back to edge-tts
- **Prosody Enhancement**: Natural speech patterns with 13 emotional markers
- **Agent Personalities**: Different voices for different agent types
- **Intelligent Cleaning**: Strips code blocks and artifacts for clean speech
- **Graceful Degradation**: Works silently when voice server is offline

## Voice Server

The voice server runs locally on port 8888 and:
- Receives notification requests via HTTP POST
- Generates speech using ElevenLabs API
- Plays audio using system audio player
- Supports emotional markers for prosody variation

### Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/notify` | POST | Full notification with voice/emotion support |
| `/pai` | POST | Simple notification with default voice |
| `/health` | GET | Health check and configuration status |

### Example Request

```bash
curl -X POST http://localhost:8888/notify \
  -H "Content-Type: application/json" \
  -d '{"message": "Task completed successfully", "voice_enabled": true}'
```

## Architecture Overview

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│   Stop Hook     │ ───► │  Voice Server    │ ───► │  ElevenLabs     │
│ (extracts msg)  │      │  (localhost:8888)│      │  TTS API        │
└─────────────────┘      └──────────────────┘      │    (premium)    │
        │                         │                 └────────┬────────┘
        │                         │                          │
        │                         │    ┌─────────────────┐   │ fallback
        │                         ├───►│  edge-tts       │◄──┘ on error
        │                         │    │  (free, no key) │
        │                         │    └─────────────────┘
        │                         ▼
        │                ┌─────────────────┐
        │                │  Audio Player   │
        │                │  (afplay)       │
        │                └─────────────────┘
        ▼
┌─────────────────┐
│ Response Format │
│ 🗣️ [AI_NAME]:  │
└─────────────────┘
```

### TTS Backend Selection

| Backend | Cost | Quality | API Key | Fallback |
|---------|------|---------|---------|----------|
| `edge-tts` (default) | Free | Good (Microsoft neural) | Not needed | N/A |
| `elevenlabs` | Paid | Premium | Required | Auto-fallback to edge-tts |

Set via `TTS_BACKEND` environment variable. When using ElevenLabs, the server automatically
falls back to edge-tts on 401 (unauthorized), 429 (rate limit), or quota exhaustion errors.

## Response Format Integration

The voice system reads from the response format defined in `pai-core-install`:

```
🗣️ PAI: [12 words max - spoken aloud by voice server]
```

The hook extracts this line, enhances it with prosody markers, and sends it to the voice server.

## The 5-Layer Prosody Enhancement Pipeline

```
┌──────────────────────────────────────────────────────────────────┐
│                    PROSODY ENHANCEMENT PIPELINE                   │
├──────────────────────────────────────────────────────────────────┤
│  1. TEXT EXTRACTION         Raw completion message                │
│  2. CONTEXT ANALYSIS        Detect emotional patterns             │
│  3. PERSONALITY PROSODY     Agent-specific speech patterns        │
│  4. SPEECH CLEANING         Remove non-spoken artifacts           │
│  5. VOICE DELIVERY          Personality → Voice ID routing        │
└──────────────────────────────────────────────────────────────────┘
```

## What's Included

| Component | File | Purpose |
|-----------|------|---------|
| Voice server | `src/voice/server.ts` | HTTP server for TTS requests |
| Server management | `src/voice/manage.sh` | Start/stop/restart server |
| Voice stop hook | `src/hooks/stop-hook-voice.ts` | Main agent voice notification |
| Subagent voice hook | `src/hooks/subagent-stop-hook-voice.ts` | Subagent voice notification |
| Prosody enhancer | `src/hooks/lib/prosody-enhancer.ts` | Add emotion/pauses to speech |
| Voice personalities | `voice-personalities.json` | Agent voice configurations |

**Summary:**
- **Files created:** 6
- **Hooks registered:** 2 (Stop, SubagentStop)
- **Dependencies:** pai-hook-system (required), pai-core-install (required)
- **TTS backends:** edge-tts (free, default) or ElevenLabs (premium, optional)

## Emotional Detection

The prosody enhancer detects emotional context from message patterns:

| Priority | Emotion | Triggers | Marker |
|----------|---------|----------|--------|
| 1 | urgent | "critical", "broken", "failing" | [🚨 urgent] |
| 2 | debugging | "bug", "error", "tracking" | [🐛 debugging] |
| 3 | insight | "wait", "aha", "I see" | [💡 insight] |
| 4 | celebration | "finally", "phew", "we did it" | [🎉 celebration] |
| 5 | excited | "breakthrough", "discovered" | [💥 excited] |
| 6 | investigating | "analyzing", "examining" | [🔍 investigating] |
| 7 | progress | "phase complete", "moving to" | [📈 progress] |
| 8 | success | "completed", "fixed", "deployed" | [✨ success] |
| 9 | caution | "warning", "careful", "partial" | [⚠️ caution] |

These markers are embedded in the message and the voice server adjusts stability/similarity_boost parameters accordingly.

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `TTS_BACKEND` | No | `edge-tts` | TTS backend: `edge-tts` (free) or `elevenlabs` (premium) |
| `EDGE_TTS_VOICE` | No | `en-GB-RyanNeural` | Default edge-tts voice |
| `ELEVENLABS_API_KEY` | For elevenlabs | - | Your ElevenLabs API key |
| `ELEVENLABS_VOICE_ID` | For elevenlabs | - | Default ElevenLabs voice ID |
| `VOICE_SERVER_PORT` | No | 8888 | Voice server port |
| `VOICE_SERVER_URL` | No | http://localhost:8888 | Voice server URL (for hooks) |
| `PAI_DIR` | No | ~/.config/pai | PAI installation directory |

### Quick Start (Free, No API Key)

```bash
# Install edge-tts
pipx install edge-tts

# Start the server (uses edge-tts by default)
bun run src/voice/server.ts

# Test it
curl -X POST http://localhost:8888/notify \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello from PAI voice system"}'
```

### Available edge-tts Voices

List all available voices:
```bash
edge-tts --list-voices
```

Pre-configured voices per agent personality (in `voice-personalities.json`):

| Agent | Voice | Gender | Accent |
|-------|-------|--------|--------|
| PAI | en-GB-RyanNeural | Male | British |
| Intern | en-US-BrianNeural | Male | American |
| Engineer | en-US-GuyNeural | Male | American |
| Architect | en-GB-ThomasNeural | Male | British |
| Researcher | en-US-AvaNeural | Female | American |
| Designer | en-US-AriaNeural | Female | American |
| Artist | en-US-EmmaNeural | Female | American |
| Pentester | en-GB-ThomasNeural | Male | British |
| Writer | en-GB-LibbyNeural | Female | British |

## Agent Voice Mapping

Configure multiple voices in `voice-personalities.json` for multi-agent conversations:

```json
{
  "voices": {
    "PAI": {
      "voice_id": "YOUR_VOICE_ID",
      "stability": 0.5,
      "similarity_boost": 0.5,
      "description": "Default PAI voice"
    },
    "Engineer": {
      "voice_id": "ENGINEER_VOICE_ID",
      "stability": 0.72,
      "similarity_boost": 0.65,
      "description": "Technical, precise"
    }
  }
}
```

## Credits

- **Author:** Daniel Miessler
- **Origin:** Extracted from production Kai system (2024-2026)
- **License:** MIT

## Works Well With

- **pai-hook-system** (required) - Hooks trigger voice notifications
- **pai-core-install** (required) - Response format provides 🗣️ line
- **pai-history-system** - Complementary functionality

## Changelog

### 1.1.0 - 2026-01-24
- **Added edge-tts backend**: Free Microsoft neural TTS voices, no API key required
- New `TTS_BACKEND` environment variable (`edge-tts` or `elevenlabs`)
- Automatic fallback from ElevenLabs to edge-tts on API errors (401/429/quota)
- Added `edge_tts_voice` field to all voice personality entries
- edge-tts is now the default backend (works out of the box, zero configuration)
- Updated `/health` endpoint to report active backend
- Prerequisite: `pipx install edge-tts`

### 1.0.1 - 2026-01-09
- **Documentation fixes**: INSTALL.md and VERIFY.md now correctly reference actual files
- Fixed: References to non-existent `start.sh`, `stop.sh`, `restart.sh`, `status.sh` → use `manage.sh`
- Fixed: `voices.json` → `voice-personalities.json`
- Fixed: `voice-server/` → `VoiceServer/` directory paths
- Clarified: Full voice server IS included at `src/voice/server.ts` (553 lines)

### 1.0.0 - 2026-01-08
- Initial release with complete voice server implementation
- ElevenLabs TTS voice server (`src/voice/server.ts`)
- Server management script (`src/voice/manage.sh`)
- Main agent stop hook (`src/hooks/stop-hook-voice.ts`)
- Subagent stop hook (`src/hooks/subagent-stop-hook-voice.ts`)
- Prosody enhancer with 13 emotional markers
- Voice personalities configuration
- Integrates with pai-core-install response format (`🗣️ [AI_NAME]:`)
