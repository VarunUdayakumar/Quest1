# README — Dialogue-to-Frame Extraction Notebook

This notebook takes a video link + a line of dialogue, and outputs the timestamp, frame number, matched text, and the actual video frame — but only if the character is confirmed **on-screen** while saying it.

See `approach.md` for the full pipeline breakdown.

## Setup

### 1. Runtime / GPU
This notebook needs a GPU (Whisper transcription + LR-ASD face detection both run much faster/more reliably on GPU, and `faster-whisper` is configured for `device="cuda"`).

In Colab:
1. Go to **Runtime → Change runtime type**
2. Set **Hardware accelerator** to **GPU** (T4 is sufficient)
3. Click **Save**, then **Runtime → Restart session** if you changed it mid-session

Without a GPU, the `WhisperModel(..., device="cuda", ...)` call will fail — either switch `device` to `"cpu"` (slower) or make sure a GPU runtime is attached.

### 2. Dependencies
The first cell installs everything needed:
```
!pip install -q yt-dlp faster-whisper rapidfuzz
```
Later cells also clone and set up `LR-ASD` (for active speaker detection) and install `python_speech_features` and a pinned version of `scenedetect`. Run cells top to bottom — some of these installs must happen before the LR-ASD step runs.

## How to use it

1. **Set your video URL and target phrase**
   In the video download cell, set:
   ```python
   url = "your video link here"
   ```
   In the phrase-matching cell, set:
   ```python
   target_phrase = "the dialogue line you're looking for"
   ```

2. **Run all cells top to bottom.** Rough flow:
   - Download the video (`yt-dlp`)
   - Transcribe it (`faster-whisper`) — this is the slowest step, scales with video length
   - Fuzzy-match your `target_phrase` against the transcript → gives timestamp + confidence score
   - Copy the matched window into LR-ASD's expected folder and run active speaker detection
   - Check if the speaker is on-screen at that timestamp
   - If confirmed on-screen: print timestamp, frame number, text, and display the extracted frame
   - If not: prints that it's off-screen/narration, no frame shown

3. **Adjust if needed:**
   - `score_threshold` in `find_phrase()` — lower it if a real match isn't being found (default 75)
   - `PAD` in the ASD window cell — increase if the timing looks slightly off
   - `window_slack` in `find_phrase()` — increase if your phrase has more transcription variance expected

## Known issue: Colab disconnects mid-run

You may hit an error like:
```
[object CloseEvent]
Error: [object CloseEvent]
```
or
```
await connected: disconnected
```

This is **Colab's frontend losing its websocket connection to the runtime** — not a bug in the notebook code. It tends to happen after long-running, resource-heavy cells (the video download and Whisper transcription steps in particular).

**What to do:**
1. Try **Runtime → Reconnect** first.
2. If that doesn't work, either:
   - **Run All** again from the top (Runtime → Run all), or
   - **Run cell by cell** instead, so you can see exactly which step it drops on and avoid re-running the expensive steps (download/transcription) unnecessarily.
3. If it keeps happening, check **Runtime → Change runtime type** to confirm a GPU is still allocated — Colab sometimes reclaims runtimes after heavy usage (especially on the free tier).
