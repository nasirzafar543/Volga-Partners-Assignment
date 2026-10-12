# Audio Transcription Pipeline

A small notebook-ready pipeline that takes an audio file, transcribes it using Groq's hosted Whisper API, and returns the text with per-segment timestamps.

```
audio file -> validate (Task 1) -> transcribe (Task 2) -> timestamped output (Task 3)
```

## Setup

```bash
pip install soundfile groq python-dotenv
```

Create a `.env` file next to the notebook (get a free key at console.groq.com):

```
GROQ_API_KEY=your-key-here
```

Add `.env` to `.gitignore` so the key is never committed.

## Usage

```python
audio = load_audio("sample.wav")
result = transcribe(audio)

print(timestamped_text(result))
```

## Design Decisions

### Task 1: Accepting audio

- **Validate before doing any work.** The script checks that the file exists, has a supported extension (WAV or MP3), is not empty, is under the size limit, and is readable. A bad file fails immediately with a clear message instead of failing halfway through an API call.
- **Header-only read.** `soundfile.info` returns duration, sample rate and channels without loading the audio into memory, so validation stays fast even for large files.
- **Typed output.** The result is an `AudioInput` dataclass, giving the next step a clean object instead of loose variables.

### Task 2: Transcription

- **Groq-hosted Whisper.** Free tier, no GPU or model download, fast inference, and accuracy comparable to OpenAI's Whisper. Trade-offs: rate limits, and audio leaves the machine.
- **`whisper-large-v3-turbo`.** Near large-v3 accuracy at lower latency. It can be swapped for `whisper-large-v3` if accuracy matters more than speed.
- **Language auto-detection.** The language is detected by default, and can be passed in explicitly (e.g. `"en"`) to skip detection.
- **Size guard.** The 25 MB API limit is checked locally using Task 1's metadata, so we do not upload a file that would be rejected.
- **Retries.** The SDK's built-in `max_retries` handles rate limits and network errors without custom code.
- **Provider-independent output.** Results are mapped into our own `Transcript` and `Segment` dataclasses, so downstream code does not depend on the API response shape and the provider can be swapped easily.

### Task 3: Segmented transcription with timestamps

- **Segment timestamps come from the API.** Requesting `verbose_json` returns them with the transcription, so there is no extra call, cost or latency.
- **Two output formats.** A readable format (`[00:00:00.000 --> 00:00:04.200] text`) for quick inspection, and JSON for downstream use such as search, summaries or storage.
- **`HH:MM:SS.mmm` format.** It is standard and easy to convert to SRT or VTT subtitles later.
- **Rounded values.** Times are rounded to milliseconds to avoid float noise like `4.199999`.

## Known Limitations

- Only WAV and MP3 are accepted. Other formats (M4A, OGG, video) would need an ffmpeg conversion step.
- Files over 25 MB are rejected. Longer audio would need to be compressed and split into chunks, with timestamps shifted per chunk when merging.
- Segment-level timestamps only, not word-level.
- Processing is synchronous. Production use would need a job queue (see Part 2).
