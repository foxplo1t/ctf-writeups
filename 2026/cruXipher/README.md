# cruXipher 2026

- **Dates:** October 23–25, 2026
- **Organiser:** CRUx, BITS Pilani Hyderabad ([event site](https://cruxipher.crux-bphc.com/) · [CTFtime](https://ctftime.org/ctf/1004/))
- **Team:** IHU_Serres
- **Result:** 8 challenges solved, 650 points (plus one 460-pt challenge reversed but unsolved)

## Write-ups (in solve order)

| Challenge | Category | Points | Key technique | Write-up |
|-----------|----------|--------|---------------|----------|
| JYU | packet analysis | 50 | TCP keystroke extraction + vim replay + Dvorak remap | [forensics/jyu.md](forensics/jyu.md) |
| t2f-program | rev | 50 | Over-subscribed deflate Huffman table | [rev/t2f-program.md](rev/t2f-program.md) |
| Paper CPU | rev / vm | 100 | ISA inference from I/O pairs + emulator | [rev/paper-cpu.md](rev/paper-cpu.md) |
| rice-writing | osint | 50 | Pixel-area measurement, sort, mod 256 | [osint/rice-writing.md](osint/rice-writing.md) |
| PDF Prince | forensics / pdf | 100 | Incremental updates, hidden OCG key, XOR, glyph rendering | [forensics/pdf-prince.md](forensics/pdf-prince.md) |
| house-music | forensics / audio | 100 | Stereo side channel + CRUX-keyed pigpen spectrogram | [forensics/house-music.md](forensics/house-music.md) |
| Wait For It | forensics / timing | 50 | Morse in IRC inter-message timing | [forensics/wait-for-it.md](forensics/wait-for-it.md) |
| what-did-you-compress | misc / compression | 100 | Custom LZW, xorshift-shuffled table | [misc/what-did-you-compress.md](misc/what-did-you-compress.md) |
| boxy proxy | web / misc | 460 | SSRF proxy + RSA/x25519/AES-GCM handshake (**unsolved**) | [web/boxy-proxy.md](web/boxy-proxy.md) |

**Toolchain:** Python 3 (scapy, numpy, scipy, Pillow, fontTools, cairosvg, PyMuPDF), vim, minimodem and multimon-ng (dead ends only).
