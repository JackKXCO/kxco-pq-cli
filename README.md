# kxco-pq-cli

[![npm](https://img.shields.io/npm/v/kxco-pq-cli?label=npm&color=b0964f)](https://www.npmjs.com/package/kxco-pq-cli)
[![Socket](https://socket.dev/api/badge/npm/package/kxco-pq-cli)](https://socket.dev/npm/package/kxco-pq-cli)
[![license](https://img.shields.io/badge/license-Apache--2.0-blue)](./LICENSE)
[![node](https://img.shields.io/node/v/kxco-pq-cli.svg)](https://nodejs.org)

CLI for KXCO post-quantum institution key management. Generates ML-DSA-65 keypairs, rotates institution keys with optional on-chain anchoring, signs files, and verifies signatures — without writing any code.

## When to use this

- Institutions managing their post-quantum identity from the command line
- DevOps and infra teams who need key rotation without writing Node.js
- Scripting identity operations in CI/CD pipelines

If you need to do any of this programmatically in your own application, use [`kxco-post-quantum`](https://www.npmjs.com/package/kxco-post-quantum) or [`kxco-post-quantum-webhook`](https://www.npmjs.com/package/kxco-post-quantum-webhook) instead.

## Install

```bash
npm install -g kxco-pq-cli
kxco-pq --help
```

You also need `kxco-post-quantum` available as a peer dependency:

```bash
npm install -g kxco-post-quantum
```

## Commands

### `kxco-pq keygen`

Generate a deterministic ML-DSA-65 keypair from a 32-byte master secret and an info label. Writes hex files to `--out-dir`.

```bash
kxco-pq keygen \
  --master 'ab83...64 hex chars...e7' \
  --info   'my-institution-v1' \
  --out-dir ./keys
```

Outputs:
- `keys/secret-key.hex` — 4032-byte secret key, hex-encoded. Store in a secrets manager, `chmod 600`. Never commit.
- `keys/public-key.hex` — 1952-byte public key, hex-encoded.
- `keys/kid.txt` — 16-character hex fingerprint. This is what receivers pin.

The keypair is deterministic: same `--master` + same `--info` always produces the same kid. Restore from master; never lose a key.

---

### `kxco-pq fingerprint`

Compute the kid for a public key without spinning up any application code.

```bash
kxco-pq fingerprint @./keys/public-key.hex
```

Accepts a hex string directly or a `@file` reference. Prints the 16-char hex kid.

---

### `kxco-pq rotate`

Rotate to a new keypair. Derives the new keypair, builds a signed rotation manifest (signed by the outgoing key so existing receivers can verify the handoff), and produces an updated `.well-known/kxco-pq-pubkey` document.

```bash
kxco-pq rotate \
  --old-secret @./current-keys/secret-key.hex \
  --old-kid    a1b2c3d4e5f60718 \
  --new-master '<32-byte master for the new key, hex>' \
  --info       'my-institution-v2' \
  --issuer     'chain.kxco.ai' \
  --out-dir    ./rotated-keys
```

Outputs (in `--out-dir`):
- `secret-key.hex`, `public-key.hex`, `kid.txt` — new keypair
- `manifest.json` — RFC 8785 JCS-canonical rotation manifest, signed by the old kid
- `well-known.json` — ready to publish at `https://<issuer>/.well-known/kxco-pq-pubkey`

After running:
1. Publish `well-known.json` at the well-known URL.
2. Publish `manifest.json` at `https://<issuer>/.well-known/kxco-pq-rotation/<new-kid>.json`.
3. Tell receivers to add the new kid to their `pinnedKids[]` alongside the old one.
4. After the drain window, retire the old kid and discard its secret key.

---

### `kxco-pq attest sign`

Sign any file with ML-DSA-65 and emit a self-contained JSON attestation envelope.

```bash
kxco-pq attest sign \
  --secret-key @./keys/secret-key.hex \
  --public-key @./keys/public-key.hex \
  --file       payload.json \
  --out        payload.attestation.json
```

The envelope contains `algorithm`, `signerKid`, `issuedAt`, `payload` (base64url), and `signature` (base64url ML-DSA-65). Any counterparty can verify it without trust delegation.

---

### `kxco-pq attest verify`

Verify an attestation envelope against a known public key.

```bash
kxco-pq attest verify \
  --public-key  @./keys/public-key.hex \
  --attestation payload.attestation.json
```

Prints `VALID` with signer kid, issue timestamp, and payload size — or `INVALID` with a reason and exits 1.

---

## Key rotation on-chain

Pass `--relay` and `--identity-file` to anchor the rotation to the KXCO chain in the same operation:

```bash
kxco-pq rotate \
  --old-secret    @./current-keys/secret-key.hex \
  --old-kid       a1b2c3d4e5f60718 \
  --new-master    '<new master hex>' \
  --info          'my-institution-v2' \
  --issuer        'chain.kxco.ai' \
  --out-dir       ./rotated-keys \
  --relay         https://relay.kxco.ai \
  --identity-file ./identity.json
```

`--identity-file` must be a JSON file containing `{ "kid": "<hex>", "secretKey": "<hex>" }` — the institution identity used to sign the chain transaction. On success the command prints the transaction hash and block number alongside the standard rotation output.

## What this does NOT do

- It is not a wallet. It does not hold, transfer, or custody assets of any kind.
- It is not for managing end-user credentials. If you need to issue or verify user-level post-quantum identities programmatically, use [`kxco-post-quantum`](https://www.npmjs.com/package/kxco-post-quantum).

## Part of the KXCO stack

| Package | Purpose |
|---------|---------|
| [`kxco-post-quantum`](https://www.npmjs.com/package/kxco-post-quantum) | Core ML-DSA-65 primitives (keygen, sign, verify, fingerprint) |
| [`kxco-post-quantum-webhook`](https://www.npmjs.com/package/kxco-post-quantum-webhook) | Runtime webhook signing and verification for Node.js frameworks |
| `kxco-pq-cli` | Operator CLI — keygen, rotation, attestation; no application code required |

All cryptographic operations delegate to `kxco-post-quantum`, which wraps [`@noble/post-quantum`](https://github.com/paulmillr/noble-post-quantum) — audited by Cure53 (2024). Private key bytes are never echoed to stdout.

To report a vulnerability, open a [private security advisory](https://github.com/JackKXCO/kxco-pq-cli/security/advisories/new) or email **security@kxco.ai**.

## License

Apache 2.0. See [LICENSE](./LICENSE).

## Maintainers

Shayne Heffernan and John Heffernan — [KXCO by Knightsbridge](https://kxco.ai)

[Knightsbridge Law](https://knightsbridge.law) · [target150.com](https://target150.com) · [livetradingnews.com](https://livetradingnews.com)
