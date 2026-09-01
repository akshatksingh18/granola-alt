# Meeting Notes — Granola Alternative (Live Diarized Transcript)

@natively-instructions.md

A standalone Electron + TypeScript desktop app (Windows 11), built on top of the **Natively**
base (`natively-cluely-ai-assistant`). Purpose: live, real-time, per-speaker transcript blocks
during Zoom/Teams calls — no bot joins the meeting — with click-to-copy per block (even mid-
transcription) or full-transcript copy, plus persistent meeting history with speaker recognition
across sessions.

This closes a specific gap in Granola (the closest competitor): Granola transcribes without a bot
but does not diarize — it cannot tell individual remote participants apart. This project does.

**This is a transcription tool. Nothing more.** No live AI-generated answers, no interview/call
coaching, no Cluely-style suggestion overlay — that is explicitly not a goal of this project, now
or later. The scope is: capture audio, diarize speakers in real time, show it as clickable/
copyable live blocks, save it to searchable history. That's the entire product.

**Status:** Planning/specification — the Natively base and its Git history are present, but no
feature code has been written yet. Moves to Building once the feasibility spike below passes.

## Files
- `spec.md` — full functional spec: requirements, architecture, data models, known limitations.
  **Read in full before implementing any part of this feature** — this `CLAUDE.md` is a pointer,
  not a substitute.
- `tasks.md` — build order (4 phases) and acceptance criteria as checkboxes. Work in order —
  don't jump to Phase 4 (speaker identity, highest risk) before 1-3 are proven.
- `setup.md` — base repo link/license, clone/build/env-var steps, known directory structure, and
  what not to rebuild (the native audio-capture module).
- `.env.example.addition` — the one new env var this feature adds (`DEEPGRAM_API_KEY`) on top of
  the base repo's own `.env.example`.
- `natively-instructions.md` — the upstream Natively development contract, preserved from its
  original `CLAUDE.md` and imported above; follow it together with this project-specific file.
- `LICENSE` — Natively Personal Use Source License v1.0; this source-available base is restricted
  to personal, educational, research, and non-commercial use.
- `.gitmodules` — upstream submodule declarations. `premium/` and `natively-api/` intentionally
  remain uninitialized; do not fetch them unless Akshat separately approves it.
- `electron/`, `renderer/`, `src/`, `native-module/`, `natively-browser/`, and `worker-script/` —
  imported Natively application, UI, native audio, browser, and worker source trees.
- `package.json` and lockfiles — upstream Node/Electron dependency and script definitions.

## Hard constraints (do not drift from these)
- Windows 11 only for this build. No macOS.
- No calendar integration. No ambient "meeting detected" auto-popup — meetings are started
  manually by the user, by design.
- No AI-generated meeting summaries in this phase (that's an independent follow-up feature, see
  `spec.md` "Explicitly out of scope").
- Don't treat Deepgram's numeric speaker IDs as identity across sessions — they reset every call.
  Cross-session naming is handled entirely by the local voice-embedding layer described in
  `spec.md`.

## Working agreement
- Track progress in `tasks.md` as phases complete — don't skip ahead.
- **Whenever a new project-owned file or top-level source area is added to this folder**, add a
  bullet for it under `## Files` above, in the same edit, with a one-line description of what
  it's for. Upstream files within an already-indexed source area do not need individual bullets.
