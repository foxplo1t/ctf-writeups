# JYU — packet analysis (50)

**Flag:** `cruXipher{v1m_netw0rk_c4ptur3}`

> just a pcap file patented in 1936

Attachment: `secret.pcap` (26 KB)

## TL;DR

The capture is a plaintext terminal session in which someone edits `flag.txt` in vim. Pull the client keystrokes, replay them through a real vim to get the saved buffer, then undo the Dvorak layout (patented 1936).

## Recon

The pcap holds a single TCP conversation on loopback:

| Direction | Bytes | Content |
|---|---|---|
| `127.0.0.1:9999 → :56243` | 3 553 | Terminal output: vim redraws, `-- INSERT --`, echoed characters |
| `127.0.0.1:56243 → :9999` | 805 | Terminal input: capability replies, then the keystrokes |

Two details matter:

- The server output ends with `"flag.txt" [New] 1L, 31B written`, so the saved file is exactly **31 bytes** — a checksum for the reconstruction.
- **1936** is the year the Dvorak Simplified Keyboard was patented (US 2,040,248). The text was typed in Dvorak positions.

## Step 1 — Extract the keystrokes

Concatenate every payload sent from port 56243. The first ~700 bytes are the terminal answering vim's capability queries (`ESC P1+r…`, cursor-position reports). The real input starts at the first `i`:

```python
from scapy.all import rdpcap, TCP, Raw
client = b"".join(bytes(p[Raw].load) for p in rdpcap("secret.pcap")
                  if TCP in p and Raw in p and p[TCP].sport == 56243)
keys = client[client.index(b"ijpgQcld"):]
open("keys.bin", "wb").write(keys)
```

Decoded, the 57 bytes are:

| Bytes | vim meaning |
|---|---|
| `i` | enter insert mode |
| `jpgQcld.p?k1m{b.y,prt` | typed text |
| `ESC b cw` | back one word, change word |
| `0pt{jmfoy.p` | replacement text |
| `DEL × 7` | seven backspaces |
| `j4lygp3+` | more text |
| `ESC :wq\r` | save and quit |

## Step 2 — Replay through vim

`cw` and the backspaces make hand-simulation error-prone, so let vim do it. `-s` feeds a file as typed keys:

```bash
rm -f flag.txt
vim -u NONE -N -i NONE -n -s keys.bin flag.txt </dev/null >/dev/null 2>&1
wc -c flag.txt   # 31 ← matches the capture
cat flag.txt     # jpgQcld.p?k1m{b.y,0pt{j4lygp3+
```

## Step 3 — Undo the Dvorak layout

The vim commands work because they're literal bytes; only the *content* was typed on Dvorak keys. Map each character back to the QWERTY character at the same physical position (shifted symbols included):

```python
q_un = "`1234567890-=qwertyuiop[]\\asdfghjkl;'zxcvbnm,./"
d_un = "`1234567890[]',.pyfgcrl/=\\aoeuidhtns-;qjkxbmwvz"
q_sh = '~!@#$%^&*()_+QWERTYUIOP{}|ASDFGHJKL:"ZXCVBNM<>?'
d_sh = '~!@#$%^&*(){}"<>PYFGCRL?+|AOEUIDHTNS_:QJKXBMWVZ'
d2q = dict(zip(d_un + d_sh, q_un + q_sh))
print("".join(d2q.get(c, c) for c in open("flag.txt").read().strip()))
# cruXipher{v1m_netw0rk_c4ptur3}
```

The opposite direction (QWERTY → Dvorak) gives `hli"jnevlZt1m?xvfw0ly?h4nfil3}` — clearly wrong. The clean `{`, `_` and `}` confirm Dvorak → QWERTY.

## Dead ends

- Reading the raw hex dump: keystrokes are interleaved with TCP/IP headers and terminal escape codes — too noisy.
- Hand-simulating the vim edits (`b`, `cw`, backspaces): error-prone, so I replayed in real vim instead.
- Wrong mapping direction (QWERTY → Dvorak).

## Takeaways

- Use leaked metadata (`31B written`) as a correctness check.
- Replay editor input in the real editor instead of simulating motions like `b` and `cw`.
- Keyboard-layout hints usually apply only to the typed text, not the control keys.
