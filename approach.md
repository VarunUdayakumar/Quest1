# Approach: Dialogue-to-Frame Extraction

## Goal
Given a video link and a line of dialogue, find and display:
- The timestamp the line is spoken
- The frame number
- The matched text
- The video frame itself

...but only if the character is actually **on-screen** while saying it (not narration/dubbing/off-camera).

## Pipeline

**1. Ingest the video**
`yt-dlp` downloads the video from any platform (YouTube, OK.ru, etc.) without needing platform-specific code, pulling the best available MP4 video+audio streams and merging them. Streaming/chunked ingestion (processing without a full download) was the ideal, but full download is the simpler, more robust fallback that's implemented now.

**2. Transcribe**
`faster-whisper` (medium model, GPU) transcribes the audio with word-level timestamps — this only depends on audio, so it's resolution/fps-independent.

**3. Locate the phrase**
A custom sliding-window fuzzy matcher (`rapidfuzz`) scores windows of the transcript against the target phrase. Window sizes range from `target_length - slack` to `target_length + slack` words (default slack: 3) to absorb Whisper insertions/omissions, and each window is scored with `fuzz.ratio`. The highest-scoring window above a threshold (default: 75) is returned with its start/end timestamp and confidence score; this absorbs Whisper's transcription errors (missed/extra/misheard words).

**4. Define the ASD window**
The matched phrase's start/end timestamps set the window that active speaker detection will run over. The code has a `PAD` variable for adding a buffer before/after the match to absorb minor timing misalignment, but it's currently set to `0` — no padding is applied yet, so the ASD window exactly matches the matched phrase span. This is a known gap: tightening/loosening this buffer is an easy tuning knob once real-world mistiming shows up.

**5. Check if on-screen**
`LR-ASD` (active speaker detection) runs on the (currently unpadded) window. Rather than just checking "is a face visible," it tracks faces and scores whether a face's lip movement is synced with the audio — closer to real lip-sync verification than a naive presence check. LR-ASD always re-encodes its working clip to 25 fps internally, so the matched timestamps are converted to frame indices at 25 fps before comparing against face-track frames. The highest sync score across all tracked faces within the window is taken; a peak score `>= 0` is treated as on-screen speech (LR-ASD's convention: positive scores indicate active speaking), otherwise the line is reported as off-screen/narration.

**6. Output**
If the on-screen check passes:
- Timestamp is formatted as `HH:MM:SS.sss`
- Frame number is computed using the *original* video's real fps (via `ffprobe`), not the 25 fps LR-ASD internally re-encodes to — keeps it accurate regardless of source frame rate
- Frame image is extracted directly from the original video with `ffmpeg` (seek-then-decode at the matched start timestamp)

If the check fails, the pipeline reports off-screen/narration and doesn't show a frame.
