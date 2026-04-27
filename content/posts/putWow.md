---
author: NmToan
date: 2025-12-29T04:59:04.866Z
lastmod: 2026-04-27T00:00:00.000Z
title: WOW!
slug:
featured: false
draft: false
tags:
  - Forensics
description: Write-up of the pipeline from `report.pdf` to the recovered SSTV image channel `Y.png`
---

# Write-up: `report.pdf` to `Y.png`

This post documents the full path from the initial `report.pdf` file to the most useful analysis artifact, `Y.png`. The steps are organized in the same order as the actual solve: detect embedded audio in the PDF, extract the WAV stream, identify the SSTV mode, reconstruct the image, and finally derive the luminance channel for further work.

## Overview

At the beginning, the only provided file was:

- `report.pdf`

Two hints inside the document immediately stood out:

- `"I will transfer the data utilizing Papa Delta"`
- `"something similar to the galaxy"`

These suggest two key ideas:

1. the PDF likely contains an embedded payload rather than just text,
2. `"Papa Delta"` strongly resembles the naming used by SSTV `PD` modes.

That leads to the main hypothesis for the challenge:

**the PDF contains an audio signal that decodes into an image.**

## Full Pipeline

The workflow for this challenge can be summarized as:

1. inspect `report.pdf` for anything unusual,
2. identify and extract the embedded audio object,
3. determine that the signal is SSTV `PD290`,
4. write a decoder to reconstruct the image,
5. recover `pd290_mono32.png`,
6. separate channels and derive `Y.png`.

## 1. Inspect the Original PDF

I first checked the workspace and file metadata:

```powershell
Get-ChildItem -Force
rg --files
Get-Item report.pdf | Format-List *
```

The important observation is that `report.pdf` is about `5.2 MB`, which is much larger than a normal text-heavy PDF and strongly suggests a large embedded object.

## 2. Read Text and Objects from the PDF

Since the environment did not provide tools like `pdfinfo`, `qpdf`, or `exiftool`, I switched to `strings` and direct Python parsing.

Quick check:

```powershell
strings -n 6 report.pdf | Select-Object -First 300
```

The most interesting object description was:

```text
/Type /Sound /R 48000 /C 1 /B 16 /E /muLaw
/Length 5155787
/Filter /FlateDecode
```

This immediately suggests:

- the PDF contains a `Sound` object,
- the stream is compressed with `FlateDecode`,
- the embedded payload is large enough to be the main target.

## 3. Locate and Extract the Audio Object

To verify that, I enumerated stream objects and decompressed the `FlateDecode` ones. The key result was:

- `OBJ 25` is the embedded sound object,
- the compressed stream is about `5,155,788` bytes,
- after decompression it expands to about `56,203,108` bytes,
- the decompressed data starts with:

```text
RIFF....WAVEfmt ...
```

So the payload is a genuine WAV file.

I extracted it with:

```python
import re, zlib, pathlib
p = pathlib.Path('report.pdf').read_bytes()
m = re.search(rb'25\s+0\s+obj(.*?)stream\r?\n', p, re.S)
start = m.end()
end = p.index(b'endstream', start)
stream = p[start:end]
out = zlib.decompress(stream)
pathlib.Path('extracted.wav').write_bytes(out)
```

Output:

- `extracted.wav`

## 4. Re-evaluate the WAV Structure

At first glance, the WAV header makes the file look like:

- stereo,
- 16-bit,
- 48000 Hz.

But when I parsed the payload as `int16`, the channels looked suspicious:

- the left side varied heavily,
- the right side was mostly `0` or `-1`.

Reading the same raw payload as `int32` produced much more coherent data:

```python
np.frombuffer(raw, dtype='<i4')
```

This led to an important conclusion:

- the header was arranged to look like `stereo 16-bit`,
- but the payload made much more sense as **mono 32-bit PCM**.

That detail matters because the rest of the SSTV decoding depends on interpreting the signal correctly.

## 5. Identify the Signal as SSTV

Next, I plotted a spectrogram for `extracted.wav`:

```python
ax.specgram(sig, NFFT=4096, Fs=48000, noverlap=3072, cmap='magma')
```

Artifacts produced:

- `spectrogram_full.png`
- `spectrogram_full2.png`

The pattern showed:

- evenly repeating vertical structures,
- a modulated signal rather than music or speech,
- a very strong SSTV signature.

I then measured the spacing between sync pulses and found that they were roughly `44993` samples apart.

## 6. Why the Mode Is `PD290`

Comparing the measured pulse spacing against the SSTV `PD` family, `PD290` matched best.

Reasons:

- assuming `PD290`, the inferred sample rate is about `48003.8 Hz`, which is nearly identical to `48000 Hz`,
- the signal contained around `308` pulses,
- `PD290` produces an `800 x 616` image,
- the scanline structure also fits the expected `PD290` layout.

So the most consistent conclusion is:

- the embedded signal is **SSTV mode `PD290`**.

## 7. Cross-check with Reference Implementations

To avoid hand-decoding the format incorrectly, I compared my findings with existing SSTV implementations:

- `_tmp_robot36`
- `_tmp_PicoSSTV`
- `_tmp_sstv_decoder`

From `robot36`, the important details for `PD290` are:

- VIS code: `94`
- image size: `800 x 616`
- `channelSeconds = 0.2288`

Each scanline follows this structure:

1. `sync pulse`
2. `sync porch`
3. `Y even`
4. `V average`
5. `U average`
6. `Y odd`

That confirmed the exact sampling layout needed for decoding.

## 8. Write the Decoder

I wrote a Python decoder that follows the same general logic as `robot36`:

1. read `mono 32-bit` samples from `extracted.wav`,
2. mix the signal down around `1900 Hz`,
3. apply a low-pass filter,
4. FM-demodulate,
5. detect `1200 Hz` sync pulses,
6. segment the stream into scanlines,
7. sample the `Y / V / U / Y` sections,
8. convert `YUV -> RGB`.

Key demod step:

```python
freq[1:] = (fs / (bw * np.pi)) * np.angle(base[1:] * np.conj(base[:-1]))
```

Intermediate outputs:

- `pd290_decoded.png`
- `pd290_decoded_ema.png`

Best final image:

- `pd290_mono32.png`

## 9. What `pd290_mono32.png` Shows

The recovered image contains:

- a bright background,
- a dark spiral-like structure,
- a shape strongly resembling a galaxy or whirlpool.

This matches the text hint from the PDF exactly:

```text
something similar to the galaxy
```

At this stage, both earlier hints are confirmed:

- `"Papa Delta"` points to SSTV `PD`,
- `"galaxy"` describes the recovered image.

## 10. Separate RGB and Bit Planes

Once `pd290_mono32.png` was available, I began extracting useful derived views.

### RGB Channels

```python
img = np.array(Image.open('pd290_mono32.png').convert('RGB'))
Image.fromarray(img[:, :, 0]).save('chan_R.png')
Image.fromarray(img[:, :, 1]).save('chan_G.png')
Image.fromarray(img[:, :, 2]).save('chan_B.png')
```

Outputs:

- `chan_R.png`
- `chan_G.png`
- `chan_B.png`

### Bit Planes

```python
for c, name in enumerate('RGB'):
    for b in range(8):
        plane = ((img[:, :, c] >> b) & 1) * 255
```

Outputs:

- `bit_R0.png` to `bit_R7.png`
- `bit_G0.png` to `bit_G7.png`
- `bit_B0.png` to `bit_B7.png`

### Channel Difference Images

I also created:

- `RminusG.png`
- `BminusG.png`
- `RminusB.png`
- `GminusRB.png`
- `sum.png`

These are useful for highlighting details that are not distributed evenly across the color channels.

## 11. Convert Approximately from RGB to `YUV`

From `pd290_mono32.png`, I computed approximate `Y`, `U`, and `V` channels:

```python
Y = 0.299 * R + 0.587 * G + 0.114 * B
U = -0.14713 * R - 0.28886 * G + 0.436 * B
V = 0.615 * R - 0.51499 * G - 0.10001 * B
```

Outputs:

- `Y.png`
- `U.png`
- `V.png`

Of these, `Y.png` is the most important starting point for further analysis.

## 12. Why `Y.png` Matters

`Y.png` is the approximate luminance channel of the recovered image. It is especially useful because:

- if hidden content is primarily stored in brightness, this view preserves it best,
- it reduces color-related distractions,
- it is a natural input for thresholding, morphology, OCR, polar unwrap, FFT, or contrast enhancement.

That is why this write-up intentionally stops at `Y.png`: it is the cleanest handoff point for the next stage of the challenge.

## Important Files in Order

1. `report.pdf`
2. `extracted.wav`
3. `spectrogram_full.png`
4. `spectrogram_full2.png`
5. `pd290_decoded.png`
6. `pd290_decoded_ema.png`
7. `pd290_mono32.png`
8. `chan_R.png`
9. `chan_G.png`
10. `chan_B.png`
11. `RminusG.png`
12. `BminusG.png`
13. `RminusB.png`
14. `GminusRB.png`
15. `sum.png`
16. `Y.png`
17. `U.png`
18. `V.png`

## Conclusion

The full logic of the solve is:

1. notice that `report.pdf` contains a large embedded object,
2. confirm that the object is audio,
3. extract it as `extracted.wav`,
4. realize the payload should be interpreted as `mono 32-bit`,
5. use the spectrogram and scanline spacing to identify SSTV `PD290`,
6. cross-check the mode layout with `robot36`,
7. write a decoder and recover `pd290_mono32.png`,
8. derive color and luminance views,
9. stop at `Y.png` as the best artifact for continued analysis.
