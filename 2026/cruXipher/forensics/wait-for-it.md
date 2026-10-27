# Wait For It — packet analysis / timing (50)

**Flag:** `cruXipher{T1M1NG_1S_3V3RYTH1NG}`

> just a pcap file

Attachment: `capture.pcap` (855 KB)

## TL;DR

The pcap is an IRC chat (port 6667) spamming decoy flags. The real flag is a **Morse covert channel in the timing** of one channel's messages: the distance between delayed messages is the dot/dash, and the delay size (0.1 s vs 0.3 s) marks the letter boundary.

## Recon

8 816 packets. Port 6667 (IRC) dominates; DNS (226) and HTTP:80 (240) are cover traffic. Two clients log in: `listener` joins `#ctf`, `watcher` joins `#offtopic`. Each loops a 6-line script spamming a decoy flag. Both channel topics say *"keep your ears open"*; `#offtopic` adds *"the flag is in the timing."*

## Step 1 — Extract the timing marks

Reassemble the PRIVMSG timestamps per channel and treat gaps > 0.05 s as "marks". Gaps cluster at ~0.1 s and ~0.3 s; the distance (in messages) between consecutive marks clusters at ~11 and ~33 (ratio 1:3).

```python
from scapy.all import rdpcap, TCP, Raw
pkts = rdpcap('capture.pcap')
rows = []
for p in pkts:
    if p.haslayer(TCP) and p.haslayer(Raw) and {p[TCP].sport, p[TCP].dport} == {6667, 57474}:
        if b'PRIVMSG' in bytes(p[Raw].load):
            rows.append(float(p.time))
rows.sort()
d = [rows[i] - rows[i-1] for i in range(1, len(rows))]
marks = [(i, g) for i, g in enumerate(d) if g > 0.05]   # (msg index, gap value)
```

## Step 2 — Decode as Morse

The correct mapping:

- **distance to the next mark** = dot (~11) or dash (~33)
- **gap value** (0.1 vs 0.3) = letter boundary
- `..--.-` is Morse for `_`

`#offtopic` decodes to a "NICE TRY" taunt (confirming the method — and that it's a decoy). `#ctf` decodes, looping, to **T1M1NG_1S_3V3RYTH1NG**.

## Dead ends

- Submitted the in-chat decoys `cruXipher{n0t_th3_fl4g_lol}` and `cruXipher{d3c0y_str34m}` — both fake.
- Chased DNS queries and rotating HTTP `Host:` headers — cover traffic.
- HTTP GET inter-request gaps — noisy (0.2–9 s), not the channel.
- Marks as raw binary (5/7/8-bit, both orders/polarities) — near-misses, no flag.
- Morse with the roles swapped (gap = dot/dash, distance = separator) — pure garbage.

## Takeaways

- "keep your ears open" + "Wait For It" = timing covert channel; measure inter-message deltas first.
- When a channel carries two orthogonal signals (symbol vs separator), map them the right way round — swapping gives total garbage, not a near-miss.
- Plaintext flags inside a stego/forensics challenge are almost always decoys.
