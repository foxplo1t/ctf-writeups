# rice-writing — osint (50)

**Flag:** `cruXipher{riCegOesWiThaNyThinG}`

> …what is written on a rice grain is often not just dependent on how big it is, but rather what remains of it. You could say the size of the grain is just how big it appears or how much space it occupies… Here is an image which depicts an interesting method of rice writing (135.3MB)

**Hint:** "Even an unordered list of integers has an inherent order if you sort it."

## TL;DR

Each grain's **pixel area** encodes one character. Sort the grains by area, take each area **mod 256**, and read the result as ASCII.

## Step 1 — Get the image

The image is a public Google Drive file; the direct-download endpoint skips the virus-scan interstitial:

```bash
curl -sSL -o rice.png \
  "https://drive.usercontent.google.com/download?id=1B8tiPwLwxdfkI_UP6ducVBzUAqzzc7gW&export=download&confirm=t"
file rice.png   # PNG image data, 10000 x 10000, 8-bit/color RGB
```

It shows hundreds of white rice grains on dark wood — no writing on any grain and no PNG metadata. So the story's wording is the real clue:

- "how much space it occupies" → **area**
- "what remains of it" → **remainder**
- the hint → **sort**

## Step 2 — Segment and measure

Grains are bright, the background dark, and they don't touch. Threshold + connected-component labelling separates them:

```python
from PIL import Image; Image.MAX_IMAGE_PIXELS = None
import numpy as np
from scipy import ndimage

g = np.asarray(Image.open("rice.png").convert("L"))
lab, n = ndimage.label(g > 150)
areas = np.bincount(lab.ravel())[1:]
areas = np.sort(areas[areas > 500])   # 311 grains
```

Thresholds of 120 and 150 give identical areas for all 311 grains (edges are crisp), so the counts are exact. Sorted, they rise by about 256 per grain, from 41 026 to 120 435 — a base size plus a small offset.

## Step 3 — Read the remainders

```python
print("".join(chr(int(a) % 256) for a in areas))
```

```
BryanQuangLe{bOrNNovEmbErNineTYnINetYsIX}betterknownasRiceGumisanAmerican
videouploaderandonlinestreamer…feudswithothercruXipher{riCegOesWiThaNyThinG}
personalitiesSincecreatinghislinuxricingchannel…
```

It's a reworded Wikipedia blurb about RiceGum. It opens with a decoy, `BryanQuangLe{…}` (his real name and birthdate in flag shape). The real flag is in the middle.

## Dead ends

- Looking for micro-text on the grains: plain white grains, nothing written.
- PNG metadata: no text chunks or comments.
- Colour steganography: 24 689 unique colours looked like natural shading.

## Takeaways

- When a story keeps stressing *size* and *what remains*, measure size and take a remainder.
- Check a measurement is stable (two thresholds, same numbers) before decoding it.
- Search decoded text for the real flag format instead of taking the first `{…}`.
