# Video Generation System – Technical Specification

## 1. System Overview

A distributed pipeline automates content ingestion, media preparation, task assembly, and final rendering.  
Operators only place assets in `raw/`; worker nodes manage the rest automatically.

---

## 2. Directory Hierarchy

```bash
project_root/
│
├── raw/
│   ├── story/<category>/<story_name>/
│   │   ├── link/<uuid>.txt
│   │   └── voice/chapter_0001.mp3
│   ├── video/<category>/<uuid>.mp4
│   ├── frame/<category>/<uuid>.mp4
│   └── music/<category>/<uuid>.mp3
│
├── scanned/
│   ├── story/<category>/<story_name>/
│   │   ├── chunk_0001.txt
│   │   └── voice/chapter_0001.mp3
│   ├── video/<category>/mp4_w3840_h2160/<duration>_<uuid>.mp4
│   ├── frame/<category>/mp4_w1920_h1080/<duration>_<uuid>.mp4
│   └── music/<category>/mp3/<duration>_<uuid>.mp3
│
├── tasks/<task_uuid>/
│   ├── metadata.json
│   ├── voice.mp3 -> ../../scanned/story/<category>/<story_name>/voice/chapter_0001.mp3
│   ├── music.mp3 -> ../../scanned/music/<category>/mp3/<duration>_<uuid>.mp3
│   └── video_list.txt
│
├── processing/<task_uuid>/
│   ├── ...
│
├── output/<task_uuid>/
│   └── final.mp4
│
└── trash/
    ├── invalid/
    └── failed/
```

---

## 3. Worker Roles

### Main Worker

- Watches `raw/` directories.
- Validates and relocates files into `scanned/`.
- Generates new tasks when voice files are ready.
- Monitors processing node health and assigns tasks.

### Render Worker Node

- Pulls one available task.
- Moves it to `processing/`.
- Executes FFmpeg render sequence.
- Uploads final output to `output/`.
- Downloads next task in parallel during current render.

---

## 4. File Handling Logic

| Source | Validation | Destination Example | Notes |
|--------|-------------|----------------------|-------|
| `.txt` under `link/` | Crawl URL, clean HTML → plain text chunks | `scanned/story/.../chunk_0001.txt` | Split into ≤10k words per file |
| `.mp4` under `video/` or `frame/` | Extract width, height, duration | `scanned/frame/mp4_w1920_h1080/<duration>_<uuid>.mp4` | Reject if corrupted |
| `.mp3` under `music/` | Validate codec, duration | `scanned/music/mp3/<duration>_<uuid>.mp3` | Normalize volume |
| `.mp3` under `voice/` | Validate speech length | `scanned/story/.../voice/chapter_0001.mp3` | Must exist for task generation |

---

## 5. Task Metadata Format

**metadata.json**

```json
{
  "task_id": "uuid-1234",
  "category": "fantasy",
  "story_name": "dragon_flight",
  "voice_file": "voice.mp3",
  "voice_duration": 125.7,
  "music_file": "music.mp3",
  "video_files": [
    "video_part1.mp4",
    "video_part2.mp4"
  ],
  "render": {
    "resolution": "3840x2160",
    "frame_rate": 30,
    "logo": "logo.png",
    "output_format": "mp4"
  }
}
```

---

## 6. FFmpeg Rendering Pipeline

### Template Command

```bash
ffmpeg -y -hide_banner -nostdin \\
  -f concat -safe 0 -i "video_list.txt" \\
  -i "music.mp3" -i "voice.mp3" -i "logo.png" \\
  -filter_complex "
    [3:v]scale=iw:-1[logo];
    [0:v]scale=3840:2160:flags=fast_bilinear:force_original_aspect_ratio=decrease,
         pad=3840:2160:(ow-iw)/2:(oh-ih)/2:color=black,format=yuv420p[base];
    [base][logo]overlay=20:20[vout];
    [1:a]volume=0.25[bg];
    [2:a]volume=1.0[vc];
    [bg][vc]amix=inputs=2:duration=longest[aout]" \\
  -map "[vout]" -map "[aout]" \\
  -c:v h264_qsv -preset medium -r 30 -t <voice_duration> \\
  "final.mp4"
```

### Notes

- Intel CPU: `-c:v h264_qsv`
- NVIDIA GPU: `-c:v h264_nvenc`
- Apple Silicon: `-c:v h264_videotoolbox`
- Duration = voice length

---

## 7. Concurrency and Optimization

### Process Model

| Thread | Responsibility | Max Concurrent |
|---------|----------------|----------------|
| File Watcher | Detect new files in `raw/` | 1 |
| Task Preparer | Generate task folders | 1 |
| Downloader | Pre-fetch next task | 1 |
| Renderer | Run FFmpeg job | 1 |
| Uploader | Push completed output | 1 |

### Parallel Strategy

- Render worker downloads next task while rendering the current one.
- On render completion:
  - Start next render immediately.
  - Upload previous result concurrently.

```bash
Time →
| Download(T1) | Render(T1)   | Upload(T1)
               | Download(T2) | Render(T2) | Upload(T2)
```

---

## 8. Failure & Recovery

| Failure Type | Action |
|---------------|---------|
| File unreadable | Move to `trash/invalid/` |
| Metadata extraction fail | Log + move to `trash/invalid/` |
| FFmpeg render error | Move task to `trash/failed/` with stderr log |
| Upload fail | Retry up to 3 times, then move to `failed/` |

---

## 9. Logging

### Standard Format

```bash
[ISO_TIMESTAMP] [LEVEL] [COMPONENT] [MESSAGE]
```

**Example**

```bash
2025-11-10T18:03:44Z INFO main_worker "Moved file raw/video/action/abc.mp4 → scanned/video/mp4_w3840_h2160/12.4_abc.mp4"
2025-11-10T18:05:21Z WARN render_worker "Low disk space: 2.3GB remaining"
```

---

## 10. Performance Metrics

| Metric | Description | Unit |
|---------|--------------|------|
| render_utilization | % time spent rendering | % |
| avg_render_time | Time per output video | sec |
| task_throughput | Completed tasks/hour | count |
| io_overhead | Download+upload delay | sec |

---

## 11. Example Task Flow

1. Operator adds `raw/story/fantasy/dragon_flight/voice/chapter_0001.mp3`.
2. Main worker prepares task:
   - Picks 1 random music and multiple videos.
   - Generates `tasks/<uuid>/`.
3. Render worker downloads task.
4. FFmpeg merges assets.
5. Output uploaded to `output/<uuid>/final.mp4`.
6. Logs success.
