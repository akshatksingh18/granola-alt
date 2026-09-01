# Functional spec — Live Diarized Meeting Transcript

See `CLAUDE.md` at repo root first for setup, constraints, and directory context. This file covers what to build.

## The problem this solves

Granola (the market leader in bot-free meeting notes) transcribes meetings without a bot joining the call, but it does not diarize — it cannot tell individual remote participants apart, and everyone who isn't the user gets lumped together. This module fixes that specific gap: real-time, per-person speaker separation, live during the meeting, with click-to-copy at the block level and full transcript history across meetings — the way Granola handles meeting history and notes, but with actual speaker identity.

## Non-negotiable functional requirements

1. Live transcript view, updating **during** the meeting — text appears as people speak, not after the meeting ends.
2. Transcript is broken into **blocks**, one block per continuous speaker turn. Each block is visually distinct (its own bubble/row) and labeled with a speaker name.
3. Clicking a block **selects** it (visually highlighted). Clicking again, or clicking elsewhere, deselects it.
4. A single, always-visible **Copy** button:
   - If a block is selected → copies exactly that block's text, **including partial text if the block is still actively being transcribed** (i.e., copy reads current live state, not a finalized-only value).
   - If no block is selected → copies the **entire transcript**, start to finish, all speakers included, formatted with speaker labels.
5. Speaker identity persists across the meeting in real time — not a post-meeting batch process.
6. First time hearing a given person, the system requires manual enrollment (a name typed in once). After that, it should recognize that person automatically in future meetings without re-enrollment.
7. Every meeting is saved with a name (user-editable, e.g. "EON team sync") and is browsable in a **history view** — a list of past meetings, searchable, each reopenable to view its full speaker-separated transcript.

## Architecture

Three real-time pipelines running concurrently per meeting, merging into one ordered transcript:

```
micAudioStream ──────────────► Deepgram streaming (no diarize) ──► "You" blocks
systemAudioStream ───────────► Deepgram streaming (diarize_model=latest) ──► "Speaker N" blocks ──► voice-ID matcher ──► named blocks
                                                                                      │
                                                                                      ▼
                                                                          local speaker embedding DB
                                                                          (enrolled voice profiles)
```

Both Deepgram connections write into a single ordered in-memory block array, keyed by timestamp, which the UI renders live and SQLite persists incrementally.

### 1. Streaming ASR + diarization (Deepgram)

Use the Deepgram Node SDK (`npm install @deepgram/sdk`), two concurrent streaming (WebSocket) connections per meeting:

**Mic channel** — no diarization needed:
- Params: `model=nova-3` (or `nova-2-meeting`), `smart_format=true`, `interim_results=true`, `utterances=true`, `punctuate=true`.
- Every final utterance → hardcode `speaker: "You"`.

**System-audio channel** — diarization on:
- Params: same as above, plus `diarize_model=latest` — verify current param names against `https://developers.deepgram.com/docs/diarization` at implementation time, this API has changed recently and will change again.
- Deepgram returns numeric `speaker: 0, 1, 2...` per word/utterance, **scoped to this single connection/session only** — these numbers are not stable across meetings and reset every session. Do not treat them as identity; treat them as "same voice within this call."

Handle `interim_results` (partial, unpunctuated, may change) separately from `is_final`/`speech_final` (locked text) — see block engine below for how these map to UI state.

### 2. Speaker identity layer (this is the part that makes it useful — build carefully)

Deepgram tells you "these two utterances were the same voice in this call." It does not tell you who that is. You need a separate, local layer for that:

- **Embedding model**: a speaker-verification embedding model run locally via ONNX (the repo already depends on `@xenova/transformers` / Xenova ONNX models for its RAG reranker — check whether a transformers.js-compatible speaker-embedding model exists before reaching for a separate runtime; SpeechBrain's ECAPA-TDNN is the standard open option if you need to bring your own ONNX export). **Confirm current model availability and licensing before hardcoding a specific model ID — do not guess a HuggingFace repo path, verify it resolves.**
- **Enrollment flow**: UI lets the user click any unnamed block ("Speaker 2") and type a name. On confirm: buffer ~5-10s of that speaker's audio from this session, compute an embedding, store it in a local `speakers` table keyed by name.
- **Runtime matching**: the *first time* a new Deepgram `speaker_id` appears in a session, buffer a few seconds of its audio, compute an embedding, compare via cosine similarity against every stored profile in the `speakers` table. Above threshold (start at ~0.75, expect to tune this against real recordings) → auto-label with that name for the rest of the session. Below threshold → label "Unknown speaker N" and let the user rename it, which then also enrolls it. **Only do this match once per new speaker per session, not per utterance** — apply the resolved name to every subsequent utterance carrying that Deepgram speaker_id for the rest of the call.
- Do not attempt cross-session identity without this embedding layer — Deepgram's numbers alone cannot give you it, no matter what parameters you pass.

### 3. Live block engine (data model + segmentation rules)

```ts
type Block = {
  id: string;
  meetingId: string;
  speakerId: string;       // "you" | enrolled speaker id | "unknown-<n>"
  speakerName: string;     // "You" | "Anujay" | "Unknown speaker 2"
  text: string;            // live-updating; current best transcript for this block
  startTime: number;
  endTime: number | null;  // null while block is still active
  isFinal: boolean;        // true once no more text will be appended
};
```

New block starts when:
- The active speaker changes (Deepgram speaker_id flips, or mic vs. system channel flips), **or**
- A silence gap exceeds ~2s from the same speaker (prevents one giant block for a long monologue), **or**
- The block hits a reasonable max length as a safety valve (e.g. ~500 words) even mid-monologue.

While a block is active: append interim results to `text` (mark visually as "in progress" — lighter weight / different color in the UI is fine), replace with the punctuated final text as `is_final`/`speech_final` events arrive, keep appending further finals to the same block until a new-block condition triggers.

**This block array, in its current live state, is what both the UI and the copy logic read from — there is no separate "finalize then display" step.** The UI must be able to read and display `text` on a block where `isFinal: false`.

### 4. UI / interaction spec

- Vertically scrolling list of blocks, most recent at bottom (or top — match whatever the base repo's existing chat/notes UI convention already is, stay consistent).
- Each block: speaker name as a small label above/beside the text, block text below.
- Click block body → toggle `selectedBlockId` in component state. Only one block selected at a time for v1.
- Fixed "Copy" button (e.g. top-right of transcript pane):
  - `selectedBlockId !== null` → `navigator.clipboard.writeText(blocks.find(b => b.id === selectedBlockId).text)`.
  - `selectedBlockId === null` → concatenate all blocks in order: `` `${speakerName} [${formatTime(startTime)}]:\n${text}\n\n` `` and copy that whole string.
- Meeting title field at top of the live view, editable, defaults to something reasonable (timestamp, or leave blank for the user to fill in — no calendar integration in v1, so no auto-fill source).

### 5. History & persistence

SQLite schema (extend the existing DB, use `better-sqlite3` or whatever the base repo already uses):

```sql
CREATE TABLE meetings (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  started_at INTEGER NOT NULL,
  ended_at INTEGER
);

CREATE TABLE blocks (
  id TEXT PRIMARY KEY,
  meeting_id TEXT NOT NULL REFERENCES meetings(id),
  speaker_id TEXT NOT NULL,
  speaker_name TEXT NOT NULL,
  text TEXT NOT NULL,
  start_time INTEGER NOT NULL,
  end_time INTEGER,
  order_index INTEGER NOT NULL
);

CREATE TABLE speakers (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  embedding BLOB NOT NULL,   -- serialized float vector
  created_at INTEGER NOT NULL,
  sample_count INTEGER DEFAULT 1
);
```

- Write blocks incrementally as they finalize (not just at meeting end) so a crash mid-meeting doesn't lose the transcript.
- History view: list of `meetings` sorted by `started_at desc`, title + date shown, simple text search over `title` and `blocks.text` (SQLite `LIKE` or FTS5 virtual table — FTS5 is barely more work and much better, use it) is sufficient for v1. Full semantic/vector search over history is a reasonable v2 addition given the base repo already has `sqlite-vec` wired up for its RAG feature — don't build it now, note it as a natural extension.
- Reopening a past meeting shows the same block list UI, read-only for editing but with identical click-to-select-and-copy behavior.

## Explicitly out of scope for this build (Phase 2, do not attempt in the same pass)

- **Live AI-generated answers or call/interview coaching, in any form, ever.** This is not a Phase 2 item to revisit later — it's a permanent non-goal for this project. If a future need for that arises, it's a different, separate project (a Cluely-style overlay), not an extension of this one. Nothing in this transcript engine should send live segments to an LLM for a suggested response.
- AI-generated meeting summaries / structured notes from a template (Granola's "Recipes"). This is a good follow-up feature — feed the finalized transcript to an LLM with a template prompt after the meeting ends — but it's a separate, independent feature from the live-diarization system above and should not be built in the same implementation pass. Note this is fundamentally different from the excluded item above: it's a post-meeting, one-shot summary of what already happened, not a live suggestion during the call.
- A typed-scratchpad-during-the-meeting feature (Granola's core mechanic of using your typed notes to steer the AI summary) — depends on the summary feature above existing first.
- Calendar integration / auto meeting titles.
- Ambient "meeting detected" auto-popup — meetings are started manually by the user, by design (not a gap, a deliberate choice).
- Cross-device sync, team sharing, Slack/Notion export.
- macOS support — Windows 11 only for this pass, matching the existing target.

## Known limitations to accept, not silently paper over

- With no bot in the meeting, the system-audio channel is a single mixed-down stream of every remote participant combined — diarization on it is inherently harder than a bot getting separated per-participant audio from the platform backend. Expect real accuracy degradation above ~3-4 simultaneous distinct voices and during heavy cross-talk. This is a physical limitation of the no-bot approach, not a bug to be "fixed" later.
- Streaming (real-time) diarization is less reliable than batch/offline diarization on the same audio. Test on real multi-person Teams/Zoom calls before trusting it, not just on clean two-person test audio.
- Voice-embedding matching will misfire on close calls (similar-sounding voices, short utterances, bad audio quality from someone's bad mic on the call). Build the "Unknown speaker N, click to name" fallback as a first-class path, not an edge case — it will get used regularly, especially in the first few weeks.
