**_Standard libx264 encode_**

```
ffmpeg -i C:\Users\adria\Downloads\IMG_3239.MOV -vf "scale=1280:720,fps=29.97" -c:v libx264 -crf 24 C:\Users\adria\Downloads\IMG_3239.mp4
```

- CRF = 18 (best) to 24 (worst)
- Lossless codec - CRF 0 or qmin/qmax 0

---

**_Left/right channel --> mono_**
`ffmpeg -i [stereo video input] -codec:v copy -af "pan=mono|c0=FR" [mono video output]`

---

- Exporting images = `ffmpeg -i [file] anything%03d.png`
- Putting images into a video = `ffmpeg -framerate xx -i [img sequence] output.mp4`

---

**_Merge video and audio (orig audio completely silenced, audio restarts when out of time)_**

```
ffmpeg -y -stream_loop -1 -i "audio.wav" -i "video.mp4" -map 0:a:0 -map 1:v:0 -shortest output.mp4
```

---

**_Merge audio with img sequence (orig audio completely silenced, audio restarts when out of time)_**

```
ffmpeg -y -stream_loop -1 -i [audio.wav] -framerate [e.g. 29.97] -i [...\%03d.png] -map 0:a:0 -map 1:v:0 -shortest -vcodec libx264 output.mp4
```

- Get frame = `ffmpeg -ss [time e.g. 00.165] -i C:\Users\adria\Downloads\IMG_9974.MOV -vframes 1 out.png`

---

**_Cut video at 50 mins in, and then create another 50 minute video ending 50 mins after said cut_**

```
ffmpeg -i largefile.mp4 -t 00:50:00 -c copy smallfile1.mp4 -ss 00:50:00 -c copy smallfile2.mp4
```

- Unmirror mirrored/flipped video = `-vf hflip`/`-vf vflip`

---

**_Concatenate_**

```
ffmpeg -i vid1.mp4 -i vid2.mp4 -i vid3.mp4 -i vid4.mp4 -filter_complex "[0:v] [0:a] [1:v] [1:a] [2:v] [2:a] [3:v] [3:a] concat=n=4:v=1:a=1 [v] [a]" -map "[v]" -map "[a]" -c:v libx264 output.mp4
```

---

_Concatenate and scale_

```
ffmpeg -i vid1.mp4 -i vid2.mp4 -i vid3.mp4 -filter_complex "[0:v]fps=29.97,scale=1280:720,setsar=1[0scale];[1:v]fps=29.97,scale=-1:720,setsar=1,pad=1280:720[1scale];[2:v]fps=29.97,scale=-1:720,setsar=1,pad=1280:720[2scale];[0scale][0:a][1scale][1:a][2scale][2:a]concat=n=3:v=1:a=1[v][a]" -map "[v]" -map "[a]" -sws_flags lanczos -c:v libx264 output.mp4
```

---

_Concatenate and scale w/filters_

```
ffmpeg -i vid1.mp4 -i vid2.mp4 -i vid3.mp4 -filter_complex "[0:v] [0:a] [1:v] [1:a] [2:v] [2:a] [3:v] [3:a] concat=n=4:v=1:a=1 [v][a];[v]scale=-1:360:flags=lanczos,pp=dr[vFinal];[a]aformat=cl=stereo,extrastereo[aFinal]" -map "[vFinal]" -map "[aFinal]" -c:v libx264 output.mp4
```
