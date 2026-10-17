# ElephantEars — plan (draft for approval, 2026-10-17)

A Plaud replacement that runs on Alex's work laptop: records meetings (Teams and in-room), transcribes them, and serves the raw transcripts to Claude the same way the Plaud connector does today, so the Work Workflow check-in keeps working with one name swapped.

## What Plaud does for Alex today (what must be replaced)

- Records a meeting at the press of a button (about 15 hours a week, Mon–Thu).
- Transcribes with timestamps and "Speaker 1 / Speaker 2" labels (no real names).
- Exposes recordings to Claude through an MCP connector: `list_files` (id, name, start, duration) and `get_transcript` (paged segments: start_time, end_time, content, speaker). Claude only uses the raw transcript, never Plaud's AI note.
- The Monday/Thursday check-in reads new recordings and saves meetings into DogHaus (`add_meeting`, Plaud id stored on the meeting so it is never processed twice).

Not needed: Plaud's AI summaries, the outline block, the mobile app. Nice to keep: the highlight button (`mark_memo`).

## Decisions proposed

1. **Laptop app records, laptop transcribes.** A small tray app (Python, installed with `uv` into the user profile, no admin rights needed) captures the microphone and the system audio (what the Teams call plays) as two channels and transcribes locally with Whisper (faster-whisper). Local transcription is what makes this cheaper than Plaud: cloud transcription of 15 h/week costs about as much as the Plaud plan.
2. **Two channels give speaker separation for free.** Mic = Alex, loopback = everyone else. Remote speakers are labelled "Speaker 2, 3…" only if diarization is turned on later; this already matches what Plaud gives.
3. **Transcripts go to a tiny Cloudflare Worker (free plan, same account as DogHaus)** with a D1 table and an MCP endpoint that mirrors the Plaud tools (`list_files`, `get_transcript`, `get_file`). Claude gets an "ElephantEars" connector; the Work Workflow page swaps "Plaud" for it, nothing else changes. Audio stays on the laptop (optional R2 upload later).
4. **Same auth pattern as the DogHouse connector** (OAuth approval page behind the Cloudflare email code); the laptop uploads with a long-lived token.
5. **Manual start/stop is parity with Plaud.** Auto-detecting Teams calls is a Phase 3 nicety.
6. **No new subscriptions.** Cloudflare free, local Whisper. Optional cloud fallback (Cloudflare Workers AI Whisper or Groq) only if the laptop CPU is too slow; cost to verify on real numbers.

## Open questions (Alex)

1. Work laptop: Windows or Mac? Can you install software / run `uv`/Python without IT approval? (If installs are blocked, the fallback is a browser recorder on a Cloudflare Pages site plus Workers AI transcription.)
2. Meetings: mostly Teams on the laptop, in-room, or both? In-room meetings rely on the laptop mic; is that acceptable, or does the Plaud device stay for those?
3. Is sending audio to a cloud transcriber acceptable as a fallback, or must everything stay local?
4. Keep the audio files, or delete after transcription? (Keep locally for 30 days is the proposal.)
5. Company policy on recording: Plaud is already in use, so the assumption is that it's fine as today.

## Phases

### Phase 0 — spike on the work laptop (1 session)
- Repo skeleton, this plan on `main`.
- A 40-line script: record 60 s of mic + system audio to WAV, transcribe with faster-whisper `small.en` and `medium.en`, print timing. This answers: can we capture Teams audio on this laptop, and how long does a 30-minute meeting take to transcribe on its CPU.
- Go/no-go on local transcription; pick the model.

### Phase 1 — recorder (the thing that replaces the device)
- Tray app: Start / Stop / Mark highlight; hotkeys. Shows "recording 12:34".
- Writes `recordings/YYYY-MM-DD-HHMM/` with audio (Opus or WAV), a `meta.json` (start, duration, title from the active Teams window if available, marks) and, after transcription, `transcript.json` in the Plaud segment shape.
- Transcription runs in the background after Stop, mic and loopback channels separately, merged by time into one speaker-labelled transcript.
- Crash-safe: audio is written in chunks so a laptop sleep or crash loses at most a few seconds.
- Tests for the merge and the transcript shape.

### Phase 2 — the feed to Claude (replaces the Plaud connector)
- Cloudflare Worker + D1: `recordings` and `segments` tables; `POST /upload` from the laptop; MCP endpoint with `list_files` (date filters, paging), `get_transcript` (paged, `mark_memo` block), `get_file` (metadata).
- OAuth sign-in reused from DogHouse; add the connector to Claude; test a live check-in.
- Update the "Work Workflow" page in DogHaus: Plaud → ElephantEars (ids keep the `plaud` field name on meetings or get a new one, decide then).
- Run both for two weeks, then cancel Plaud.

### Phase 3 — polish
- Auto-start when a Teams call begins (audio session detection), auto-title from the calendar/Teams window, auto-stop on silence.
- Optional speaker diarization on the remote channel (pyannote, local) or speaker naming by Alex after the fact.
- Retention: delete local audio after N days; export a recording on demand.
- Optional cloud transcription fallback; optional audio upload to R2.

## Stack

- Laptop: Python 3.12 via `uv`; `sounddevice` + WASAPI loopback (Windows) or ScreenCaptureKit/BlackHole (Mac); `faster-whisper`; `pystray` for the tray; `pytest`.
- Cloud: Cloudflare Worker (TypeScript), D1, same tooling and tests as DogHouse; `wrangler`.
- Shape of a transcript segment (matches Plaud): `{start_time, end_time, content, speaker, segment_uid}` with times in milliseconds.

## Risks

- Corporate laptop restrictions (no installs, locked audio devices, endpoint security flagging a loopback recorder). Phase 0 exists to find out early.
- CPU too slow for `medium.en`: fall back to `small.en` or distil-whisper, or a cloud fallback.
- Mac vs Windows audio capture is very different; the plan assumes Windows until Alex confirms.
