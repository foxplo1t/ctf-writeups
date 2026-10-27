# PDF Prince — forensics / pdf (100)

**Flag:** `cruXipher{xr3f_r3m3mb3rs_wh4t_y0u_d3l3t3d}` *(the PDF writes the prefix as `cruX{`)*

> An internal memo from a sanitisation review.

Attachment: `chall.pdf` (9 240 bytes)

## TL;DR

The "sanitiser" only appended **incremental updates**, so the original page text still sits in revision 1. A hidden optional-content layer holds an XOR key. Decrypting the old text gives glyph IDs, and rendering those glyphs from the embedded font shows the flag.

## Recon — three revisions

```
size 9240   %%EOF × 3   xref × 3   startxref × 3
Info: /Producer (SanitizerPro 2.1 - history purged)
```

Each trailer's `/Prev` points to the one before, so the file is a stack of three versions:

| Revision | Adds / changes | What it holds |
|---|---|---|
| 1 (original) | objects 1–9 | Page whose content stream (obj 4) is 74 bytes of hex text in font *GhostMono* |
| 2 | new obj 1, 3, 4; obj 11, 12 | New page text, a second font, and an optional-content group (OCG) `keyring` switched **OFF** |
| 3 | frees obj 10; adds Info (obj 13) | "history purged" |

I dumped every object (including superseded and freed ones) with a regex over `n 0 obj … endobj` and zlib-inflated each stream.

## Step 1 — Clear the decoys

1. Freed object 10 is a raw stream: `flag{nice_grep_but_not_the_flag}` — bait for `strings | grep flag`.
2. Revision 2's visible text uses an Identity-H font with a ToUnicode CMap padded with zero-width spaces (U+200B). Stripped, it's ROT13:
   - `VAGREANY ZRZB 0k2S // ERI 3 // FNAVGVFRQ // QB ABG SBEJNEQ` → *INTERNAL MEMO 0x2F // REV 3 // SANITISED // DO NOT FORWARD*
   - `Gur synt jnf fhccbfrq gb or urer.` → *The flag was supposed to be here.*
   - `V jbaqre jurer vg jrag.` → *I wonder where it went.*
3. The fourth line copies out as `ctf{copy_paste_is_a_trap}`, but rendering the page shows only dashes there. Copy-paste reads the ToUnicode map; the screen shows the glyphs — they deliberately disagree.

## Step 2 — Find the key

Revision 2's content stream ends with a marked-content block bound to the hidden OCG:

```
/OC /MC0 BDC
BT /F2 10 Tf 1 0 0 1 40 140 Tm 14 TL
(KEYRING / layer 7) Tj T*
(key = L7-b5acb9c6) Tj T*
ET
EMC
```

The catalog lists OCG 12 (`keyring`) under `/OFF`, so viewers never draw it.

## Step 3 — Decrypt the original text

Revision 1's page draws one hex string. Read as 2-byte glyph IDs, almost every value is above 196 (the font's glyph count). The first bytes `4C 32 2D` look a lot like `L7-` — so XOR with the repeating key:

```python
b = bytes.fromhex(hexstr); key = b"L7-b5acb9c6"
x = bytes(c ^ key[i % len(key)] for i, c in enumerate(b))
codes = [int.from_bytes(x[i:i+2], "big") for i in range(0, len(x), 2)]
# [5, 126, 119, 187, 196, 87, 126, 11, …, 66, 128] ← 37 codes, all 0–196
```

Every high byte becomes `00` and all 37 IDs are valid — strong confirmation.

## Step 4 — Read the glyphs, not the text map

The embedded font's cmap covers only 21 characters and most of its 197 glyphs are unnamed, so draw the IDs straight from the outlines:

```python
from fontTools.ttLib import TTFont
from fontTools.pens.svgPathPen import SVGPathPen
f = TTFont("ghost.ttf"); gs = f.getGlyphSet(); order = f.getGlyphOrder()
# one <path> per code, translated 1200 units right each time → flag.svg
# cairosvg.svg2png(url="flag.svg", write_to="flag.png")
```

The rendered strip reads `cruX{xr3f_r3m3mb3rs_wh4t_y0u_d3l3t3d}` — 37 glyphs, one per code.

## Dead ends

- Grepping for `flag` → only the decoy in freed object 10.
- Extracting text via ToUnicode → ROT13 memo + `ctf{copy_paste_is_a_trap}`.
- Reading revision-1 codes directly as glyph IDs → all out of range (encrypted).
- Mapping glyph IDs through the font's cmap / matching outlines to named glyphs → blanks.
- Rendering the page → a row of dashes on the flag line.

## Takeaways

- Count `%%EOF`s: incremental updates never remove old content.
- ToUnicode and the drawn glyphs can disagree — render to see what's really shown.
- Hidden OCG layers (`/OFF`) are a classic place to stash keys.
- A key's first bytes lining up with the ciphertext is a quick XOR tell.
