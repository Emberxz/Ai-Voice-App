AI Voice App — MVP Plan and Implementation Guide

This document contains a detailed, step-by-step plan to build a desktop-first, real‑time AI voice recognition app (cloud-first for fast iteration and Git learning). Save this file and refer to it as you work through the project.

Table of contents
- 1) MVP summary and success criteria
- 2) Decisions you should lock in (quick)
- 3) End‑to‑end architecture (diagram + explanation)
- 4) Project scaffold & Git onboarding (exact commands)
- 5) Audio capture & format (how to capture correctly)
- 6) VAD / silence suppression
- 7) Client → server streaming (protocol & chunking)
- 8) Server: relay to cloud STT (Google example)
- 9) UI (desktop) design and threading model
- 10) Latency tuning & measurement (how to measure E2E)
- 11) Testing & evaluation (WER, datasets)
- 12) Packaging and deployment (desktop + server)
- 13) Security, cost, and operational items
- 14) Minimal checklist to reach MVP (day-by-day)
- 15) What I can do next (pick one)

---

1) MVP summary and success criteria

- What the MVP does:
  - Desktop app captures microphone audio live.
  - Streams audio in small chunks to a server.
  - Server forwards audio to cloud STT (streaming) and returns partial/final transcripts.
  - Desktop displays live partial transcripts with small latency (<300 ms feel).
- Success criteria (measurable):
  - Partial transcripts displayed within 300 ms end-to-end on LAN.
  - Shows partial updates while user speaks.
  - Works reliably for short utterances (1–10 s).
  - Basic Git workflow in place: repo, branch, commits, push, PR.

---

2) Decisions you should lock in (quick)

- Desktop GUI tech: Python + PySide6 (recommended).
- Audio capture lib: sounddevice (Python).
- Transport: WebSocket for simplicity, WebRTC (aiortc) if you later need lower jitter/Opus.
- Cloud STT: Google Cloud Speech-to-Text Streaming (suggested), but Azure/AWS are similar.
- Audio parameters: 16 kHz, 16-bit PCM, mono.
- Buffering: capture 20–40 ms frames, send ~100–200 ms buffers.

---

3) End‑to‑end architecture (short)

- Client (desktop): Capture → VAD → encoder (optional) → chunk → WebSocket → display partials.
- Server (cloud): WebSocket endpoint per client → opens streaming session to cloud STT → forwards audio frames → receives partial transcripts → forwards partials back to client.

Diagram (conceptual)
Client (mic) → [VAD, chunking] → WebSocket → Server → Cloud STT → Server → WebSocket → Client (UI)

---

4) Project scaffold & Git onboarding (exact PowerShell commands)

Run these commands from your workspace to create the project scaffold and a Git repo.

```powershell
cd 'C:\Users\ajbas\OneDrive\Desktop\Python Projects'
mkdir ai-voice-app
cd ai-voice-app

# create venv and activate
python -m venv .venv
.\.venv\Scripts\Activate.ps1

pip install --upgrade pip setuptools wheel
```

Create folders and a `.gitignore`:

```powershell
mkdir backend frontend models
ni README.md
ni .gitignore
```

Add this to `.gitignore`:
```
.venv/
__pycache__/
models/
*.pyc
dist/
build/
```

Initialize git and commit:

```powershell
git init
git add -A
git commit -m "chore: scaffold project"
```

Create a GitHub repo (manual via web UI) or use `gh`:

```powershell
# Optional with GitHub CLI
gh repo create Emberxz/ai-voice-app --public --source=. --push
# Or, if you already created a remote manually:
# git remote add origin https://github.com/Emberxz/ai-voice-app.git
# git push -u origin master
```

Git workflow tips
- Use feature branches: `git checkout -b feature/streaming-prototype`
- Commit often. Use descriptive messages.
- Open PRs and merge via GitHub to learn the review workflow.

---

5) Audio capture & format (exact how)

- Target audio format: 16 kHz, 16-bit PCM, mono.
- Use `sounddevice` to record into numpy arrays.

Key parameters:
- sample_rate = 16000
- blocksize (frame capture) ≈ 320 (20 ms) to 640 (40 ms) samples
- Use callback-based capture for lowest latency.

Important functions/patterns:
- `sounddevice.query_devices()` to list mics.
- `sounddevice.InputStream(samplerate=16000, channels=1, dtype='int16', blocksize=320, callback=on_audio_block)`
- In callback, convert buffer to bytes with `.tobytes()` and push to a send queue.

Buffering strategy:
- Accumulate small blocks into a send buffer of ~100–200 ms before sending to reduce packet overhead but keep latency low.
- If VAD detects speech start, send the first buffer immediately.

---

6) Voice Activity Detection (VAD)

- Use `webrtcvad` (Python wrapper).
- Feed it 10/20/30 ms frames. Use aggressiveness 2 as a start.
- Use hysteresis: require N consecutive speech frames to start sending, M consecutive silence frames to stop.

---

7) Client → server streaming (protocol & chunking)

WebSocket approach (recommended for MVP):
- Open WebSocket to server: `wss://yourserver.example/ws`
- Message framing options:
  - Binary frames: raw PCM bytes (int16 little-endian). Server converts and forwards.
  - JSON with base64: `{"type":"audio","data":"BASE64"}` (easier debug, more overhead).

Suggested chunking:
- Capture 20–40 ms frames; accumulate to ~120 ms and send as one message.
- Mark the first frame of an utterance with metadata: `{"event":"start","sample_rate":16000}` so server can open STT session.

Server message types:
- `start_stream` — metadata
- `audio_chunk` — audio bytes
- `end_stream` — close utterance

Binary framing (efficient): prefix each frame with a 4-byte `uint32 length` then raw PCM bytes. Use WebSocket binary frames.

---

8) Server: relay to cloud STT (Google example)

Server responsibilities:
- Accept WebSocket connections.
- On `start_stream`, create a streaming session to Google Cloud Speech-to-Text (gRPC stream).
- Forward audio chunks to the cloud stream.
- Translate cloud partial results to JSON partials and send to clients.

Google Cloud onboarding (short):
- Create GCP project, enable Speech-to-Text API, enable billing.
- Create a service account and download JSON key.
- Install client libs:

```powershell
pip install google-cloud-speech websockets aiohttp fastapi uvicorn
```

Server pseudo-steps (SDK usage):
- Create `speech_v1p1beta1` or `speech_v1` streaming client.
- Stream audio chunks to `streaming_recognize` with config:
  - `sample_rate_hertz=16000`, `encoding=LINEAR16`, `language_code='en-US'`, `enable_automatic_punctuation=True`.
- Forward partials (`is_final==False`) and finals to clients as JSON messages.

Latency tips:
- Use gRPC streaming.
- Keep chunk sizes small (100–200 ms).
- Host server in same region as the STT endpoint.

---

9) Desktop UI design and threading model

- Use PySide6 (Qt). Keep UI main thread responsive. Run audio capture & WebSocket client in a background thread or `asyncio` task.
- UI elements: Start/Stop button, mic selection, live transcript area, connection indicator.

Thread model:
- Main thread: Qt event loop
- Worker thread: capture + VAD + queue audio → WebSocket sender
- Receiver: on partial/final from server, push to UI via Qt signals or thread-safe queue polled by UI timer.

UX tips:
- Show partial text in muted style; replace with final when `is_final` arrives.
- Consider push-to-talk initially.

---

10) Latency tuning & measurement (how to measure E2E)

Instrument timestamps at:
- `client_capture_time`, `client_send_time`, `server_receive_time`, `cloud_receive_time`, `cloud_response_time`, `server_send_partial_time`, `client_receive_partial_time`.

Compute total = `client_receive_partial_time - client_capture_time`.

Measurement methods:
- Use synthetic audio with an embedded tone or known start time.
- Log and compute percentiles (p50/p95/p99).

Tuning checklist if latency > 300 ms:
- Reduce buffer size to 80–100 ms.
- Check network RTT with `ping`.
- Test STT provider latency directly from server host.

---

11) Testing & evaluation

- Datasets: Mozilla Common Voice, LibriSpeech, domain samples.
- WER measuring: `jiwer`.

Commands:
```powershell
pip install jiwer
# python -c "from jiwer import wer; print(wer(ref, hyp))"
```

---

12) Packaging & deployment

Desktop packaging (Windows):
- `pip install pyinstaller`
- `pyinstaller --onefile --add-data "models;models" frontend/app.py`

Server deployment (Cloud Run example):
- Dockerfile example (backend):
```
FROM python:3.11-slim
WORKDIR /app
COPY backend/requirements.txt .
RUN pip install -r requirements.txt
COPY backend/ /app/
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

Build & deploy (gcloud):
```powershell
gcloud builds submit --tag gcr.io/PROJECT_ID/ai-voice-server
gcloud run deploy ai-voice-server --image gcr.io/PROJECT_ID/ai-voice-server --platform managed --region us-central1 --allow-unauthenticated
```

---

13) Security, cost, and operational items

- Security:
  - Use HTTPS/WSS and API keys or OAuth.
  - Don’t commit service account keys.
- Cost:
  - Cloud STT billed per audio minute. Track and set budget alerts.
  - Use VAD to reduce usage.
- Ops:
  - Add logs and metrics; monitor latency and errors.

---

14) Minimal checklist to reach MVP (day-by-day)

Day 0 — Git & scaffold
Day 1 — Local console prototype: mic -> WebSocket -> server (prints bytes)
Day 2 — Server -> Cloud STT relay; client displays partials
Day 3 — Simple PySide6 UI showing partials/finals
Day 4 — VAD and latency tuning
Day 5 — Packaging and PR

---

15) What I can do next (pick one)

- A) Guide you through Google Cloud STT onboarding and server skeleton.
- B) Provide the minimal client console prototype steps (sounddevice + websockets) with exact function names.
- C) Provide the PySide6 UI skeleton and threading pattern (detailed pseudo-code & signal wiring).
- D) Help set up GitHub Actions for CI.
- E) Provide a Dockerfile and `backend/requirements.txt` template.

Pick an option and I’ll produce the detailed next steps and commands you can run. Good luck — save this file and refer to it as you build your MVP!