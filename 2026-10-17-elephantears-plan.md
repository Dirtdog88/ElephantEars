# ElephantEars — plan (draft for approval, 2026-10-17)

A Plaud replacement that runs on Alex's work laptop: records meetings (Teams and in-room), transcribes them, and writes **one markdown file per meeting** with the transcript. The markdown file is the product; everything else exists to produce it or to hand it to Claude for the Work Workflow check-in.

## What Plaud does for Alex today (what must be replaced)

- Records a meeting at the press of a button (about 15 hours a week, Mon–Thu).
- Transcribes with timestamps and "Speaker 1 / Speaker 2" labels (no real names).
- Exposes recordings to Claude through an MCP connector: `list_files` (id, name, start, duration) and `get_transcript` (paged segments: start_time, end_time, content, speaker). Claude only uses the raw transcript, never Plaud's AI note.
- The Monday/Thursday check-in reads new recordings and saves meetings into DogHaus (`add_meeting`, Plaud id stored on the meeting so it is never processed twice).

Not needed: Plaud's AI summaries, the outline block, the mobile app. Nice to keep: the highlight button (`mark_memo`).

## Decisions proposed

1. **Laptop app records, laptop transcribes.** A small tray app (Python, installed with `uv` into the user profile, no admin rights needed) captures the microphone and the system audio (what the Teams call plays) as two channels and transcribes locally with Whisper (faster-whisper). Local transcription is what makes this cheaper than Plaud: cloud transcription of 15 h/week costs about as much as the Plaud plan.
2. **Two channels give speaker separation for free.** Mic = Alex, loopback = everyone else. Remote speakers are labelled "Speaker 2, 3…" only if diarization is turned on later; this already matches what Plaud gives.
3. **The output is a markdown file per meeting** (format below), written to a folder Alex chooses (decision 2026-10-17). No JSON, no database, no connector in the core build. Audio stays on the laptop.
4. **Getting the file to Claude is a separate, small step** (Phase 2). Options, cheapest first: Alex attaches or pastes the file into the check-in chat; the app posts it to DogHaus as a Work page or as a file on a meeting; or, later, a tiny Cloudflare Worker that serves the files to Claude as a connector mirroring the Plaud tools. Decide after Phase 1, once the files exist.
5. **Manual start/stop is parity with Plaud.** Auto-detecting Teams calls is a Phase 3 nicety.
6. **No new subscriptions.** Local Whisper. Optional cloud fallback (Cloudflare Workers AI Whisper or Groq) only if the laptop CPU is too slow; cost to verify on real numbers.

## The markdown file

One file per recording, named `YYYY-MM-DD-HHMM-short-title.md` (title from the Teams window when available, otherwise from the first minute of speech, editable afterwards). Timestamps are `hh:mm:ss` from the start of the recording; speakers are "Alex" (mic channel) and "Speaker 2, 3…" (system audio), renamable by editing the file.

```markdown
# Retail media consultation

- Date: 2026-10-16 12:30
- Duration: 29 min
- Source: Teams (ElephantEars)
- Highlights: 2

## Highlights

- [00:12:40] Alex: We'd want the API connection in place before the Walmart move.
- [00:21:05] Speaker 2: I'll send the proposal by Friday.

## Transcript

**[00:00:00] Alex:** Hey. How are you? My headphones did not connect.

**[00:00:09] Speaker 2:** I hate when that happens. It's funny, I have my AirPods, but…

**[00:00:28] Alex:** Yeah. We've had that. My wife works from home too…
```

Rules: consecutive segments by the same speaker within 3 seconds are merged into one paragraph; a highlight (the Mark hotkey) quotes the segment it landed on; the header block stays machine-readable so a later Phase 2 can index the files.

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
- Records mic and system audio as two channels into `<folder>/audio/YYYY-MM-DD-HHMM.wav` (chunked, so a sleep or crash loses at most a few seconds).
- After Stop, transcribes both channels in the background, merges them by time, and writes `<folder>/YYYY-MM-DD-HHMM-short-title.md` in the format above. A notification says when the file is ready.
- Tests for the channel merge, paragraph merging, highlight placement and the markdown output.

### Phase 2 — the file reaches Claude (replaces the Plaud connector)
- Pick one after Phase 1 has produced real files: (a) attach/paste into the check-in chat, nothing to build; (b) the app posts the markdown to DogHaus (a Work page, or a file on the meeting) through its API; (c) a Cloudflare Worker that serves the files to Claude as an "ElephantEars" connector mirroring the Plaud tools.
- Update the "Work Workflow" page in DogHaus: Plaud → ElephantEars files.
- Run both for two weeks, then cancel Plaud.

### Phase 3 — polish
- Auto-start when a Teams call begins (audio session detection), auto-title from the calendar/Teams window, auto-stop on silence.
- Optional speaker diarization on the remote channel (pyannote, local) or speaker naming by Alex after the fact.
- Retention: delete local audio after N days; the markdown files are kept.
- Optional cloud transcription fallback.

## Stack

- Laptop: Python 3.12 via `uv`; `sounddevice` + WASAPI loopback (Windows) or ScreenCaptureKit/BlackHole (Mac); `faster-whisper`; `pystray` for the tray; `pytest`.
- Output: markdown only, format above. No cloud component unless Phase 2 picks option (c), in which case a Cloudflare Worker (TypeScript) like DogHouse.

## Risks

- Corporate laptop restrictions (no installs, locked audio devices, endpoint security flagging a loopback recorder). Phase 0 exists to find out early.
- CPU too slow for `medium.en`: fall back to `small.en` or distil-whisper, or a cloud fallback.
- Mac vs Windows audio capture is very different; the plan assumes Windows until Alex confirms.
