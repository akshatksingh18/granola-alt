# Setup & Base Repo

## Base repo
- **https://github.com/Natively-AI-assistant/natively-cluely-ai-assistant**
- License: Natively Personal Use Source License v1.0 — source-available, personal/non-commercial
  use only. This project stays within that: no monetization, no resale, no SaaS wrapper.
- The base is now imported on `main` with its original history. Its official repository is named
  `upstream` (push disabled). The personal workflow remote `origin` is the private
  <https://github.com/akshatksingh18/granola-alt> repository; `main` tracks `origin/main`. The earlier same-machine bare
  repository `C:\Users\aksha\git-remotes\granola-alt.git` is kept as the `local-backup` remote. Keep
  `origin` private: the Natively licence allows personal, non-commercial use only. Three model/demo
  files are 66–94 MB (under GitHub's 100 MB limit); move them to Git LFS only if pushes start failing.
- `premium/` and `natively-api/` are upstream gitlinks and remain uninitialized. In particular,
  `premium/` points to a separate private repository and is irrelevant to this work — its paid
  diarization feature is not part of this project. Do not initialize either submodule unless
  Akshat separately approves it.

## Local Windows setup and validation

The local toolchain is installed and available to Akshat's Windows user:

- Git 2.53.0.windows.1
- Node.js 24.14.0 with npm 11.9.0
- Rust 1.98.0 through `rustup` (`C:\Users\aksha\.cargo\bin`)
- Visual Studio Build Tools 2026

Install the exact locked dependencies with:

```powershell
npm ci
```

The repository is on `D:` and Windows Application Control blocks Cargo build helpers generated
there. Keep Cargo's generated target directory on the user profile's `C:` drive when building the
native module:

```powershell
$env:CARGO_TARGET_DIR = 'C:\Users\aksha\AppData\Local\Natively\cargo-target'
npm run build:native
```

The following validation has been completed on Windows:

- `npm run build:native` passed with the `CARGO_TARGET_DIR` override above.
- `npm run build` and `npm run typecheck:electron` passed.
- `npm run build:electron:core-smoke` passed. The corresponding core test suites passed:
  `test:core-smoke` (90 passed), `test:lib` (384 passed, 1 skipped), and `test:scripts`
  (86 passed).
- The full `npm run build:electron` cannot resolve imports from the uninitialized private
  `premium/` gitlink. That is an expected boundary of this public base, not a reason to fetch the
  private repository. A full premium-enabled package has therefore not been produced.
- `npm audit` reports 30 advisories in the locked tree (1 low, 4 moderate, 25 high, 0 critical).
  No automatic audit fix was applied because it would change the reviewed dependency graph.

Build validated on Windows for the native, renderer, type-check, and core Electron scopes.
Reviewed but not executed on macOS. Requires physical macOS verification for macOS-only behavior.

- Dev: `npm start`. Production build: `npm run dist` (Vite build → TypeScript compile → native
  module build → electron-builder); the production path still requires the private premium
  sources described above.
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
