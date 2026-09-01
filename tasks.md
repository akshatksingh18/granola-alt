# Build order & acceptance criteria

See `spec.md` for the full functional spec these phases implement. Work through phases in order — don't jump to Phase 4 before 1-3 are proven, it's the highest-risk piece and needs a working system underneath it to debug against.

## Phase 1 — Raw diarized transcription
Wire both Deepgram streaming connections to the existing `micAudioStream` / `systemAudioStream`. Confirm live transcribed text with numeric speaker labels on the system-audio channel, before touching UI or embeddings.

- [ ] Mic channel streams to Deepgram, returns text tagged "You"
- [ ] System-audio channel streams to Deepgram with diarization on, returns text tagged with numeric speaker IDs
- [ ] Both channels' output is visible somewhere (console log is fine at this stage) in real time, not batched

## Phase 2 — Block engine & live UI
Build the block engine and live UI against raw Deepgram output (numeric labels are fine at this stage).

- [ ] Blocks segment correctly on speaker change, silence gap, and max-length safety valve
- [ ] Interim results visibly update a block's text in real time; final results replace/lock it in
- [ ] Click a block → selects it (visual highlight); click again or elsewhere → deselects
- [ ] Copy button with a block selected → copies exactly that block's current text, including if it's still actively receiving new words
- [ ] Copy button with nothing selected → copies the full transcript in order, all speakers, formatted with labels

## Phase 3 — Persistence & history
Add the SQLite layer and history view, still with numeric labels.

- [ ] `meetings` and `blocks` tables created; blocks write incrementally as they finalize (not just at meeting end)
- [ ] Meeting title is editable in the live view
- [ ] History view lists past meetings sorted by date, searchable by title/text (FTS5)
- [ ] Reopening a past meeting shows its full transcript with the same select/copy behavior

## Phase 4 — Speaker identity (highest risk — build last)
Add the embedding-based naming layer once everything above is proven.

- [ ] Speaker-embedding model confirmed available and wired (verify the actual model/repo resolves before committing to it)
- [ ] `speakers` table stores name + embedding
- [ ] Enrollment: clicking an unnamed block and typing a name buffers audio, computes an embedding, stores it
- [ ] Runtime matching: a new Deepgram speaker ID in a session is matched against stored profiles once, applied to all their utterances for that call
- [ ] Below-threshold matches fall back to "Unknown speaker N" with the same click-to-name flow, not an error state

## Final acceptance criteria (full system, all phases integrated)

- [ ] Start a meeting, speak into the mic, have someone else speak into system audio (or play audio) — both show up as separate, correctly-labeled blocks live, word-by-word, not after the fact
- [ ] Click a block while it's still actively receiving new words, hit Copy, confirm clipboard has exactly that block's current text
- [ ] Deselect, hit Copy, confirm clipboard has the full transcript with every speaker included, in order
- [ ] Name an unrecognized speaker mid-meeting, confirm all their prior and future blocks in that session update to the new name
- [ ] End the meeting, find it in history by title search, reopen it, confirm the full transcript is intact with correct speaker labels
- [ ] Start a *new* meeting with the same person from before — confirm they're auto-recognized without re-enrollment (this is the real test of whether the embedding layer works, not just the happy path within one session)
