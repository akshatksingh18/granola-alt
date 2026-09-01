# Setup & Base Repo

## Base repo
- **https://github.com/Natively-AI-assistant/natively-cluely-ai-assistant**
- License: Natively Personal Use Source License v1.0 — source-available, personal/non-commercial
  use only. This project stays within that: no monetization, no resale, no SaaS wrapper.
- The base is now imported on `main` with its original history. Its official repository is named
  `upstream`; there is deliberately no `origin` until Akshat creates a separate private personal
  repository in a future GitHub phase.
- `premium/` and `natively-api/` are upstream gitlinks and remain uninitialized. In particular,
  `premium/` points to a separate private repository and is irrelevant to this work — its paid
  diarization feature is not part of this project. Do not initialize either submodule unless
  Akshat separately approves it.

## First local build (not run yet)

```
npm install
npm run build:native
```

- Prerequisites: Node.js v20+, Git, Rust (`rustup`) — Rust is required to build the native audio
  module.
- Dev: `npm start`. Production build: `npm run dist` (Vite build → TypeScript compile → native
  module build → electron-builder).
- Env vars: copy `.env.example` → `.env`. It already lists the base repo's existing AI/speech
  provider keys. Add the var from `.env.example.addition` (`DEEPGRAM_API_KEY`) — get one free at
  `https://console.deepgram.com`.
- `.env` is ignored by Git. Never commit provider keys, meeting audio, transcripts, speaker
  embeddings, local databases, or exported meeting data.

## Do not rebuild
The native audio module already provides two continuous PCM streams — `systemAudioStream`
(everyone else, mixed) and `micAudioStream` (you). This project consumes those streams; it does
not touch or reimplement audio capture itself.

## Known directory structure
The imported base currently uses these top-level areas; still inspect the relevant implementation
before changing it because upstream conventions can evolve:

- `electron/` — main process
- `renderer/` — React UI
- `src/` — shared/core logic
- `native-module/` — Rust audio capture (consume, don't modify)
- `models/Xenova/...`, `resources/models/Xenova/...` — existing local ONNX models via
  `@xenova/transformers`, relevant to sourcing a speaker-embedding model (see `spec.md` §2)
- `worker-script/node/`, `scripts/`, `build/`

**Before writing any code**: explore the actual cloned repo — how it currently wires audio
streams to the renderer, how it talks to SQLite, existing Electron main/renderer IPC patterns —
and follow those conventions rather than inventing new ones.

## Note on the base repo's existing features
`natively-cluely-ai-assistant` is itself a Cluely clone — it ships with screenshot capture, a
live-answer overlay, and keyboard shortcuts for that workflow. This project doesn't use, build on,
or extend any of that. Leave it alone and unused rather than spending effort stripping it out —
this project only consumes the base repo's native audio-capture module and adds its own separate
transcription/history UI alongside it.
