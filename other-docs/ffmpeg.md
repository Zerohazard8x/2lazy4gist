## Table of Contents

1. [Standard libx264 Encode](#standard-libx264-encode)
2. [Audio Channel Conversion (Stereo → Mono)](#audio-channel-conversion-stereo--mono)
3. [Image Export / Import](#image-export--import)
4. [Merging Video and Audio](#merging-video-and-audio)
5. [Cutting and Splitting Videos](#cutting-and-splitting-videos)
6. [Mirroring and Flipping](#mirroring-and-flipping)
7. [Concatenation](#concatenation)

    - [Basic Concatenation](#basic-concatenation)
    - [Concatenate + Scale](#concatenate--scale)
    - [Concatenate + Scale + Filters](#concatenate--scale--filters)
    - [Concat via Text File](#concat-via-text-file)

8. [Frame Extraction and Counting](#frame-extraction-and-counting)
9. [High-Framerate → Lower-Framerate with Motion Blur](#high-framerate--lower-framerate-with-motion-blur)
10. [Stream Copy & Subtitles](#stream-copy--subtitles)
11. [Color-Space & Tone-Mapping Filters](#color-space--tone-mapping-filters)
12. [YouTube-DL Live-Stream Link](#youtube-dl-live-stream-link)

---

## 1. Standard libx264 Encode

Use libx264 to re-encode a QuickTime MOV into an MP4 at 720p/29.97 fps.

```bash
ffmpeg -i "C:\Users\adria\Downloads\IMG_3239.MOV" \
  -vf "scale=1280:720,fps=29.97" \
  -c:v libx264 \
  -crf 24 \
  "C:\Users\adria\Downloads\IMG_3239.mp4"
```

-   **CRF (Constant Rate Factor)**:

    -   `18` = visually lossless → `24` = more compression (lower quality).
    -   For fully lossless: use `-crf 0` (or set `qmin=0:qmax=0`).

---

## 2. Audio Channel Conversion (Stereo → Mono)

Extract just the right channel (or left) from a stereo video and output a mono video:

```bash
ffmpeg -i [stereo-video-input.mp4] \
  -codec:v copy \
  -af "pan=mono|c0=FR" \
  [mono-video-output.mp4]
```

-   Replace `FR` with `FL` if you want the left channel instead.

---

## 3. Image Export / Import

### 3.1 Exporting Frames from a Video

Save every frame of a video as sequential PNG files (e.g., `frame001.png`, `frame002.png`, …):

```bash
ffmpeg -i [file.mp4] anything%03d.png
```

### 3.2 Creating a Video from Images

Combine an image sequence (`img001.png`, `img002.png`, …) into a video at a specified framerate:

```bash
ffmpeg -framerate XX \
  -i [img-sequence-pattern%03d.png] \
  output.mp4
```

-   Example: `-framerate 24 -i img%03d.png output.mp4`

---

## 4. Merging Video and Audio

### 4.1 Merge Video + Looping Audio (Audio Restart When It Ends)

Silence the original video’s audio entirely and replace it with a looping WAV:

```bash
ffmpeg -y \
  -stream_loop -1 -i "audio.wav" \
  -i "video.mp4" \
  -map 0:a:0 \
  -map 1:v:0 \
  -shortest \
  output.mp4
```

-   `-stream_loop -1` loops the audio infinitely.
-   `-shortest` ensures the output stops when the video ends.

### 4.2 Merge Audio with Image Sequence

Take a looping audio track and combine it with an image sequence, encoding with libx264:

```bash
ffmpeg -y \
  -stream_loop -1 -i [audio.wav] \
  -framerate [e.g. 29.97] -i [.../%03d.png] \
  -map 0:a:0 \
  -map 1:v:0 \
  -shortest \
  -vcodec libx264 \
  output.mp4
```

-   Similar logic: audio loops (`-stream_loop -1`) and stops when images run out (`-shortest`).

### 4.3 Extract a Single Frame from a Video

Grab one frame (e.g., at timestamp `00:00:00.165`) and save as `out.png`:

```bash
ffmpeg -ss 00:00:00.165 \
  -i "C:\Users\adria\Downloads\IMG_9974.MOV" \
  -vframes 1 \
  out.png
```

---

## 5. Cutting and Splitting Videos

Split a large file at the 50 min mark, creating two 50 min segments:

```bash
ffmpeg -i largefile.mp4 \
  -t 00:50:00 -c copy smallfile1.mp4 \
  -ss 00:50:00 -c copy smallfile2.mp4
```

-   `-t 00:50:00` → first segment is the first 50 minutes.
-   `-ss 00:50:00` → second segment starts at 50 minutes and continues to the end.

---

## 6. Mirroring and Flipping

Flip a mirrored or rotated video horizontally or vertically:

-   **Horizontal Flip** (mirror left/right):

    ```bash
    -vf hflip
    ```

-   **Vertical Flip** (mirror top/bottom):

    ```bash
    -vf vflip
    ```

Example usage inside a full command:

```bash
ffmpeg -i input.mp4 -vf hflip -c:v libx264 output_flipped.mp4
```

---

## 7. Concatenation

### 7.1 Basic Concatenation (Four Inputs)

Concatenate `vid1.mp4` + `vid2.mp4` + `vid3.mp4` + `vid4.mp4` (all have same codec/format):

```bash
ffmpeg \
  -i vid1.mp4 \
  -i vid2.mp4 \
  -i vid3.mp4 \
  -i vid4.mp4 \
  -filter_complex "
    [0:v][0:a]
    [1:v][1:a]
    [2:v][2:a]
    [3:v][3:a]
    concat=n=4:v=1:a=1 [v][a]
  " \
  -map "[v]" \
  -map "[a]" \
  -c:v libx264 \
  output.mp4
```

### 7.2 Concatenate + Scale (Three Inputs)

Concatenate three inputs, scaling/resampling each to 1280×720 at 29.97 fps and setting square pixels (SAR=1):

```bash
ffmpeg \
  -i vid1.mp4 \
  -i vid2.mp4 \
  -i vid3.mp4 \
  -filter_complex "
    [0:v]fps=29.97,scale=1280:720,setsar=1[0scale];
    [1:v]fps=29.97,scale=-1:720,setsar=1,pad=1280:720[1scale];
    [2:v]fps=29.97,scale=-1:720,setsar=1,pad=1280:720[2scale];
    [0scale][0:a]
    [1scale][1:a]
    [2scale][2:a]
    concat=n=3:v=1:a=1[v][a]
  " \
  -map "[v]" \
  -map "[a]" \
  -sws_flags lanczos \
  -c:v libx264 \
  output.mp4
```

-   `scale=-1:720` preserves aspect ratio, then `pad=1280:720` centers/pads to 1280×720.
-   `setsar=1` ensures square pixels.

### 7.3 Concatenate + Scale + Filters (Four Inputs)

Concatenate four inputs, then apply scaling to 360 px height and add a stereo audio format:

```bash
ffmpeg \
  -i vid1.mp4 \
  -i vid2.mp4 \
  -i vid3.mp4 \
  -i vid4.mp4 \
  -filter_complex "
    [0:v][0:a]
    [1:v][1:a]
    [2:v][2:a]
    [3:v][3:a]
    concat=n=4:v=1:a=1 [v][a];
    [v]scale=-1:360:flags=lanczos,pp=dr[vFinal];
    [a]aformat=cl=stereo,extrastereo[aFinal]
  " \
  -map "[vFinal]" \
  -map "[aFinal]" \
  -c:v libx264 \
  output.mp4
```

-   Uses `lanczos` scaling plus `pp=dr` (denoise/repair) on video.
-   Resamples audio to stereo with `aformat=cl=stereo,extrastereo`.

### 7.4 Concat via Text File

When inputs share codecs/formats, you can list filenames in a text file (`copy.txt`):

```
file 'vid1.mp4'
file 'vid2.mp4'
file 'vid3.mp4'
...
```

Then run:

```bash
ffmpeg -f concat -safe 0 -i ./copy.txt \
  -c:v copy -c:a copy -c:s copy \
  -movflags +faststart \
  output.mp4
```

-   `-safe 0` disables path safety checks (allows absolute paths).
-   `-c:s copy` also carries over any subtitle streams.

---

## 8. Frame Extraction and Counting

### 8.1 Count Total Video Frames

Use `ffprobe` to count packets (frames) in the primary video stream:

```bash
ffprobe -v error \
  -select_streams v:0 \
  -count_packets \
  -show_entries stream=nb_read_packets \
  -of csv=p=0 \
  input.mp4
```

### 8.2 Pseudo-Lossless Encode at 30 FPS

Re-encode `gt1.mp4` at 30 fps using `ultrafast` preset and Opus audio, aiming for near-lossless:

```bash
ffmpeg -i gt1.mp4 \
  -vf fps=30 \
  -c:v libx264 -preset ultrafast \
  -c:a libopus \
  ex-hw1.mp4
```

---

## 9. High-Framerate → Lower-Framerate with Motion Blur

Reduce `240 fps` → `120 fps` → `60 fps` while blending frames to simulate motion blur:

```bash
ffmpeg -i [input_240fps.mp4] \
  -vf tblend=all_mode=average,framestep=2, \
       tblend=all_mode=average,framestep=2, \
       fps=60 \
  [output_60fps.mp4]
```

-   First `tblend+framestep` halves 240 → 120 with blending.
-   Second `tblend+framestep` halves 120 → 60 with blending.
-   Final `fps=60` ensures the output is precisely 60 fps.

---

## 10. Stream Copy & Subtitles

Copy all streams (video, audio, subtitles) without re-encoding, with faststart:

```bash
ffmpeg -i [input] \
  -c:v copy \
  -c:a copy \
  -c:s mov_text \
  -movflags +faststart \
  [output.mp4]
```

-   For Matroska MKV with embedded SRT subtitles:

    ```bash
    ffmpeg -i [input] \
      -c:v copy \
      -c:a copy \
      -c:s srt \
      -movflags +faststart \
      [output.mkv]
    ```

-   To copy subtitles as-is (no re-muxing):

    ```bash
    ffmpeg -i [input] \
      -c:v copy \
      -c:a copy \
      -c:s copy \
      -movflags +faststart \
      output.mp4
    ```

---

## 11. Color-Space & Tone-Mapping Filters

### 11.1 SDR (Rec.709) Tone-Mapping from HDR

Convert HDR to SDR (Rec.709) with Reinhard tonemapping:

```text
-vf zscale=t=linear,tonemap=reinhard,zscale=t=bt709,format=yuv420p
```

### 11.2 HDR10 (Rec.2020) Tone-Mapping from HDR

Convert using Rec.2020 → Rec.709 10-bit:

```text
-vf zscale=t=linear,tonemap=reinhard,zscale=t=bt2020,format=yuv420p10
```

### 11.3 Hardware-Accelerated Mobius Tone-Mapping

Use OpenCL to apply Mobius operator (Rec.709 input/output):

```text
-vf format=p010,hwupload,tonemap_opencl=tonemap=mobius:p=bt709:t=bt709:m=bt709:format=nv12,hwdownload,format=nv12
```

### 11.4 Automatic White-Balance (Grayworld)

Let FFmpeg attempt to auto-fix white balance:

```text
-vf grayworld
```

---

## 12. YouTube-DL Live-Stream Link

Retrieve the direct (M3U8) URL for a YouTube video/stream:

```bash
youtube-dl -f 94 -g https://www.youtube.com/watch?v=21X5lGlDOfg
```

-   `-f 94` chooses format code 94 (720p MP4, H.264)
-   `-g` prints the final streaming URL to stdout

---

### Notes & Tips

-   Always ensure your FFmpeg build is up-to-date.
-   Escape or quote Windows paths that contain spaces or special characters.
-   When concatenating files with different codecs/resolutions, you must re-encode or use filter graphs to match parameters.
-   Use `-preset veryfast` (or similar) for faster encoding if you don’t need ultrafast.
-   For lossless video editing, use `-crf 0` with `-preset ultrafast` or consider FFV1 if you need a long-term archival format.