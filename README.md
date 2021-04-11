# ffmpeg-koraktor

An occasionally-growing selection of FFmpeg invocations that have proven handy in various situations.

*I'm calling it "Koraktor" for [reasons](https://de.wikipedia.org/wiki/Koraktor).*

Some of these have been grabbed straight from `history | grep ffmpeg` during the initial writeup of this list – I haven't tested everything for working-tude in current versions of FFmpeg.


## Vertically and horizontally stacking videos

I used this while prototyping [earthacrosstime](https://github.com/doersino/earthacrosstime).

```sh
ffmpeg -i topleft.mp4 -i bottomleft.mp4 -filter_complex vstack=inputs=2 left.mp4
ffmpeg -i topright.mp4 -i bottomright.mp4 -filter_complex vstack=inputs=2 right.mp4
ffmpeg -i left.mp4 -i right.mp4 -filter_complex hstack=inputs=2 whole.mp4
```

The first line takes `topleft.mp4` and `bottomleft.mp4` and stacks them vertically, *i.e.*, on top of each other, yielding a video with the same width but twice the height. The last line does the same, but horizontally. Taken together, the three lines montage four videos into a 2×2 grid.


## Trimming, colorgrading, adding an end card to a video, and fading it in and out

Used to turn a raw video into [this YouTube upload](https://www.youtube.com/watch?v=46z6EHJtr5E).

```sh
ffmpeg -i raw.mov -loop 1 -i endcard.png -ss 00:01:30 -t 00:00:59 -filter_complex "[0:v]eq=contrast=1.1:brightness=0.01 [corr]; [1:v]fade=in:st=140:d=2:alpha=1 [end]; [corr][end] overlay=0:0:enable='between(t,140,149)' [final]; [final]fade=in:st=90:d=1,fade=out:st=148:d=1" out.mp4
```

The bits do:

* `-i raw.mov` loads in the video.
* `-loop 1 -i endcard.png` loads in the end card (`-loop 1` is required, I think to turn it into an "infinite" video instead of just a single frame, which is needed for fading it in).
* `-ss 00:01:30 -t 00:00:59` trims the video to only include the interesting bit. Despite this, all future time references are with regard to the original.
* `[0:v]eq=contrast=1.1:brightness=0.01 [corr]` makes the video prettier and stores in in "corr".
* `[1:v]fade=in:st=140:d=2:alpha=1 [end]` fades in the end card at 140s (i.e. 50s into the trimmed span) for 2s and stores in "end".
* `[corr][end] overlay=0:0:enable='between(t,140,149)' [final]` overlays the two, showing the end card starting at 140s (note that this is also where the fade starts) until 149s (end of video).
* `[final]fade=in:st=90:d=1,fade=out:st=148:d=1` fades the whole video in and out from black.

Note that `endcard.png` must be the same resolution as the video.

When you run this command, it will look like it's doing nothing for a minute or so – I think it spends that time "scanning" to the start timestamp in a really inefficient way (it probably runs through the filters here already).

Note that this will preserve the original audio, as well as the frame rate etc. (The latter of which was the whole reason for doing this – iMovie emits 30 fps, which I felt was unacceptable given that my iPhone records at 60fps. That's every other frame lost! *Gasp!*)

*I'm sure there's a way more elegant way of doing all this.*

Similar, for [another video](https://www.youtube.com/watch?v=_gCZxBp2fOk):

```sh
ffplay -i raw.AVI -vf "eq=contrast=1.3:brightness=0.3"  # preview

ffmpeg -i raw.AVI -loop 1 -i endcard.png -t 00:00:59 -filter_complex "[0:v]crop=in_h:in_h,eq=contrast=1.3:brightness=0.3 [corr]; [1:v]fade=in:st=50:d=2:alpha=1 [end]; [corr][end] overlay=0:0:enable='between(t,50,59)' [final]; [final]fade=in:st=0:d=1,fade=out:st=58:d=1" out.mp4
```

The `ffplay` bit allows for preview of the "colorgrading".

The main bits are very similar to the previous command. `crop=in_h:in_h` makes the video square, and there was no need for trimming off the start here, which made things more straightforward. Note that here, `endcard.png` must be the same resolution as the *output* video.


## Converting all images in a directory into a video

```sh
ffmpeg -framerate 24 -pattern_type glob -i '*.jpg' -pix_fmt yuv420p out.mp4
```

Or, as an alias in your `.bashrc` (mind the quotes!):

```sh
alias jpg2mp4='ffmpeg -framerate 24 -pattern_type glob -i '"'"'*.jpg'"'"' -pix_fmt yuv420p out.mp4'
```

There's also this:

```sh
ffmpeg -framerate 30 -i %05d.jpg -c:v libx264 -r 30 -pix_fmt yuv420p barge.mp4
```


## Stabilizing a video

Assuming the filename lives in `$VID`:

```sh
ffmpeg -i $VID -vf vidstabdetect=shakiness=9:show=1 dummy.mp4
ffmpeg -i $VID -vf vidstabtransform,unsharp=5:5:0.8:3:3:0.4 dummy2.mp4
```

The result will be `dummy2.mp4`. Feel free to give it a better name.

This particualr invocation works well for very shaky video, *i.e.*, kestrels filmed with a 800 mm equivalent lens and basically nonexistant in-camera stabilization. Play with the parameters, perhaps look them up in the documentation.


## Concatenating a list of videos

Write a newline-separated list of video paths to `mylist.txt`, then:

```sh
ffmpeg -f concat -i mylist.txt -c copy output.ts
```

This is handy for video from streaming services. I think I used this to rip a personal copy of the [Harmontown movie](https://www.imdb.com/title/tt3518988/) from whatever now-defunct streaming service it had been released on (or, rather, merge the ripped `.ts` files into a usable video).

Similarly, if it's only a few videos:

```sh
ffmpeg -i "concat:heatvisionandjack_1.mpg|heatvisionandjack_2.mpg|heatvisionandjack_3.mpg" -c copy heatvisionandjack.mpg
```


## Scaling (and padding, if required) all videos in the current directory to a fixed size

```sh
for i in $(ls -1); do ffmpeg -i $i -vf 'scale=1080:720,pad=1280:720:(ow-iw)/2:(oh-ih)/2' scaled_$i; done
```


## Slowing a video down to 50% of the original speed

Handy for making raw slow-motion footage actually look slo-mo.

```sh
ffmpeg -i 2015-01-03.mp4 -filter:v "setpts=2.0*PTS" output.mp4
```

[See](https://trac.ffmpeg.org/wiki/How%20to%20speed%20up%20/%20slow%20down%20a%20video) also.


## Extracting frames from a 60 fps video

This starts at 65 seconds:

```sh
ffmpeg -ss 00:01:05 -i video.mp4 -filter:v fps=fps=60/1 frames/ffmpeg_%3d.png
```


## Making the audio twice as loud

```sh
ffmpeg -i video.mp4 -filter:a "volume=2" video-loud.mp4
```

## Removing audio

```sh
ffmpeg -i video.mov -c copy -an video_wosound.mov
```


## Extracting audio

```sh
ffmpeg -i Desktop/video.mp4 Desktop/audio.mp3
```


## Adding WAV audio to a video as MP3

```sh
ffmpeg -i video.mp4 -i audio.wav -c:v copy -c:a mp3 -strict experimental -map 0:v:0 -map 1:a:0 video_with_audio.mp4
```


## Transcoding M4A to MP3

```sh
ffmpeg -i music.m4a -acodec libmp3lame -ab 256k music.mp3
```


## Trimming a video

```sh
ffmpeg -i in.mp4 -ss 01:01:00 -t 00:00:10 out.mp4
```

Note that `-t` isn't the end, it's the duration.


## Convert a video to a more digestible codec

```sh
ffmpeg -i video.mov -vcodec h264 -acodec mp2 video2.mp4
```
