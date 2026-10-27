# t2f-program — rev (50)

**Flag:** `cruXipher{sighunzips}`

> My 6-year-old son asked me for $5 to buy a Lego set yesterday… Download my Toddler-to-Founder (T2F) Masterclass Trial. Find it out yourself.

Attachment: `t2f-masterclass.zip` (121 646 bytes)

## TL;DR

The zip's deflate stream has a tampered dynamic Huffman table that strict inflaters reject. Decode it leniently, find the rare characters hidden in 128 KB of filler, and undo the symbol shift the tamper caused.

## Recon

```
$ unzip -l t2f.zip
   131072  2026-10-22 12:00   flag.txt
$ unzip t2f.zip
  error: invalid compressed data to inflate flag.txt
```

`zipinfo -v` shows consistent headers: method 8 (deflate), compressed 121 532 bytes, uncompressed 131 072, CRC `cce1c38d`, data starting at byte 38. Python's raw inflate fails immediately:

```python
import zlib
d = open("t2f.zip", "rb").read()[38:38+121532]
zlib.decompressobj(-15).decompress(d)
# zlib.error: Error -3 … invalid literal/lengths set
```

The failure is in the very first block header, so the container is fine and the **Huffman table** is the puzzle.

## Step 1 — Parse the block header by hand

A small LSB-first bit reader decodes the dynamic-block header (RFC 1951 §3.2.7):

| Field | Value |
|---|---|
| BFINAL / BTYPE | 1 / 2 (dynamic Huffman) |
| HLIT / HDIST / HCLEN | 257 / 2 / 19 |
| Code-length code Kraft sum | 1.0 (valid) |
| Literal/length Kraft sum | **1.00390625** |
| Distance codes | 2 (distances 1 and 2 only) |

A valid prefix code has a Kraft sum of exactly 1. This one is over-subscribed by exactly 2⁻⁸ — the same as two extra 9-bit codes. That's the tamper.

## Step 2 — Read the length table

The literal alphabet is `\n` plus printable ASCII, with lengths rising neatly by symbol value (B–E = 4, F–M = 5, N–[ = 6, \–o = 7 …). A handful of letters break the pattern at length 9: `X c d e h i p q r u {` — the letters of `cruXipher{` plus two extras, `d` and `q`.

## Step 3 — Rule out a single-length fix

I tried all 24 one-symbol changes that bring Kraft to exactly 1, decoding each and checking the CRC. None matched; all stopped early at 52–63 KB. So the tamper isn't one wrong length.

## Step 4 — Decode leniently

Build the canonical code anyway. The two overflowing 9-bit codes (`{` and EOB) become unreachable. Decode until the declared 131 072 bytes:

```python
# build(): canonical Huffman → {(length, code): symbol}, no completeness check
out = bytearray()
while len(out) < 131072:
    s = dec(reader, litlen_table)
    if s < 256:    out.append(s)
    elif s == 256: break
    else:          # length/distance pair (distances are only 1 or 2)
        L = LEN_BASE[s-257] + reader.bits(LEN_EXTRA[s-257])
        D = DIST_BASE[dec(reader, dist_table)]
        for _ in range(L): out.append(out[-D])
```

This uses 972 240 of 972 256 bits with no desync — strong evidence it's the intended decode; the CRC differs only because the table itself was altered.

## Step 5 — Find the needle

The output is 1 652 lines of random-looking printable characters. Frequency count: almost everything appears ~1 400 times, but `c X d r p q i e h` appear only 1–3 times, and `u` and `{` never. All the rare ones sit together at offset 65 525:

```
…A+|NgFWH3 cpqXhiedprshgeqnzhis} ^Nf9%?t#Pg…
```

## Step 6 — Undo the symbol shift

Canonical codes are assigned in symbol order within each length. The tamper inserted `d` and `q` into the 9-bit group, so every real 9-bit symbol after them decodes one or two places too early:

| Decoded | c | p | q | X | h | i | e | d | p | r |
|---|---|---|---|---|---|---|---|---|---|---|
| **Real** | c | r | u | X | i | p | h | e | r | { |

That recovers `cruXipher{`. Applying the same map to `shgeqnzhis` (9-bit `h→i`, `e→h`, `q→u`, `i→p`; 8-bit `s g n z` unchanged) gives `sighunzips`, followed by `}`.

## Dead ends

- `unzip`, 7-Zip (Deflate64 theory), zlib raw inflate — all reject the stream.
- Fixing one code length: none of the 24 candidates pass the CRC.
- Grepping the lenient output for `crux` / `flag` / `{` — zero hits, because the flag letters decode shifted.

## Takeaways

- "Invalid literal/lengths set" means a broken code — check the Kraft sum before blaming the data.
- A lenient decode that uses nearly every bit is strong evidence it's correct.
- With uniform filler, frequency analysis finds the payload instantly.
