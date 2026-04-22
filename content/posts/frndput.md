---
author: NmToan
date: 2025-12-29T04:59:04.866Z
lastmod: 2025-12-29T13:39:20.763Z
title: MY FRIEND ARTIST
slug:  
featured: false
draft: false
tags:
  - Forensics
  
description:  |
  
---
# my friend artist — writeup

## Challenge info
- **Name:** my friend artist
- **Category:** misc / steg
- **Points:** 500
- **Attachment:** `my_first_song.wav`

## Goal
Recover the flag from the provided WAV file.

---

## 1. First observations
We are given a single audio file, `my_first_song.wav`, and the challenge text says:

> I recently received a song from my friend, who is a painter. He always wanted to learn to play music and recently stated that he found an excellent way to express himself in the medium. However, I am unable to see anything in this piece, so could you lend me an ear?

The important hint is that the friend is a **painter**, and the author says they are unable to **see anything** in the piece. That strongly suggests the audio may actually contain **visual information**.

So instead of treating the WAV like a normal song, we should suspect:
- spectrogram art, or
- oscilloscope / vectorscope art where stereo channels draw shapes.

---

## 2. Check the file type
First confirm the file is really just a WAV audio file.

```bash
file my_first_song.wav
```

Typical result:

```text
my_first_song.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16 bit, stereo 44100 Hz
```

Useful takeaways:
- PCM WAV
- stereo
- 44.1 kHz

The fact that it is **stereo** is important, because oscilloscope art often uses:
- **left channel = X axis**
- **right channel = Y axis**

---

## 3. Try normal audio stego checks
At this point, it is reasonable to do quick basic checks:

```bash
strings my_first_song.wav | head
exiftool my_first_song.wav
binwalk my_first_song.wav
```

Nothing useful appears from metadata or embedded files, so the interesting content is likely encoded in the waveform itself.

---

## 4. Consider the painter hint
The painter hint makes much more sense if the audio is meant to be **seen**, not heard.

A classic trick is to feed stereo audio into an X-Y oscilloscope or vectorscope. If the audio was crafted that way, plotting the left and right channels against each other reveals text or drawings.

So the next step is to visualize the stereo channels as coordinates.

---

## 5. Plot left channel vs right channel
A quick way is to use Python and scatter/line plot the stereo samples.

Example:

```python
import wave
import numpy as np
import matplotlib.pyplot as plt

with wave.open("my_first_song.wav", "rb") as w:
    n = w.getnframes()
    raw = w.readframes(n)
    data = np.frombuffer(raw, dtype=np.int16).reshape(-1, 2)

left = data[:, 0]
right = data[:, 1]

plt.figure(figsize=(8, 8))
plt.plot(left[::20], right[::20], linewidth=0.2)
plt.axis('equal')
plt.show()
```

This does not immediately give the full flag cleanly, but it confirms a crucial fact:

**the audio is drawing vector shapes**.

So we now know this is **oscilloscope / vectorscope art**.

---

## 6. Focus on the middle of the file
Looking at the whole file at once is messy because the shapes overlap over time.

The trick is to inspect **small time windows** or the **middle section** of the track, where the useful content appears.

One approach is to render short chunks as separate X-Y plots.

Pseudo-process:
- split the audio into small windows
- for each window, plot left vs right
- inspect the resulting frames

When doing that, the middle portion clearly shows **letters scrolling horizontally**.

So the WAV is effectively a moving vector-text display.

---

## 7. Reconstruct the text visually
There are several ways to continue:
- record vectorscope frames and read them manually
- align the frames into a panorama
- write code to “unscroll” the moving text
- or just inspect the centered composite and read the letters by eye

In this solve, the message was readable enough from the reconstructed vector text, so **manual visual decoding** was enough. OCR was not needed.

From the aligned frames, the text reads as:

```text
dr4w1n9 w1th s0und
```

The leetspeak digits are important:
- `a -> 4`
- `i -> 1`
- `g -> 9`
- `o -> 0`

---

## 8. Build the flag
The challenge uses the format:

```text
putcCTF{...}
```

So the final flag is:

```text
putcCTF{dr4w1n9_w1th_s0und}
```

---

## 9. Final answer
```text
putcCTF{dr4w1n9_w1th_s0und}
```

---

## Short solve summary
- The WAV was not meant to be listened to like a normal song.
- The hint about a **painter** suggested hidden **visual** content.
- Since the file is **stereo**, plotting **left vs right** reveals oscilloscope art.
- Inspecting the middle section showed scrolling vector text.
- Reading the text by eye gave `dr4w1n9 w1th s0und`.
- Wrapping that in the flag format gave:
  `putcCTF{dr4w1n9_w1th_s0und}`
