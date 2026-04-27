---
author: NmToan
date: 2025-12-29T04:59:04.866Z
lastmod: 2026-04-27T00:00:00.000Z
title: MY FRIEND ARTIST
slug:
featured: false
draft: false
tags:
  - Forensics
description: Write-up for hidden vector text in `my_first_song.wav`
---

# My Friend Artist Write-up

## Challenge Info

- **Name:** my friend artist
- **Category:** misc / steg
- **Points:** 500
- **Attachment:** `my_first_song.wav`

## Goal

Recover the flag from the provided audio file `my_first_song.wav`.

## Core Idea

The challenge text says the sender is a **painter**, and the narrator says they cannot **see anything** in the piece. That is a strong clue that the audio is not meant to be interpreted only by listening to it. Instead, it likely contains visual information.

The two most likely approaches are:

- spectrogram art,
- oscilloscope or vectorscope art using stereo channels as drawing coordinates.

After checking the file format, the second path turns out to be correct.

## 1. Check the Input File

First I confirmed the file type:

```bash
file my_first_song.wav
```

Typical result:

```text
my_first_song.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16 bit, stereo 44100 Hz
```

Important details:

- PCM WAV,
- stereo,
- `44100 Hz`.

The stereo property is especially important, because oscilloscope art often maps:

- the left channel to `X`,
- the right channel to `Y`.

## 2. Try Basic Audio Stego Checks

Before going deeper, I ran a few quick checks:

```bash
strings my_first_song.wav | head
exiftool my_first_song.wav
binwalk my_first_song.wav
```

No useful metadata or embedded files appeared, which suggests the hidden content is inside the waveform itself.

## 3. Follow the "Painter" Hint

The hint makes much more sense if the audio is meant to be **seen** instead of simply heard.

A classic trick is to feed stereo audio into an X-Y oscilloscope or vectorscope. In that setup:

- `Left` controls the horizontal axis,
- `Right` controls the vertical axis,
- the waveform becomes a drawing.

That makes visualizing the stereo sample pairs the next logical step.

## 4. Plot Left Against Right

I used Python to plot the two channels:

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

This did not reveal the full flag immediately, but it confirmed the most important fact:

**the WAV is drawing vector shapes.**

So the correct direction is indeed oscilloscope or vectorscope art.

## 5. Why the Middle of the File Matters

Plotting the entire file at once produces a messy overlap, because the content changes over time. It behaves more like moving vector text than a static image.

A better approach is:

1. split the audio into short time windows,
2. plot each window separately,
3. inspect the sequence of frames.

When doing that, the middle portion clearly shows **letters scrolling horizontally**.

This is the turning point of the solve.

## 6. Reconstruct the Hidden Text

From here, several approaches are possible:

- record vectorscope frames and read them manually,
- align frames into a panorama,
- write code to "unscroll" the motion,
- or inspect a suitable composite and read it by eye.

For this challenge, manual visual reading was enough. OCR was not necessary.

The text reads:

```text
dr4w1n9 w1th s0und
```

The leetspeak substitutions matter:

- `a -> 4`
- `i -> 1`
- `g -> 9`
- `o -> 0`

## 7. Build the Flag

The challenge uses this flag format:

```text
putcCTF{...}
```

So the final answer is:

```text
putcCTF{dr4w1n9_w1th_s0und}
```

## Conclusion

The reasoning chain is short and elegant:

1. read the hint and suspect hidden visual content,
2. confirm that the file is a stereo WAV,
3. rule out simple metadata or embedded-file tricks,
4. plot `left` and `right` as X-Y coordinates,
5. recognize oscilloscope art,
6. focus on the middle section to read the scrolling text,
7. recover `dr4w1n9 w1th s0und`,
8. wrap it in the challenge flag format.

Final flag:

```text
putcCTF{dr4w1n9_w1th_s0und}
```
