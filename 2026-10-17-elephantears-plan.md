# ElephantEars — plan (draft for approval, 2026-10-17)

A Plaud replacement that runs on Alex's work laptop: records meetings (Teams and in-room), transcribes them, and writes **one markdown file per meeting** with the transcript. The markdown file is the product; everything else exists to produce it or to hand it to Claude for the Work Workflow check-in.

## What Plaud does for Alex today (what must be replaced)

- Records a meeting at the press of a button (about 15 hours a week, Mon–Thu).
- Transcribes with timestamps and "Speaker 1 / Speaker 2" labels (no real names).
- Exposes recordings to Claude through an MCP connector: `list_files` (id, name, start, duration) and `get_transcript` (paged segments: start_time, end_time, content, speaker). Claude only uses the raw transcript, never Plaud's AI note.
- The Monday/Thursday check-in reads new recordings and saves meetings into DogHaus (`add_meeting`, Plaud id stored on the meeting so it is never processed twice).

Not needed: Plaud's AI summaries, the outline block, the mobile app. Nice to keep: the highlight button (`mark_memo`).

## Decisions proposed

1. **Laptop app records, laptop transcribes, nothing leaves the laptop.** A small Windows tray app (Python, installed with `uv` into the user profile, no admin rights needed) captures the microphone and the system audio (what the Teams or Zoom call plays) as two channels and transcribes locally with Whisper (faster-whisper). Local only, no cloud fallback (decision 2026-10-17). Local transcription is also what makes this cheaper than Plaud: cloud transcription of 15 h/week costs about as much as the Plaud plan.
2. **Two channels give speaker separation for free.** Mic = Alex, loopback = everyone else. Remote speakers are labelled "Speaker 2, 3…" only if diarization is turned on later; this already matches what Plaud gives.
3. **The output is a markdown file per meeting** (format below), written to `Documents\ElephantEars` by default (configurable). No JSON, no database, no connector in the core build.
3a. **Audio is deleted as soon as the markdown file is written** (decision 2026-10-17). It exists only as a temporary working file during the recording and the transcription; if transcription fails, the audio is kept until a retry succeeds, then deleted.
4. **The app posts every markdown file to DogHaus automatically** (decision 2026-10-18). Each transcript becomes a Work page in a "Recordings" project, tagged `new`; the local file stays as the copy. The check-in reads the `new` pages instead of Plaud. This needs one small addition on the DogHaus side (an ingest endpoint with a device token; see Phase 2) in the DogHouse repo.
5. **Manual start/stop is parity with Plaud.** Auto-detecting Teams calls is a Phase 3 nicety.
6. **No new subscriptions.** Local Whisper only. If the laptop CPU is too slow for `medium.en`, step down to `small.en` or `distil-whisper`; no cloud option.
7. **Windows only; Teams and Zoom calls only** (decision 2026-10-17). No in-room recording to design for; both apps play remote audio through the default output device, which WASAPI loopback captures.

## The markdown file

One file per recording, named `YYYY-MM-DD-HHMM-short-title.md` (title from the Teams window when available, otherwise from the first minute of speech, editable afterwards). Timestamps are `hh:mm:ss` from the start of the recording; speakers are "Alex" (mic channel) and "Speaker 2, 3…" (system audio), renamable by editing the file.

```markdown
# Retail media consultation

- Date: 2026-10-16 12:30
- Duration: 29 min
- Source: Teams
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

## Answered (2026-10-17)

1. Work laptop is **Windows**. Whether `uv`/Python can be installed without IT approval is what Phase 0 checks first.
2. Meetings are **Teams and Zoom only**, on the laptop.
3. **Local only.** No cloud transcription, not even as a fallback.
4. **No audio kept.** Deleted once the markdown file is written.
5. Company policy on recording: Plaud is already in use, so the assumption is that it's fine as today.

6. Transcripts are **added to DogHaus automatically** (2026-10-18). Only the transcript text leaves the laptop, and only to DogHaus; audio never does.

## Still open

- Output folder: `Documents\ElephantEars` unless Alex prefers somewhere synced.
- Whether the corporate network lets the laptop reach `home.dirtdogcollective.com` (Phase 0 checks).

## Phases

### Phase 0 — spike on the work laptop (1 session)
- Repo skeleton, this plan on `master`.
- Check `uv` installs into the user profile and Python runs (no admin).
- Check the laptop can reach `home.dirtdogcollective.com` over HTTPS from the work network (corporate proxies and TLS inspection are the usual blockers).
- A 40-line script: record 60 s of mic + system audio (WASAPI loopback) during a Teams or Zoom test call, transcribe with faster-whisper `small.en` and `medium.en`, print timing. This answers: can we capture call audio on this laptop, and how long does a 30-minute meeting take to transcribe on its CPU.
- Go/no-go on local transcription; pick the model.

### Phase 1 — recorder (the thing that replaces the device)
- Tray app: Start / Stop / Mark highlight; hotkeys. Shows "recording 12:34".
- Records mic and system audio as two channels into a temporary WAV under `%LOCALAPPDATA%\ElephantEars\work\` (chunked, so a sleep or crash loses at most a few seconds).
- After Stop, transcribes both channels in the background, merges them by time, writes `Documents\ElephantEars\YYYY-MM-DD-HHMM-short-title.md` in the format above, then deletes the WAV. A notification says when the file is ready.
- Title from the Teams or Zoom window title when available.
- Tests for the channel merge, paragraph merging, highlight placement and the markdown output.

### Phase 2 — the file lands in DogHaus (replaces the Plaud connector)
DogHaus side (DogHouse repo, added to the session for this phase):
- `POST /api/transcripts` on the Worker: body is the markdown plus `title`, `date`, `duration_sec`, `source` (teams/zoom), `recorded_at`; creates a Work page under the "Recordings" project with tag `new` and those values in `props`; idempotent on a client-generated recording id so a retry never makes a duplicate.
- Auth: a per-device API token (created in DogHaus settings, stored once on the laptop in Windows Credential Manager), checked by the Worker; the path is added to the Cloudflare Access bypass like `/mcp`.
- The meeting's existing `plaud` field holds the transcript page id (renamed to `recording` later if it bothers us); `list_items` type meeting plaud <id> keeps working for "already processed".

ElephantEars side:
- After the markdown is written, post it; on failure (offline, DogHaus down) leave a `.pending` marker next to the file and retry on the next app start and every 10 minutes.
- Tray shows "sent" / "pending (3)" so nothing is lost silently.

Workflow side:
- Update the "Work Workflow" page in DogHaus: step 1 of the check-in reads pages tagged `new` in Recordings (`list_items` type page tag new), saves meetings with `add_meeting`, puts the page id in `plaud`, and swaps the tag from `new` to `processed`.
- Run both ElephantEars and Plaud for two weeks, then cancel Plaud.

### Phase 3 — polish
- Optional speaker diarization on the remote channel (pyannote, local) or speaker naming by Alex after the fact.
- Auto-start when a Teams or Zoom call begins (audio session detection), auto-stop on silence.

## Stack

- Laptop: Windows, Python 3.12 via `uv`; `pyaudiowpatch` (PyAudio with WASAPI loopback) for system audio plus `sounddevice` for the mic; `faster-whisper`; `pystray` for the tray; `pytest`.
- Output: markdown, format above; posted to DogHaus with `httpx`, token kept in Windows Credential Manager via `keyring`.
- DogHaus side: one endpoint and one token type in the existing DogHouse Worker (TypeScript, D1), with its usual tests.

## Risks

- Corporate laptop restrictions (no installs, locked audio devices, endpoint security flagging a loopback recorder). Phase 0 exists to find out early.
- CPU too slow for `medium.en`: fall back to `small.en` or distil-whisper (no cloud option).
- Deleting audio means a bad transcription can't be redone; mitigation is keeping the WAV until the markdown is written and a short "redo with a bigger model" window is not offered. Accepted.
- Zoom and Teams can each be set to use a non-default output device; the app records the default device, so both apps must use it (documented in the setup guide).
