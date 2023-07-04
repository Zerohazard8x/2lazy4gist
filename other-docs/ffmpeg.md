`ffmpeg -framerate 24 -i img%03d.png output.mp4`

`ffmpeg -f concat -safe 0 -i ./copy.txt -c:v copy -c:a copy -c:s copy -movflags +faststart [output]`

`ffmpeg -i input.wav -map 0:a -filter:a volumedetect -f null /dev/null`

---

`ffmpeg -y -stream_loop -1 -i "OST\one.mp3" -i [converted .h264] -map 0:a:0 -map 1:v:0 -shortest -c:v h264_nvenc -rc vbr -cq:v 22 -qmin 22 -qmax 22 -t 00:15:00 vid1.mp4`
`ffmpeg -y -stream_loop -1 -i "OST\one.mp3" -sseof -600 -i [converted .h264] -map 0:a:0 -map 1:v:0 -shortest -c:v h264_nvenc -rc vbr -cq:v 22 -qmin 22 -qmax 1  vid2.mp4`

---

```
mpv --profile=pseudo-gui [--demuxer-readahead-secs=0] --demuxer-lavf-o-add=rtbufsize=2147480000--hwdec=auto av://dshow:video=“Game Capture HD60 S”:audio=“Game Capture HD60 S Audio”
```

- `ffplay -fflags nobuffer -flags low_delay -vf setpts=0 -bufsize 99999G -probesize 1M -sync ext -f dshow -i video=“rapoo camera”:audio=“Microphone (rapoo camera)” -af alimiter`
- `ffmpeg -list_devices true -f dshow -i dummy`
- **_NOTE_**: VLC can do this too

---

_Count frames_

```
ffprobe -v error -select_streams v:0 -count_packets -show_entries stream=nb_read_packets -of csv=p=0 input.mp4
```

---

```
ffmpeg -i .\gt1.mp4 -vf fps=30 -c:v libx264 -preset ultrafast -c:a libopus ex-hw1.mp4 (Pseudo-lossless)
```

---

**_FPS reduction motionblur | Example: 240fps -> 120fps -> 60fps_**

```
ffmpeg -i [input] -vf tblend=all_mode=average,framestep=2,tblend=all_mode=average,framestep=2,fps=60 [output]
```

---

```
ffmpeg -i [input] -c:v copy -c:a copy -c:s mov_text -movflags +faststart [output.mp4]
```

```
ffmpeg -i [input] -c:v copy -c:a copy -c:s srt -movflags +faststart [output.mkv]
```

```
ffmpeg -i [input] -c:v copy -c:a copy -c:s copy -movflags +faststart output.mp4
```

---

```
-vf zscale=t=linear,tonemap=reinhard,zscale=t=bt709,format=yuv420p
-vf zscale=t=linear,tonemap=reinhard,zscale=t=bt2020,format=yuv420p10
-vf format=p010,hwupload,tonemap_opencl=tonemap=mobius:p=bt709:t=bt709:m=bt709:format=nv12,hwdownload,format=nv12
```

---

`youtube-dl -f 94 -g https://www.youtube.com/watch?v=21X5lGlDOfg` - grab live link