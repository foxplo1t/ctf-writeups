# house-music — forensics / audio steg (100)

**Flag:** `cruXipher{spectro_phase_mason}` *(formatting unconfirmed; alternatives `cruXipher{spectrophasemason}`, `cruXipher{SPECTROPHASEMASON}`)*

> tried making some house music, but the foundation was kinda off. lesson learned: CRUX first.

Attachment: `transmission.wav` (1 410 380 bytes)

## TL;DR

Hidden tones are mixed in **opposite phase** in the two channels, so they cancel in mono and drown in noise in each channel. **L − R** removes the noise. In a spectrogram, the tones draw **pigpen-cipher** glyphs. Decoding them with a **CRUX-keyed** alphabet gives SPECTRO PHASE MASON.

## Step 1 — Triage

| Check | Result |
|---|---|
| Format | PCM stereo, 16-bit, 44 100 Hz, 7.995 s |
| RIFF chunks | `fmt ` + `data` only; data ends exactly at EOF |
| Metadata / strings | nothing |
| LSB planes (L, R, interleaved, L−R) | random |
| Spectrogram of L or R | uniform white noise 0–22 kHz |

## Step 2 — Find the side channel

The channels are 99.5% correlated — almost the same signal with a small difference. Split into mid and side:

```python
import numpy as np, wave
w = wave.open("transmission.wav")
d = np.frombuffer(w.readframes(w.getnframes()), np.int16).reshape(-1, 2).astype(float)
mid, side = d[:,0] + d[:,1], d[:,0] - d[:,1]
```

Mid is pure noise. Side holds **17 clean tone bursts** between 2.9 and 7.3 kHz. The construction was:

```
L = N + T,  R = N − T   ⇒   L + R = 2N,   L − R = 2T
```

That's "the foundation was kinda off": the hidden part is exactly where the channels differ.

## Step 3 — Characterise the tones

- Tones sit on a grid of 2907 + n × 151 Hz, in beating pairs (n, n+1).
- Pairs cluster at five slots: n = 0, 7, 14, 21, 28 ≈ 2982 / 4040 / 5096 / 6154 / 7210 Hz.
- 3 short rising-sweep bursts and 14 long ones, separated by gaps of ~70, 240 and 340 ms.

## Step 4 — Read the spectrogram as a picture

Render the side channel in high contrast between 2.5 and 7.7 kHz:

```python
from scipy.signal import spectrogram
f, t, S = spectrogram(side, fs=44100, nperseg=1024, noverlap=1000)
# 10*log10(S), gray_r colormap, vmin at the 60th percentile, axes off
```

The bursts are glyphs — `<`, `V`, `⊓`, `⊔`, `□`, some with dots. That's the **pigpen (Freemason's) cipher**:

| Stroke | Audio feature |
|---|---|
| Top bar | steady tone at slot s4 (~7.2 kHz) |
| Bottom bar | steady tone at slot s0 (~3.0 kHz) |
| Left / right vertical | hard start / stop of the burst (broadband click) |
| Dot | short blob at slot s2 (~5.1 kHz) mid-burst |
| Diagonals `<` `V` `Λ` | frequency sweeps |

## Step 5 — Measure every glyph

To avoid misreading, measure each stroke per burst (energy in top/bottom bands, broadband energy at the edges, dot-frequency energy in the middle):

```
<  V  ⊓  ⌟  Λ  ⊔  ⌜·  |  V  ⊔·  □  <  ⊓  |  ⌝·  □  <  ⌜·  ⊓·
```

The `|` marks are the two longest gaps (~340 ms).

## Step 6 — Decode with a CRUX-keyed alphabet

Standard pigpen gives `USHAVBRSKEUHPEURQ` — gibberish. "CRUX first" means a keyed alphabet: write CRUX first, then the remaining letters:

```
C R U X A B D E F | G H I J K L M N O | P Q S T | V W Y Z
  grid (plain)    |   grid (dotted)   | X plain | X dotted
```

| Burst | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Glyph | < | V | ⊓ | ⌟ | Λ | ⊔ | ⌜· | V | ⊔· | □ | < | ⊓ | ⌝· | □ | < | ⌜· | ⊓· |
| Standard | U | S | H | A | V | B | R | S | K | E | U | H | P | E | U | R | Q |
| **CRUX-keyed** | **S** | **P** | **E** | **C** | **T** | **R** | **O** | **P** | **H** | **A** | **S** | **E** | **M** | **A** | **S** | **O** | **N** |

The word gaps split it into **SPECTRO · PHASE · MASON** — which also summarises the solve: spectrogram, phase cancellation, mason cipher.

## Dead ends

LSB steganography, vectorscope art, binary FSK, tone-index → ASCII, known-plaintext substitution, chord bitmasks, minimodem / multimon-ng (Morse, DTMF, AFSK, POCSAG…), Morse from burst lengths, piano-roll bitmaps, pitch-contour handwriting, and standard (unkeyed) pigpen.

## Takeaways

- If the two channels are almost but not quite identical, check **L − R**.
- A spectrogram is an image — look at its shape before decoding frequencies as numbers.
- Hints about a "foundation" or what comes "first" often mean a keyed alphabet.
