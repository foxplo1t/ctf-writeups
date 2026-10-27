# what-did-you-compress — misc / compression (100)

**Flag:** `cruXipher{m3ss3d_up_compXsion}`

> Someone wrote their own compressor, then forgot how it works. Recover the data they compressed. Flag format `cruXipher{...}`.

Attachments: `sample.txt`, `sample.lzw` (known-plaintext pair), `flag.lzw` (27 bytes)

## TL;DR

A hand-rolled **LZW** variant: 10-bit starting codes packed LSB-first, reset/end control codes, and a 256-entry symbol table shuffled Fisher–Yates with **xorshift32 (13,17,5)** seeded by the "index key" `0x0745B6`. Decoding `flag.lzw` with that seed gives `m3ss3d_up_compXsion`.

## Recon

The story is the spec: each tape gets its own symbol table, reshuffled "Fisher–Yates style, walking down from the last slot," with randomness from "a small xorshift-family generator" seeded by the index key printed in the log — `0745B6` in hex (also hidden as "07:45" and "shelf B-6"). It writes a leading reset code, data codes, an end code, packs bits from the low end of each byte, starts with wider codes, and reserves the first slots for control codes.

## Step 1 — Recover the parameters from the known pair

Using `sample.txt` ↔ `sample.lzw`, confirm the shuffle: xorshift32 (13,17,5), a 256-entry table, and `table[code] = byte`. All ten tested byte→code pairs match.

## Step 2 — Write the decoder and round-trip the sample

```python
M32 = 0xffffffff

def perm(seed):
    s = seed
    def r():
        nonlocal s
        s ^= (s << 13) & M32; s ^= s >> 17; s ^= (s << 5) & M32
        return s
    p = list(range(256))
    for i in range(255, 0, -1):
        j = r() % (i + 1); p[i], p[j] = p[j], p[i]
    return p

def decode(data, seed, W0=10):
    p = perm(seed); n = int.from_bytes(data, 'little'); L = len(data) * 8; pos = 0
    d = {i: bytes([p[i]]) for i in range(256)}; nxt, w = 258, W0; out = b''; prev = None
    while pos + w <= L:
        c = (n >> pos) & ((1 << w) - 1); pos += w
        if c == 256:                       # reset
            d = {i: bytes([p[i]]) for i in range(256)}; nxt, w = 258, W0; prev = None; continue
        if c == 257:                       # end
            return out
        e = d[c] if c in d else prev + prev[:1]
        out += e
        if prev is not None:
            d[nxt] = prev + e[:1]; nxt += 1
        prev = e
        if nxt >= (1 << w) and w < 16: w += 1
    return out
```

`decode(open('sample.lzw','rb').read(), 0x0745B6)` reproduces `sample.txt` byte-for-byte.

## Step 3 — Decode flag.lzw

The same seed works (brute force confirms it's the only clean one). Every code resolves to a single character: `m 3 s s 3 d _ u p _ c o m p X s i o n` → `cruXipher{m3ss3d_up_compXsion}`.

## Dead ends

- Plain LZW with a standard table — codes didn't line up until accounting for reset = 256, end = 257, entries from 258.
- Several xorshift variants (xs16/xs32/xs64, other triples) before (13,17,5) matched.
- Suspected `flag.lzw` used a different key — brute-forced all 2²⁴ seeds; only `0x0745B6` yields clean text ending on the end code.
- The `X` in `compXsion` looked like a decoder bug — re-verified with an independent LZW decoder; it's genuinely in the data.

## Takeaways

- A known-plaintext pair plus a prose spec lets you rebuild a custom codec exactly — always verify by round-tripping the sample.
- Parameters hide in flavour text (`0745B6` = "07:45" + "shelf B-6").
- Verify a surprising output with an independent implementation before calling it a bug.
