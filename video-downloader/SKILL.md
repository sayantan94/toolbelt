---
name: video-downloader
description: Downloads videos and audio from URLs using yt-dlp. Use when user says "download this video", "grab this clip", "save this video", "download as mp3", or pastes a URL from YouTube, TikTok, Instagram, Twitter/X, Reddit, Vimeo, or any video platform.
---

# Video Downloader

## Important: Help the User Get the Right URL

If the user doesn't provide a URL, or the URL fails, guide them on how to get the correct video URL for their platform:

| Platform | How to get the video URL |
|----------|--------------------------|
| **YouTube** | Copy the URL from the browser address bar, or click Share > Copy link |
| **Twitter/X** | Click the tweet, then copy the URL from the browser address bar. It should look like `https://x.com/user/status/123456789` |
| **Reddit** | Click the post, copy the URL from the address bar. For v.redd.it videos, use the full Reddit post URL, not the v.redd.it short link |
| **Other sites** | Right-click on the video > "Copy video address" or "Copy link address". If that doesn't work, copy the page URL from the address bar |

**Tips to share with the user:**
- If a short/shared link fails, try the full page URL from the browser address bar instead
- For embedded videos, right-click the video and look for "Copy video address"
- Private or login-required videos may need `--cookies-from-browser chrome`

## Step 1: Check Tools

Run both checks. If either is missing, stop and show install commands — do NOT proceed.

```bash
command -v yt-dlp && command -v ffmpeg
```

If missing:
- macOS: `brew install yt-dlp ffmpeg`
- Linux: `sudo apt install ffmpeg && pip install yt-dlp`

## Step 2: Fetch Video Info

Before downloading, always fetch the video title and available formats first:

```bash
yt-dlp --no-playlist --print title --print duration_string -F "URL"
```

Show the user:
- Video title
- Duration
- Available qualities (e.g., 1080p, 720p, 480p)

## Step 3: Ask the User

Before downloading, confirm these with the user:

1. **Format**: Video (MP4) or Audio (MP3)?
   - Default to MP4 if they already said "video" or didn't specify
   - Default to MP3 if they said "audio", "mp3", "song", or "music"

2. **Quality**: Which resolution?
   - Show available options from Step 2
   - Default to best quality if they don't care

3. **Save location**: Where to save the file?
   - Suggest `~/Downloads` as default
   - If user specifies a path, use that
   - Create the directory if it doesn't exist

Skip asking if the user already made their preferences clear (e.g., "download this video to my Desktop in 720p").

## Step 4: Download

**Video (MP4):**
```bash
yt-dlp --no-playlist -f "bestvideo+bestaudio/best" --merge-output-format mp4 -o "<save_path>/%(title)s.%(ext)s" "URL"
```

**Video at specific quality (e.g., 720p):**
```bash
yt-dlp --no-playlist -f "bestvideo[height<=720]+bestaudio/best[height<=720]" --merge-output-format mp4 -o "<save_path>/%(title)s.%(ext)s" "URL"
```

**Audio (MP3):**
```bash
yt-dlp --no-playlist -x --audio-format mp3 -o "<save_path>/%(title)s.%(ext)s" "URL"
```

Always use `--no-playlist` unless the user explicitly asks for a full playlist.

## Step 5: Confirm Result

After the command finishes:

1. Run `ls -lh` on the output file to show filename and size
2. Tell the user the full path where the file was saved
3. If it failed, show the error and suggest a fix from the table below

## Errors

| Error | What to do |
|-------|------------|
| 403 Forbidden | Update yt-dlp: `yt-dlp -U`. If still failing, try `--cookies-from-browser chrome` |
| Unsupported URL | URL is not from a supported site — ask user to double-check the link |
| Video unavailable | Video is private, deleted, or region-locked — nothing to do |
| Merge fails | Retry with `-f "best"` instead (uses a pre-merged format) |
| No audio in output | Retry with `--merge-output-format mp4` or `-f "best"` |
