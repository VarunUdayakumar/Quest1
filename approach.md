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
`yt-dlp` downloads the video from any platform (YouTube, OK.ru, etc.) without needing platform-specific code. Streaming/chunked ingestion (processing without a full download) was the ideal, but full download is the simpler, more robust fallback that's implemented now.

**2. Transcribe**
`faster-whisper` (medium model) transcribes the audio with word-level timestamps — this only depends on audio, so it's resolution/fps-independent.

**3. Locate the phrase**
A custom sliding-window fuzzy matcher (`rapidfuzz`) scores windows of the transcript against the target phrase, allowing for a few words of slack in window size. This absorbs Whisper's transcription errors (missed/extra/misheard words) and returns a confidence score along with the best-matching timestamp.

**4. Pad the window**
A small buffer is added before/after the matched timestamp to absorb minor timing misalignment before checking for an on-screen speaker.

**5. Check if on-screen**
`LR-ASD` (active speaker detection) runs on the padded clip. Rather than just checking "is a face visible," it tracks faces and scores whether a face's lip movement is synced with the audio — closer to real lip-sync verification than a naive presence check. The highest sync score within the padded window determines on-screen vs. off-screen.

**6. Output**
If the on-screen check passes:
- Timestamp is formatted as `HH:MM:SS.sss`
- Frame number is computed using the *original* video's real fps (via `ffprobe`), not the 25fps LR-ASD internally re-encodes to — keeps it accurate regardless of source frame rate
- Frame image is extracted directly from the original video with `ffmpeg`

If the check fails, the pipeline reports off-screen/narration and doesn't show a frame.

## What's solid vs. what's a fallback

| Piece | Status |
|---|---|
| Multi-platform ingestion | Solid — `yt-dlp` |
| Confidence-scored phrase matching | Solid — fuzzy sliding window |
| On-screen detection | Near best-case — real lip-sync scoring, not just face presence |
| Frame-rate robustness | Solid — uses native fps, not the ASD re-encode rate |
| Streaming ingestion | Fallback — full download used instead |
| Dubbing / multi-speaker disambiguation | Not fully solved |
