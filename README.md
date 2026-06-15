# MedKit

Hands-free first-aid coaching through Meta Ray-Ban smart glasses. Say "Hey Medkit" — the AI sees through your glasses camera, asks clarifying questions, and guides you step-by-step until EMS arrives.

Built at DevFest CU 2025.

## What it does

MedKit follows established Red Cross and AHA first-aid playbooks. It never diagnoses — it coaches. Every response opens with "Call 911" for life-threatening situations, and a persistent banner reads: *Decision support only — call emergency services.*

Supported scenarios:
- **CPR** — 110 BPM metronome through the glasses speakers, compression rhythm guidance
- **Severe bleeding** — pressure hold timers, wound care steps
- **Choking** — Heimlich maneuver walkthrough
- **Burns, fractures, allergic reactions** — playbook-based guidance

If you go quiet during an active emergency, MedKit checks in — "Still pressing on the wound?" — because silence might mean you need encouragement, not that you're done.

## Architecture

```
iOS App (SwiftUI) ──WebSocket──▶ Modal Backend ──▶ OpenAI Realtime API
                                       │
                               Dedalus (GPT-4o Vision)
                                       │
                          Tool commands ──▶ iOS App (local execution)
```

The backend runs four concurrent async loops per session:

| Loop | Role |
|---|---|
| iOS → Realtime | Streams PCM16 audio and blurred video frames |
| Realtime → iOS | Routes AI voice, transcripts, and tool call commands |
| Scene analysis | GPT-4o Vision analyzes frames every 8 seconds, injects visual context |
| Proactive follow-ups | Checks in after 30s of silence, then every 45s (max 3 without response) |

## Stack

| Layer | Technology |
|---|---|
| Glasses | Meta Ray-Ban + Wearables DAT SDK (`MWDATCore`, `MWDATCamera`) |
| iOS app | SwiftUI, AVFoundation, SceneKit, MapKit |
| Voice | OpenAI Realtime API (PCM16, 24kHz, `alloy` voice) |
| Vision | GPT-4o Vision via Dedalus (DAuth credential isolation in hardware enclaves) |
| Backend | Modal (serverless ASGI), FastAPI WebSockets |
| Privacy | MediaPipe face blurring — on-device, before any frame leaves the phone |

## iOS Components

| Component | Role |
|---|---|
| `AudioManager` | Bidirectional audio: wake word detection via `SFSpeechRecognizer`, glasses mic capture, AI voice playback through glasses speakers |
| `ToolExecutor` | Runs metronome, countdown timers, and UI checklist cards locally on device |
| `SessionLogger` | Records 30 FPS H.264 video via `AVAssetWriter`, timestamped transcripts, PDF and EMS report export |
| `WebSocketManager` | Backend communication — audio/frame streaming, tool command ingestion |
| 3D wireframe | SceneKit body model with animated sphere overlays highlighting the active region (chest, arm, etc.) |
| MapKit | Nearby AED, hospital, and pharmacy search |

## Privacy

Face blurring runs on every frame on-device before any data is sent to the cloud. No raw video reaches the backend. Session recordings stay local — video, transcripts, and EMS reports are only exported when the user explicitly chooses to share them.

## Session Export

At the end of every session:

| Export | Format | Contents |
|---|---|---|
| Video | MP4, H.264, 30 FPS | Full session recording |
| Transcript | PDF | Timestamped conversation with session metadata |
| EMS report | TXT | Structured briefing for paramedics: timeline, scene observations, actions taken |

## Quick start

### Backend

```bash
pip install modal
modal deploy app.py
```

Set secrets in the Modal dashboard: `OPENAI_API_KEY`, `DEDALUS_API_KEY`.

### iOS App

1. Open `MedKit.xcodeproj` in Xcode
2. Set your team in Signing & Capabilities
3. In `Info.plist`: set `MetaAppID = ""` and `TeamID = <your-team-id>`
4. Build and run on a physical iPhone with the Ray-Ban glasses paired
