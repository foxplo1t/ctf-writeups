# Paper CPU — rev / vm / bytecode (100)

**Flag:** `cruXipher{4456e680}` *(unconfirmed — `cruXipher{4480cc90}` was rejected; fallback `cruXipher{44aa9280}`)*

> A fictional 8-bit CPU with three registers (R0, R1, R2) and 256 bytes of RAM. Programs are raw bytecode.

Attachments: `README.md`, `challenge.bin` (121 B), and four examples: `kront.bin`, `vexx.bin`, `blip.bin`, `snok.bin`.

## TL;DR

The instruction set isn't documented. Infer it from the example programs and their I/O pairs, write an emulator, and run the challenge. The catch is opcode `3`: it appears only in the challenge, and its operand is a **register**, not an immediate.

## The rules

- Start: R0 = input byte, R1 = R2 = 0, RAM all zeros.
- Examples: "output" is R0 at halt.
- Challenge: run `challenge.bin` with input `2a` and read `RAM[0x40–0x43]` as 8 lowercase hex digits.

| Program | In | Out |
|---|---|---|
| kront.bin | 2a | 4a |
| vexx.bin | 00 | 00 |
| vexx.bin | 01 | c3 |
| blip.bin | 2a | 7e |
| snok.bin | ff | 55 |

## Step 1 — Infer the ISA

Every program reads cleanly as 2-byte instructions `[op<<4 | reg] [operand]`, with `70` as halt.

- **snok** `50 0a | 00 aa | 50 0b | 60 0a | 20 aa | 70`: `20 aa` turns `ff` into `55`. `ff ⊕ aa = 55`, so opcode 2 is **XOR immediate** (kront rules out subtraction: `2a ⊕ 33 = 19` matches, `2a − 33` doesn't).
- **blip**: `11 01` takes R1 = 2a to 54, then `10 01` gives R0 = 2a + 54 = 7e. Opcode 1 is **Rr += Rs** (operand names a register).
- **kront**: adds 07, XORs 33, adds 31 → 4a. Confirms opcodes 1 and 2.
- **vexx** `01 c3 | 51 00 | 40 02 | 60 00 | 70`: with input 00 the `60 00` load is skipped, so opcode 4 is **jump if Rr == 0**, relative to the next instruction.

| Opcode | Meaning |
|---|---|
| `0r imm` | Rr = imm |
| `1r s` | Rr = Rr + Rs |
| `2r imm` | Rr = Rr ⊕ imm |
| `3r s` | not used in any example — see Step 3 |
| `4r off` | if Rr == 0: PC = PC + 2 + (signed) off |
| `5r a` | RAM[a] = Rr |
| `6r a` | Rr = RAM[a] |
| `70` | halt |

## Step 2 — Emulator

```python
def run(prog, inp, op3):
    R, M, pc = [inp, 0, 0], [0]*256, 0
    while pc < len(prog) - 1:
        op, x = prog[pc], prog[pc+1]; hi, r = op >> 4, op & 15; npc = pc + 2
        if   hi == 0: R[r] = x
        elif hi == 1: R[r] = (R[r] + R[x]) & 255
        elif hi == 2: R[r] ^= x
        elif hi == 3: R[r] = op3(R[r], R[x])        # register operand
        elif hi == 4 and R[r] == 0: npc += x - 256 if x > 127 else x
        elif hi == 5: M[x] = R[r]
        elif hi == 6: R[r] = M[x]
        elif hi == 7: break
        pc = npc
    return R, M
```

It reproduces all 5 example I/O pairs.

## Step 3 — The rejected answer and the fix

My first run treated opcode 3's operand as an immediate. Every candidate meaning (set, add, sub, and, or) gave `4480cc90`, which the board rejected.

The fix: read opcode 3 like opcode 1, with a **register** operand. Then `31 00` = "R1 ← R0" copies the input at the start of the program — exactly how you'd expect it to begin.

Re-running every combination (opcode-3 meaning × jump-offset unit × jump-condition model) — all pass the examples; the runs that end cleanly on the `70` halt give:

| Opcode 3 (register) | RAM[0x40..0x43] |
|---|---|
| mov, xor, or, add, rsub | **4456e680** |
| sub | 44aa9280 |
| and | 4480cc90 (rejected) |

Five meanings agree because at each opcode 3 the destination register is still 0, so copy/add/xor/or all produce the same value.

## Disassembly (opcode 3 = mov)

```
00 50 40  RAM[40]=R0        ; save input 2a
02 31 00  R1=R0
04 51 41  RAM[41]=R1
06 32 00  R2=R0
08 52 42  RAM[42]=R2
0a 61 40  R1=RAM[40]
0c 21 3c  R1^=3c
0e 62 41  R2=RAM[41]
10 12 01  R2+=R1
12 52 43  RAM[43]=R2
14 31 00  R1=R0
16 30 01  R0=R1             ; loop counter
18 02 6d  R2=6d
1a 11 02  R1+=R2   ← loop 1
1c 21 2c  R1^=2c
1e 12 01  R2+=R1
20 22 17  R2^=17
22 51 21  RAM[21]=R1
24 02 ff  R2=ff
26 10 02  R0+=R2            ; R0--
28 61 21  R1=RAM[21]
2a 32 01  R2=R1
2c 40 02  jz R0 → 30
2e 40 ea  jz R0 → 1a
30 60 43  R0=RAM[43]
…  loop 2 at 42–54, same shape, then mixing into RAM[40..43] …
78 70     halt
```

Both loops are a `jz` exit followed by a backward `jz`. Under the jump rule vexx establishes, the backward jump can't fire right after the exit check falls through, so each body runs once. Either way the final `RAM[0x40..0x43]` is the same, because the loop bodies only write scratch RAM (0x21, 0x22).

## Dead ends

- Opcode 3 with an immediate operand (set/add/sub/and/or/no-op): always `4480cc90` — rejected.
- Bit 7 of the jump operand flipping the condition: loops ran thousands of steps, still `4480cc90`.
- Jump offsets counted in instructions: passed the examples but `challenge.bin` never reached the halt.

## Takeaways

- Pin each opcode to the specific example that proves it, and note which opcodes no example covers.
- When an unknown opcode is ambiguous, enumerate candidates and look for the answer that's stable across them.
- Look for operand-type symmetry between opcodes — register vs immediate was the real bug here.
