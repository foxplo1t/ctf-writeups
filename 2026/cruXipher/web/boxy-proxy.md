# boxy proxy — web / misc (460) — unsolved

**Flag:** — *(not solved; protocol fully reversed, blocked on a CA-signed certificate)*

> Check out this cool new proxy provider. I hope they make their service cheaper soon.

Attachment: `ca_cert.pem` (the "Pre-established CA" public certificate — the only file provided)

## TL;DR

`POST /register-proxy {host, port}` makes the backend connect out to your node and run an authenticated, end-to-end-encrypted handshake (**RSA identity + x25519 ECDH + AES-GCM**) destined for an internal service, then deliver the flag. I reversed the whole protocol but got blocked: the server requires a certificate genuinely signed by its CA, and only the CA's public cert was provided — no private key, no signing endpoint, no source.

## Recon — infrastructure

| Port | Service | Notes |
|---|---|---|
| 443 | nginx → Express | Landing page ("Authstorm"), `/quote.html` (random-USD troll), `/register-proxy`. The 404 page leaks `/register-proxy`. Flaky (502/504). |
| 6001 | Express + helmet | Only route: `/stats` → `401 {"error":"Unauthorized"}` |
| 8034 | TCP name-echo | `Enter your name:` → `Hello, <name>!` — decoy |
| 8053 | custom proxy protocol | The handshake's target; firewalled from outside |

## The protocol (reversed)

`POST /register-proxy` with JSON `{"host": string, "port": number}` makes the backend connect to your node. It sends an optional `host:port\n` target line, then **4-byte big-endian length-prefixed** messages:

1. A Hello describing the scheme
2. Its RSA public key (fixed identity)
3. An ephemeral x25519 public key
4. An RSA signature over that key

It then expects our reply — a **CA-signed certificate**, our x25519 key, and our signature — after which both sides derive an AES-GCM key via x25519 ECDH and it sends the flag. Because ECDH is symmetric, any certificate the server accepts would let us compute the same key and decrypt.

## Current blocker

The server performs full, correct validation: the certificate must be genuinely signed by the Pre-established CA, and the x25519 signature must verify against that certificate's key. With only the CA's public cert we can't produce a valid cert + key pair, and the CA key isn't weak. Realistic unblockers: the CA private key, the server source, or a crypto flaw only visible in source.

## Tooling (ready to finish)

A [bore](https://github.com/ekzhang/bore) tunnel (`bore.pub:PORT → local :9000`) exposes a Python handler that answers the handshake and auto-attempts AES-GCM decryption across common KDF/nonce layouts. The register-to-trigger loop is confirmed working — the moment a valid cert + key appears, it plugs straight in.

## What I tried

- Self-signed X.509 cert → rejected (connection closed, 0 bytes).
- Raw RSA public-key PEM as the "certificate" → rejected.
- Reflecting the peer's own RSA key back as our cert → rejected.
- Forged cert with issuer CN `Pre-established CA` but our own key → rejected (issuer string isn't enough).
- Presenting the real CA cert itself → rejected (x25519 signature must bind to the cert key).
- Fermat + small-factor + e-checks on the CA and peer RSA keys → both strong.
- Directory brute force on 443 (63k words) and 6001 → only `/register-proxy` and `/stats`.
- Extra fields on `/register-proxy` (csr, pubkey, name, email, provider) → ignored.
- `/stats` auth bypass via `X-Forwarded-For`, `X-Real-IP`, `Authorization: Bearer`, `X-Api-Key` → still 401.
- Format-string and template-injection payloads on port 8034 → plain name-echo decoy.

## Takeaways

- Reversing a custom binary protocol from a live service is tractable: register a callback, capture the handshake, and read the Hello — it described its own scheme.
- A correctly implemented RSA + x25519 + AES-GCM handshake resists a proxy MITM by design; without the CA key or a bug, black-box cert forgery is a dead end.
- Know when to stop: after every cert bypass was rejected and the keys proved strong, the remaining paths all needed material we didn't have.

*If you solved this one, I'd love to read your write-up!*
