---
title: "Posy's Motion Amplification - Part 1: Python and FFmpeg "
layout: post
author: Martino Trapanotto
tags: [motion_extraction, python, computer vision]
---

# Motion Extraction

A while ago I came across a [beautiful video by creator Posy](https://www.youtube.com/watch?v=NSS6yAMZF78), which captured my interest and sparked my curiosity. 

The technology described is quite simple, but the creativity on display and ingenuity in applying it to various situations hit me, and I wanted to reconstruct the effect to run whenever I wanted.

In the video, Posy describes only very briefly how the process works, and by referencing mostly what a video editor or other visual artists might see, not what a developer might see or think exactly.

I decided that a good position to start might be getting a nice video from Pexels, such as this [nice waterfall](https://www.pexels.com/video/beautiful-sight-of-nature-2098988/) here reproduced in GIF form thanks to [gifski](https://github.com/ImageOptim/gifski), cause NeoCities doesn't allow videos for free users.

![](/assets/posts/MA1/waterfall.gif "Waterfall")

## The Basics

The problem should be quite easy, take the difference between two subsequent frames and either display that or compose/post-process/videomagick your way into something interesting.

Notice that here in OpenCV we can't just subtract the frames directly, as the values of the underlying NumPy matrices are `uint8` and these can underflow or have other nasty numerical errors if not handled correctly. `cv2.subtract` does that for us, or we could probably use the NumPy arrays underneath directly

```python
import cv2
from pathlib import Path

# Change to your video file path
video_path = Path("resources/raw/waterfall.mp4")

# Load as OpenCV source
cap = cv2.VideoCapture(filename=video_path.as_posix())

# Prepare the OpenCV window
window_name = "Motion Extraction Demo"

# Load the first frame
working, frame = cap.read()

dimensions = frame.shape

# Width and height can be accessed via indexing
width = dimensions[1]
height = dimensions[0]

output_frames = []
# If frame loads are succesful keep on going
while working:
    # Store the previous frame
    prev_frame = frame.copy()
    # Load the new one
    working, frame = cap.read()
    if not working:
        break
    # Compute the difference
    motion_frame = cv2.subtract(frame, prev_frame)

    # Combine horizontally
    composite_frame = cv2.hconcat([frame, motion_frame])
    # Scale down and show
    cv2.imshow(window_name, cv2.resize(composite_frame, (width//2, height//2)))
    # Store for later
    output_frames.append(motion_frame)

    if cv2.waitKey(5) == ord('q'):
        break
cap.release()
cv2.destroyAllWindows()

```
![](/assets/posts/MA1/watrerfall_py1.gif "Motion Extraction example 1 with python")

You might notice a black frame. 
The simple approach means that the first frame is always compared to itself, so the first frame is blank. This is true for all our Python results.

Another method I found uses the `cv2.addWeighted`, in a very similar fashion to the method Posy describes in the video.

This also shares the characteristic of centering static pixels not on black, but on a neutral gray, highlighting more movement and providing a clear sense of direction in the gradient.

We can also see more nuanced movement, likely as the contrast between the individual gray levels is better than the lower ends of the black ones. There might be some processing possible also here, via gamma correction or enhanced contrast.


```python
import cv2
from pathlib import Path

# Change to your video file path
video_path = Path("resources/raw/lake.mp4")

# Load as OpenCV source
cap = cv2.VideoCapture(video_path.as_posix())

# Prepare the OpenCV window
window_name = "Motion Extraction Demo"

# Load the first frame
working, frame = cap.read()

dimensions = frame.shape

# Width and height can be accessed via indexing
width = dimensions[1]
height = dimensions[0]

output_frames = []
# If frame loads are succesful keep on going
while working:
    # Store the previous frame
    prev_frame = frame.copy()
    # Load the new one
    working, frame = cap.read()
    if not working:
        break
    
    # Compute the difference
    motion_frame = cv2.addWeighted(frame, .5, cv2.bitwise_not(prev_frame), 0.5, 0)

    # Combine horizontally
    composite_frame = cv2.hconcat([frame, motion_frame])
    # Scale down and show
    cv2.imshow(window_name, cv2.resize(composite_frame, (width//2, height//2)))
    # Store for later
    output_frames.append(motion_frame)

    if cv2.waitKey(5) == ord('q'):
        break
cap.release()
cv2.destroyAllWindows()
```

![](/assets/posts/MA1/watrerfall_py2.gif "Motion Extraction example 2 with python"){:loading="lazy"}

Here we also build a quick script to take the motion frames and save them to disk.


```python
import cv2
from pathlib import Path

# Define the codec and create a VideoWriter object
fourcc = cv2.VideoWriter_fourcc(*'XVID')
dimensions = output_frames[0].shape

# Width and height can be accessed via indexing
width = dimensions[1]
height = dimensions[0]
out = cv2.VideoWriter('output.avi', fourcc,  30.0, (width, height))

# Write frames to the output file
for frame in output_frames:
    out.write(frame)

# Release the VideoWriter
out.release()

```

## Variable delays
After the basics of the effect, Posy expands on using longer time offsets. This allows to enhance more subtle movements and details, or slower changes.

How to make it in code?
I can think of two approaches: a video buffer or multiple reader heads.

Using a buffer means reading the video frame by frame and storing the last N frames in a buffer. This might require more memory, but also means reading the video only once, so potentially less I/O load and perhaps making it easier for live feeds.

Multiple reader heads means opening the file multiple times and reading with an offset between these, no need to store all the intermediate frames. Considering how modern video compression reconstruct most frames from previous and future frames, this might not be that much lighter on memory, unless the equivalent buffer is massive.

### Two heads are better than one

Accessing the same file multiple times is not a big deal in OpenCV, and this approach means we can also change the delay easily.

```python
import cv2
from pathlib import Path

# Change to your video file path
video_path = Path("resources/raw/waterfall.mp4")

# Load as OpenCV source
cap = cv2.VideoCapture(video_path.as_posix())
cap_past = cv2.VideoCapture(video_path.as_posix())

# Prepare the OpenCV window
window_name = "Motion Extraction Demo"

# Load the first frame
working, frame = cap.read()
frame_past = frame.copy()
frame_offset_target = 5
frame_offset = 0


dimensions = frame.shape
# Width and height can be accessed via indexing
width = dimensions[1]
height = dimensions[0]

output_frames = []
# If frame loads are succesful keep on going
while working:
    # Store the previous frame
    if frame_offset <= frame_offset_target:
        working, frame = cap.read()
        if not working:
            break
        
    if frame_offset >= frame_offset_target:
        working, frame_past = cap_past.read()
        if not working:
            break
    
    
    # Compute the difference
    motion_frame = cv2.addWeighted(frame, .5, cv2.bitwise_not(frame_past), 0.5, 0)

    # Combine horizontally
    composite_frame = cv2.hconcat([frame, motion_frame])
    # Scale down and show
    cv2.imshow(window_name, cv2.resize(composite_frame, (width//2, height//2)))
    # Store for later
    output_frames.append(motion_frame)

    if cv2.waitKey(5) == ord('q'):
        break
cap.release()
cv2.destroyAllWindows()
```
![](/assets/posts/MA1/watrerfall_delay5.gif "Motion Extraction example with 5 frame delay with python"){:loading="lazy"}

This works quite well, but the process is very aggressive on my RAM, as the usage goes up as it keeps going. My poor old laptop is having major issues with it, and I can't tell if it's a leak somewhere or is just a result of having to decompress two streams of the same video, separately.

### Memory games

Perhaps using two read heads causes some major I/O overhead, or the two different processes reading overcrowd everything, it doesn't matter, we can instead try reading once and storing the relevant frames in a small ring buffer.

Python's `collections` packages has some ready-made tricks for us to play with

```python
import collections
from pathlib import Path

import cv2

# Change to your video file path
video_path = Path("resources/raw/waterfall.mp4")

# Load as OpenCV source
cap = cv2.VideoCapture(video_path.as_posix())

# Prepare the OpenCV window
window_name = "Motion Extraction Demo"

# Load the first frame
working, frame = cap.read()

# Create buffer, define how many frames of delay
frame_buffer = collections.deque(maxlen=6)

dimensions = frame.shape

# Width and height can be accessed via indexing
width = dimensions[1]
height = dimensions[0]
frame = cv2.resize(frame, (width//2, height//2))

output_frames = []
# If frame loads are succesful keep on going
while working:
    # Store the previous frame
    frame_buffer.append(frame.copy())
    # Load the new one

    working, frame = cap.read()
    if not working:
        break
    
    frame = cv2.resize(frame, (width//2, height//2))
    # Compute the difference
    motion_frame = cv2.addWeighted(frame, .5, cv2.bitwise_not(frame_buffer[0]), 0.5, 0)

    # Combine horizontally
    composite_frame = cv2.hconcat([frame, motion_frame])
    # Scale down and show
    cv2.imshow(window_name, cv2.resize(composite_frame, (width//2, height//2)))
    # Store for later
    output_frames.append(motion_frame)

    if cv2.waitKey(5) == ord('q'):
        break
cap.release()
cv2.destroyAllWindows()
```
![](/assets/posts/MA1/watrerfall_delay10.gif "Motion Extraction example with 10 frame delay with python"){:loading="lazy"}

This again works, but with some incredibly harsh memory consumption rates.

It might be an issue of garbage collection, some OpenCV memory bug, or just that this is a lot harsher than it looks as a task, especially for Full HD video to run at real time speed. Maybe I should not use a cheap 2016 laptop to do this.

But regardless, Python is probably not the right choice for this kind of problem. 

We could move everything to C++ and manage memory manually, but I'm really not a fan of the experience of managing libraries and compiler instructions. I'm spoiled by `pip` and prefer my sysadmin gruel predigested and masticated, thank you.

Instead, we might have a better approach.

## FFmpeg
While working on the Python version and musing about the memory problems, I got curious: can you recreate this effect via FFmpeg?

FFmpeg would have a wide array for extra effects, ability to run on a lot of machines and on many video sources, including live streams and is a lot more efficient with resources. Also, no dependency hell!

After some tries, some pleading against the Obscure Monolith (the [FFmpeg docs](https://ffmpeg.org/ffmpeg-filters.html)), I consulted the Oracle ([Phind.com](https://www.phind.com)) and found out that, as always, FFmpeg has a command for that. Multiple ones, actually.

There are two techniques I found to reproduce the effect:

#### `tblend` in `subtract` mode
[`tblend`](https://ffmpeg.org/ffmpeg-filters.html#toc-tblend) is the temporal version of the `blend` filter, usually needed to merge different video sources together. This time variant filters subsequent frames from the same source.

Do note that you will have to move to RGB color space if you don't like your motion amplification on green screen:
```bash
ffplay /dev/video0 -vf "format=gbrp,tmix=frames=3:weights=-1 0 1:scale=2,format=yuv420p" 
```
![](/assets/posts/MA1/waterfall_ffmpeg_delay2.gif "Motion Extraction example with 2 frame delay with FFmpeg"){:loading="lazy"}

Think adding motion blur for example, or if used in `subtract` mode, to detect motion.

The result is quite simple and intuitive, but is limited to only two frames.

#### `tmix`

[`tmix`](https://ffmpeg.org/ffmpeg-filters.html#toc-tmix) is the true solution. It is a generalized temporal version of the `mix` command, making us able to combine frames at an arbitrary delay, and even on live stream video. FFmpeg's abilities always surprises me.

We simply have to declare the number of frames to process and how the weights are distributed among these frames.
The `weights` parameters describes how the frame buffer is combined, with a multiplier for each frame as an integer number in a sequence.

For example, if `frames=4`, then we would set `filter=-1 0 0 1`, so that the first frame is subtracted from the fourth, or vide versa. It should not matter much which it is.
For longer delays, add more zeroes.

```bash
ffmpeg -i sunset.mp4 -filter_complex "format=gbrp,split[v1][v2];[v1]tmix=frames=3:weights=-1 0 1:scale=2[ov1];[v2][ov1]blend=all_mode=multiply128,format=yuv420p[out]" -map "[out]"  test.mp4 -y
```
![](/assets/posts/MA1/waterfall_ffmpeg_delay8.gif "Motion Extraction example with 8 frame delay with FFmpeg"){:loading="lazy"}

The other trick we might want to do is splitting the input with `split`, giving us two streams that we can process in parallel, for example to extract the motion, blur it, and then combine back again to give a nice glow effect, similar to that shown in the video

```bash
ffmpeg -i sunset.mp4 -filter_complex "format=gbrp,split[v1][v2];[v1]tmix=frames=5:weights=-1 0 0 0 1:scale=2[ov1];[ov1]gblur=steps=6:sigma=5[ov2];[v2][ov2]blend=all_mode=addition,format=yuv420p[out]" -map "[out]" test.mp4 -y; ffplay test.mp4
```
![](/assets/posts/MA1/waterfall_ffmpeg_bloom.gif "Motion Extraction example with 2 frame delay with FFmpeg"){:loading="lazy"}

There is for sure a lot more to do from here on out, from multiple motion streams at different delays to more serious post-processing and recombination, more artistic passages using custom blend modes, or the `lut2` filter to merge the channels creatively. I hopw someone else will have some nicer ideas and more patience.

Sadly my creativity is quite limited, as is the detail of the video I'm working with, and my patience to try all the *undocumented* formulas and blend modes and methods that FFmpeg allows for, so I'll have to stop.

## More?

There is a lot more I'm curious to explore on this topic:
- Reading the YouTube comments on the video, many discussions arise about how the process can be interpreted or what other approaches it is similar to, from a low pass filter to temporal video compression, which makes me curious about some more mathematically oriented discourse
- Other YouTube addicts like me might already have seen a similar topic on a [Steve Mould Video]((https://www.youtube.com/watch?v=rEoc0YoALt0&t=687s)), which might also be fun to try and recreate in Python or FFmpeg (or C++ if needed) 
- The idea reminded me of a vaguely referenced topic from my Computer Vision class called Optical Flow, which might be nice to read more about
- The question of where did my RAM go is still open. Learning about some basics of memory profiling and investigating if this can be managed (again some compiled language might even be featured)

But I decided I want to at least bring myself a bit closer to the [Cult of Done](https://www.youtube.com/watch?v=bJQj1uKtnus), so this is done, and perhaps more will come. Or maybe not, but the ghosts are now here, and they keep good company.
