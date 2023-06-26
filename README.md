# ffmpeg-koraktor

An occasionally-growing selection of FFmpeg invocations that have proven handy in various situations.

*I'm calling it "Koraktor" for [reasons](https://de.wikipedia.org/wiki/Koraktor).*

Some of these have been grabbed straight from `history | grep ffmpeg` during the initial writeup of this list – I haven't tested everything for working-tude in current versions of FFmpeg.

There's more in [this Hacker News thread](https://news.ycombinator.com/item?id=26747207) (some of which I found interesting enough to include here for future reference).

*Related:* [Fred's ImageMagick Scripts](http://www.fmwconcepts.com/imagemagick/index.php).

## Cropping a letterboxed originally-vertical 1080p video

```sh
ffmpeg -i input.mp4  -vf "crop=606:1080:657:0" -crf 24 result.mp4
```

Numbers (format `w:h:x:y`, where `x:y` is counted from the top left corner) might differ depending on interpolation. And `-crf 24` is a quality setting – lower is better, 18 is basically lossless.


## Straightening (rotating and cropping) a video

For example:

```sh
ffmpeg -i DSCF3983.MOV  -vf "rotate=-PI/90:ow='iw*0.92':oh='ih*0.92'" out.mp4
```

This rotates a video by `-π/90`, i.e. one degree counterclockwise (you'll have to convert your degrees to radians). The resulting video would end up with black triangles in the corners, so we crop in by 5% via `ow='iw*0.95':oh='ih*0.95'`. This'll depend on the angle (you could use trigonometry to figure this out, alas, trial & error requires less thought).


## Speeding up a video (+ audio) by 400% while setting a different framerate

```sh
ffmpeg -i video.mov -filter:v "setpts=0.25*PTS,fps=60" -filter:a "atempo=4.0" output.mp4
```

Explanation: `setpts=0.25*PTS` speeds up the video (to 0.25 of its original duration, *i.e.*, a 400% speedup), `fps=60` sets the target framerate, and the audio filter `atempo=4.0` speeds up the audio by 400%.

## Making a video black and white

Via [Alastair Tse](https://twitter.com/liquidx/status/1524626492784664577):

```sh
ffmpeg -i "${x}" -vf hue=s=0,eq=brightness=0.1 "${OUTPUT_DIR}/${x}"
```


## Turning a series of images into a GIF (or GIF, depending on your pronunciation preferences)

This is taken from a [BIT-101](https://www.bit-101.com/blog/2021/09/more-ffmpeg-tips/) post where things in explained in more detail and with more background information. Assuming you've got a series of images `frames/frame_%04d.png`, there's a two-step process:

1. Have FFmpeg analyze your images to generate the ideal palette (since GIFs are limited to 256 colors):

    ```sh
    ffmpeg -i frames/frame_%04d.png -vf palettegen palette.png
    ```

2. Assemble the GIF:

    ```sh
    ffmpeg -framerate 30 -i frames/frame_%04d.png -i palette.png -filter_complex paletteuse out.gif
    ```

## Turning a video into a faux slomo video (while also fiddling with the colors and setting a certain output quality)

I've deployed a variant of this to generate the video in [this tweet](https://twitter.com/doersino/status/1424809570262781957). Note that anything more than a slowdown factor of 2 (I've used 8 for the video in the tweet) is invariably going to lead to significant artifacts which make the whole thing basically unusable.

```sh
ffmpeg -i in.mov -vf "eq=contrast=1.1:brightness=0.08:saturation=1.08,minterpolate='fps=60',setpts=2*PTS" -c:v libx264 -preset fast -crf 20 -c:a aac -b:a 192k out-slow.mp4
```

(The `minterpolate='fps=60',setpts=2*PTS` part's the important part. This whole thing doesn't do anything to the audio, so it's best to also strip it off using `ffmpeg -i out-slow.mp4 -c copy -an out-slow-silent.mp4`, which doesn't reencode the video.)

(Note that while experimenting, you can use the `-y` flag to keep FFmpeg from asking whether to override files.)


## Scaling a video down to 50% of its original size and halving the frame rate

This is handy for screen recordings (and accordingly assumes the input `fps` was 60):

```sh
ffmpeg -i in.mov -vf "scale=iw*.5:ih*.5,fps=30" -crf 18 out.mp4
```


## Saving all keyframes to images

This is a good way of getting some high-quality wallpapers from a Ghibli movie, I suppose (via [Hacker News](https://news.ycombinator.com/item?id=26747821)).

```sh
ffmpeg -i video.mp4 -vf "select=eq(pict_type\,I)" -vsync vfr video-%03d.png
```

It's pretty quick, too (the `fps` count in FFmpeg's output refers to the image files written, not the progress in the video file).


## Extracting 1 second of video every 90 seconds

This one can come in handy to condense a very long video that doesn't look good as a timelapse – think bird nest building, dashcam footage, or reminding yourself what happened in a movie you've seen a while ago (via [Hacker News](https://news.ycombinator.com/item?id=26748165)).

```sh
ffmpeg -i video.mp4 -vf "select='lt(mod(t,90),1)',setpts=N/FRAME_RATE/TB" -af "aselect='lt(mod(t,90),1)',asetpts=N/SR/TB" out.mp4
```

The `lt(mod(t,90),1)` bit (repeated twice!) steers the length `1` and interval `90`.


## Cutting out a corner

Everybody's done it: Having recorded a neat little video, you realize that a corner of the frame is obscured by your finger or part of whatever you've mounted or perched your camera on.

Assuming the blemish is in the bottom right corner and covers a 250×50 pixel area, the following invocation will crop the video to avoid it, then resize it back to 1920×1080 pixels. (Note that `1030 = 1080 - 50` and `1831 = 1920 * (1030 / 1080)`.)

```sh
ffmpeg -i video.mov -vf "crop=1831:1030:0:0,scale=1920:-2" -crf 18 result.mp4
```

As usual, you can employ `ffplay` to check whether the crop is correct:

```sh
ffplay -i video.mov -vf "crop=1831:1030:0:0,scale=1920:-2"
```

I required this after recording [a 30-minute timelapse of some clouds passing by](https://www.youtube.com/watch?v=JcH9GHx_Rrk) where I failed to notice that part of my little phone tripod thingy was in frame.


## Almost losslessly converting a video from `mjpeg`/`pcm_s16le` to `h264`/`aac`

When recording video, my aging camera, a Pentax K-7, produces AVI files that each contain an MJPEG-encoded video stream and whatever audio format that in the headline is. The following command compresses those videos to roughly half their size with *zero* perceptible quality loss.

```sh
ffmpeg -i raw.AVI -c:v libx264 -preset fast -crf 18 -c:a aac -b:a 192k compressed.mp4
```

* `-c:v libx264` selects the new video codec.
* `-preset fast` does the work [faster than the default at the expense of file size](https://trac.ffmpeg.org/wiki/Encode/H.264). I think this steers how many frames are looked at for diffing and stuff.
* `-crf 18` sets the compression quality – values range from 0 (lossless but uselessly massive) to 51 (just no), with 18 [being visually lossless](https://trac.ffmpeg.org/wiki/Encode/H.264) while still cutting filesizes in half or so.
* `-c:a aac` selects the new audio codec.
* `-b:a 192k` sets the bitrate to 192k – this is roughly equivalent to a 256k MP3, *i.e.*, lossless unless you're in a soundproof room with *very* expensive speakers and your ears are made of literal magic.


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

This particular invocation works well for very shaky video, *i.e.*, kestrels filmed with a 800 mm equivalent lens and basically non-existent in-camera stabilization. Play with the parameters, perhaps look them up in the documentation.


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


## Converting a video to a more digestible codec

```sh
ffmpeg -i video.mov -vcodec h264 -acodec mp2 video2.mp4
```


## Test patters

More of a curiosity, really (via [Hacker News](https://news.ycombinator.com/item?id=26751167)).

```sh
ffmpeg -f lavfi -i testsrc=d=60:s=1920x1080:r=24,format=yuv420p -f lavfi -i sine=f=440:b=4 -b:v 1M -b:a 192k -shortest output-testsrc.mp4
ffmpeg -f lavfi -i testsrc2=d=60:s=1920x1080:r=24,format=yuv420p -f lavfi -i sine=f=440:b=4 -b:v 1M -b:a 192k -shortest output-testsrc.mp4
ffmpeg -f lavfi -i smptebars=d=60:s=1920x1080:r=24,format=yuv420p -f lavfi -i sine=f=440:b=4 -b:v 1M -b:a 192k -shortest output-smptebars.mp4
```


## Bonus: Averaging a series of images with ImageMagick

```sh
convert image1.jpg ... imageN.jpg -evaluate-sequence Mean average.jpg
```

Also possible: Pixel-wise min/max/median.

```sh
convert image1.jpg ... imageN.jpg -evaluate-sequence Min minimum.jpg
convert image1.jpg ... imageN.jpg -evaluate-sequence Max maximum.jpg
convert image1.jpg ... imageN.jpg -evaluate-sequence Median median.jpg
```

More possible modes of the `-evaluate-sequence` tool can be found out by running `convert -list evaluate`.


## Bounus: Making a GIF with ImageMagick

```sh
convert -delay 15 -loop 0 -dispose previous * -resize 1000x1000\> animated.gif
```

The delay between successive frames must be given in hundreths of a second, `-loop` should be zero for a proper GIF, and `-resize` isn't required but it's usually sensible.
