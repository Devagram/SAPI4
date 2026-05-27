# STT → SAPI4 TTS pipeline (live-streaming character voice)

Goal: speak into mic → transcribe → SAPI4 TruVoice speaks the same text → route output into OBS so viewers hear the character voice instead of (or in addition to) my real voice.

**Architecture is batched by sentence/utterance, not streaming.** SAPI4 takes complete text and emits a finished WAV — there's no partial synthesis. So the pipeline naturally gates on end-of-utterance anyway, which means streaming STT (with partial results) buys nothing. Batch STT after VAD-detected silence is the right shape: simpler, cheaper, opens up local Whisper as a clean default.

Target end-to-end latency: **≤ 700 ms** from end-of-speech to character voice playing. The ~200–400 ms silence threshold for endpointing is the floor and is part of the brand — a tiny "loading" beat before the character speaks reads as character, not lag.

Use this file as a working checklist. Don't aim for a "polished product" — aim for "I can go live with this."

## Latency budget

| Stage | Target | Notes |
|---|---|---|
| Mic capture frame | 20–50 ms | Smaller chunks = less buffering but more CPU. 30 ms is standard. |
| VAD silence threshold (endpointing) | 200–400 ms | Hard floor on perceived latency. Lower = clipped phrases; higher = audible drag. Tune this carefully — it's the dominant knob. |
| Batch STT (whole utterance) | 200–500 ms | Whisper `base.en` on a modest GPU. CPU-only is 500 ms–1.5 s. |
| Text → SAPI4 spawn + synth | < 20 ms | `sapi4out.exe` at 16M× realtime is basically free. |
| Audio decode + route to OBS | 20–50 ms | Depends on virtual cable + buffer sizes. |
| **Total** | **450–900 ms** typical | Worst case ≤ 1.2 s acceptable for long sentences. |

Measure each stage separately before optimizing. Most users blow the budget on endpointing (waiting too long for silence) and on cold-start audio device latency on the first utterance.

## Decisions to make first

These shape everything downstream. Don't start building until they're settled.

### STT engine

Since the pipeline is batched on end-of-utterance, you want a *fast one-shot transcriber*, not a streaming-partials engine. That changes the ranking.

- [ ] **Pick one.** Options ranked by effort-to-quality for batched use:
  - [ ] **faster-whisper** with `base.en` (local, free): **default recommendation**. Runs on CPU or GPU, no API key, 200–500 ms per utterance on a modest GPU, ~500 ms–1.5 s on CPU. Accuracy is great. Model is ~150 MB. Use `tiny.en` (75 MB) if CPU latency hurts.
  - [ ] **Whisper.cpp** (local, free): same model family, C++ implementation, no Python deps if you want to call it from D directly later. Latency comparable to faster-whisper.
  - [ ] **OpenAI Whisper API** (cloud, paid): one HTTPS call per utterance, $0.006/min, ~300–600 ms typical. Worth it only if your hardware can't run local Whisper fast enough.
  - [ ] **Deepgram batch** (cloud, paid): also fine; not better than Whisper for this shape since you're not using their streaming partials.
- [ ] If cloud: provision API key, put in a `.env`, confirm not committed.

Avoid: Vosk (lower accuracy, the streaming advantage is wasted here), Web Speech API (no endpointing control, Chrome-tied), Deepgram streaming (paying for partials you don't consume).

### Audio routing for OBS

You need the character voice to land in OBS as its own audio source, separately from (or replacing) your mic.

- [ ] **Install a virtual audio cable.** Options:
  - [ ] [VB-CABLE](https://vb-audio.com/Cable/) — single virtual cable, free.
  - [ ] [VoiceMeeter Banana](https://vb-audio.com/Voicemeeter/banana.htm) — 3 virtual inputs + mixing/EQ, donation-ware. Worth it if you want to keep your real voice as a separate quieter channel, push-to-character-voice toggle, etc.
- [ ] Set the TTS playback device to the virtual cable's input.
- [ ] In OBS, add an Audio Input Capture pointing at the virtual cable's output.
- [ ] Decide: does your real voice still go to stream (gated/ducked while character speaks), or is character voice the only audio?

### Where the pipeline runs

- [ ] **Pick architecture.** Options:
  - [ ] **Standalone local Python script** (recommended). Direct: mic → STT → spawn `sapi4out.exe` → play WAV. No HTTP, minimum latency. Add a hotkey (push-to-talk or toggle).
  - [ ] Extend existing `sapi4.exe` web server with a `/stt` WebSocket. Adds 10–30 ms per round trip and complicates the deploy. Only worth it if you want browser-side capture.
  - [ ] Run STT in browser (Web Speech API) → existing `/SAPI4` endpoint. Lowest-effort prototype but you're at Google's mercy for STT.

### Push-to-talk vs always-on

- [ ] **Pick interaction model.**
  - [ ] Always-on with VAD endpointing — natural feel, risk of transcribing background chatter / your own breathing.
  - [ ] Push-to-talk (hold key) — zero false triggers, breaks immersion slightly, easier to debug.
  - [ ] Toggle (key starts/stops listening) — middle ground.
- [ ] Pick a hotkey that doesn't conflict with OBS scene-switching.

### Voice character

- [ ] Lock in which TruVoice voice you're using. The 16 voices in `sapi4limits.exe` output vary a lot — try them with a representative phrase before committing.
- [ ] Lock in pitch + speed. SAPI4 takes `pitch` and `speed` params; pick once, document the values.
- [ ] (Optional) Plan a second "alt" voice for character emotion variants (excited, whispered, etc.).

## Build checklist

### Scaffolding

- [ ] Add a `stt/` directory to repo root (Python lives separately from D/C++).
- [ ] `pyproject.toml` or `requirements.txt` with pinned deps.
- [ ] `.env.example` with `DEEPGRAM_API_KEY=` etc. Add `.env` to `.gitignore`.
- [ ] Decide Python version (3.11+ recommended for `asyncio` + `sounddevice` ergonomics).

### Audio capture

- [ ] Mic capture with `sounddevice` (PortAudio binding) — pick the right input device by name, not index.
- [ ] Resample to whatever the STT engine wants (Deepgram: 16 kHz mono PCM; Whisper: 16 kHz mono).
- [ ] Chunk at 20–30 ms (640–960 samples at 16 kHz).
- [ ] Sanity-check: plot a few seconds of capture to confirm no clipping / drops.

### VAD + endpointing

- [ ] Pick a VAD: `webrtcvad` (lightweight C lib, good default) or Silero VAD (PyTorch-based, more accurate, heavier).
- [ ] Implement: roll an audio buffer while VAD reports speech; when silence runs > N ms (start at 300 ms), close the buffer and submit it to STT.
- [ ] Tune the silence threshold against your own speech cadence — natural conversation has 150–300 ms inter-word gaps that you must NOT treat as utterance-end.
- [ ] Set a max utterance length (e.g. 30 s) to flush even if no silence detected (catches stuck states).
- [ ] If push-to-talk: skip VAD entirely; the hotkey defines the buffer boundaries.

### STT integration

- [ ] Wire up chosen engine as a **one-shot batch call**: full utterance buffer in, transcript text out.
- [ ] Test with representative phrases (long sentence, short interjection, filler word "um", a name from your stream lexicon).
- [ ] Measure cold-start: first call to faster-whisper after process start loads the model — pre-warm with a dummy call at startup so the first real utterance isn't slow.
- [ ] Add a kill-switch / safe word that mutes the TTS output (so you can pause without taking off the headset).

### TTS dispatch

- [ ] Call `sapi4out.exe <voice> <pitch> <speed> <text>` from Python (`subprocess.run` with timeout).
- [ ] Parse the WAV path from stdout, same way `app.d` does today.
- [ ] Play the WAV through the virtual cable device.
- [ ] Delete the temp WAV after playback.

### Output playback

- [ ] Use `sounddevice.play()` or stream the WAV with `soundfile` to the virtual cable device.
- [ ] Pre-warm the audio device on startup (first play has ~50–100 ms cold-start latency).
- [ ] If queueing multiple utterances: decide cancel-vs-queue policy when new transcript arrives while old one is still playing.

### Hotkey / control

- [ ] Global hotkey listener (`pynput` or `keyboard` — `keyboard` needs admin on Windows).
- [ ] Status indicator: console line, system tray icon, or OBS browser source showing "listening / speaking / idle."

## Latency measurement

- [ ] Add timestamps at each pipeline stage; log `(t_speech_end, t_stt_done, t_synth_done, t_audio_first_sample)` per utterance.
- [ ] Compute the four deltas above per utterance, write to a CSV.
- [ ] Speak ~20 representative phrases, look at p50/p95 of each delta. Identify the worst stage first.
- [ ] Re-measure after each tuning change. Don't tune blind.

## Go-live checklist (before first stream)

- [ ] Run the pipeline for 10 min, verify no memory leak / file-handle leak (`sapi4out.exe` writes temp WAVs).
- [ ] Confirm OBS picks up the virtual cable output, audio level is healthy.
- [ ] Test that the kill-switch / safe word actually mutes within < 200 ms.
- [ ] Plan for STT failure mode: what happens if Deepgram drops the WebSocket mid-stream? Local fallback or just silence?
- [ ] Decide what your viewers see on screen if the character voice fails — silent gap, "[voice down]" overlay, manual fallback to text-to-speech via a chat command?
- [ ] Have a back-out plan: hotkey to instantly disable the whole pipeline and revert to your raw mic.

## Stretch / nice-to-haves (don't block first stream)

- [ ] Sentence-level prosody: punctuate transcripts and pause SAPI4 between sentences instead of running them together.
- [ ] Profanity / forbidden-word filter pre-TTS (avoid making the character say what you didn't mean).
- [ ] Custom dictionary for character-specific names / in-jokes (most cloud STT APIs support this).
- [ ] Speaker diarization if you have a co-host — only transcribe your channel.
- [ ] Save transcripts + audio to disk per session for clip review.
- [ ] Web dashboard for live status, latency graph, last 10 transcripts — extend the existing `sapi4.exe` server for this part since it's not in the hot path.

## Open questions to resolve as you build

- Does the character voice fully replace your real voice on stream, or layer with it?
- Are you OK with a 200–400 ms perceived "lip sync" delay between your mouth moving on cam and the character voice playing?
- If yes — do you want the cam to play with a matched delay so it lines up?
- What happens for non-speech sounds (laughing, coughing) — transcribed-and-spoken (cursed), or filtered (better)?
