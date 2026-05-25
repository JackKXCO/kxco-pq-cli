# kxco-pq-cli

[![npm](https://img.shields.io/npm/v/kxco-pq-cli?label=npm&color=b0964f)](https://www.npmjs.com/package/kxco-pq-cli)
[![Socket](https://socket.dev/api/badge/npm/package/kxco-pq-cli)](https://socket.dev/npm/package/kxco-pq-cli)
[![license](https://img.shields.io/badge/license-Apache--2.0-blue)](./LICENSE)
[![node](https://img.shields.io/node/v/kxco-pq-cli.svg)](https://nodejs.org)

CLI tooling for the [`kxco-post-quantum`](https://www.npmjs.com/package/kxco-post-quantum) ecosystem. Produces deterministic ML-DSA-65 keypairs from a master + info label, computes kid fingerprints, and orchestrates **signed rotation manifests** for the [webhook contract's key-rotation flow](https://github.com/JackKXCO/kxco-post-quantum-webhook/blob/main/docs/webhook-contract.md#key-rotation-and-history).

This is the ops-side companion to [`kxco-post-quantum-webhook`](https://www.npmjs.com/package/kxco-post-quantum-webhook). The webhook library does signing/verification at runtime; this CLI is for the keygen + rotation events that happen out-of-band, on a hardened machine, by a human.

## Install + run

```bash
# One-shot, no install
npx kxco-pq --help

# Or install globally
npm install -g kxco-pq-cli
kxco-pq --help
```

You also need `kxco-post-quantum` available as a peer dep:

```bash
npm install kxco-post-quantum   # if running globally
```

## Commands

### `kxco-pq keygen`

Generate a deterministic ML-DSA-65 keypair from a 32-byte master + an info label. Writes hex files to `--out-dir`.

```bash
kxco-pq keygen \
  --master 'ab83...64 hex chars...e7' \
  --info 'my-app-v1' \
  --out-dir ./keys
```

Outputs:
- `keys/secret-key.hex` — 8064 hex chars (4032 bytes). **Treat like any other private key**: `chmod 600`, store in a secrets manager, never commit.
- `keys/public-key.hex` — 3904 hex chars (1952 bytes).
- `keys/kid.txt` — 16 hex chars. The fingerprint receivers pin.

The keypair is deterministic: same `--master` + same `--info` always produces the same kid. This is intentional — restore from master, never lose a key.

### `kxco-pq fingerprint`

Compute the kid for a public key. Useful for confirming a kid out-of-band without spinning up Node.

```bash
kxco-pq fingerprint 'ab83...3904 hex chars...e7'
# or
kxco-pq fingerprint @./keys/public-key.hex
```

Prints the 16-char hex kid.

### `kxco-pq rotate`

The main event. Atomically:

1. Derives a NEW keypair from `--new-master` + `--info`.
2. Builds a **rotation manifest** signed by the OLD secret key — receivers who already trust the old kid can verify the bridge to the new kid.
3. Builds an updated `.well-known/kxco-pq-pubkey` JSON document with both kids in the history.

```bash
kxco-pq rotate \
  --old-secret @./current-keys/secret-key.hex \
  --old-kid    a1b2c3d4e5f60718 \
  --new-master '<32-byte master for the NEW key, hex>' \
  --info       'my-app-v2' \
  --issuer     'chain.kxco.ai' \
  --out-dir    ./rotated-keys
```

Outputs (in `--out-dir`):
- `secret-key.hex`, `public-key.hex`, `kid.txt` — the NEW keypair files
- `manifest.json` — RFC 8785 JCS-canonical, signed by the OLD kid
- `well-known.json` — ready to publish at `https://<issuer>/.well-known/kxco-pq-pubkey`

Then:

1. Publish `well-known.json` at the well-known URL.
2. Publish `manifest.json` at `https://<issuer>/.well-known/kxco-pq-rotation/<new-kid>.json`.
3. Tell receivers to add the new kid to their verifier's `pinnedKids[]` alongside the old one.
4. After your drain window, retire the old kid (mark `status: 'retired'`).
5. Discard the old secret key.

The [rotation playbook](https://github.com/JackKXCO/kxco-post-quantum-webhook/blob/main/docs/key-rotation-playbook.md) covers each of these steps in detail.

### `kxco-pq attest sign`

Sign any file with ML-DSA-65 and emit a self-contained JSON attestation envelope. Any counterparty can verify it without trust delegation.

```bash
kxco-pq attest sign \
  --secret-key @./keys/secret-key.hex \
  --public-key @./keys/public-key.hex \
  --file payload.json \
  --out payload.attestation.json
```

Outputs a JSON envelope with `algorithm`, `signerKid`, `issuedAt`, `payload` (base64url), and `signature` (base64url ML-DSA-65).

### `kxco-pq attest verify`

Verify an attestation envelope against a known public key.

```bash
kxco-pq attest verify \
  --public-key @./keys/public-key.hex \
  --attestation payload.attestation.json
```

Prints `VALID` with signer kid, issue timestamp, and payload size — or `INVALID` with a reason and exits 1.

## Why a separate package

Rotation events are rare and run from operator workstations — they don't need to live in the runtime webhook library. Splitting the CLI out keeps `kxco-post-quantum-webhook` small (5 framework adapters + verifiers, no CLI overhead in cold-start environments like Vercel Edge / Cloudflare Workers).

## Wire-format compatibility

The manifest and well-known shapes this CLI emits are stable under the [webhook contract](https://github.com/JackKXCO/kxco-post-quantum-webhook/blob/main/docs/webhook-contract.md). A receiver in Rust/Go/Python that implements the contract can verify manifests this CLI produces without depending on Node.

## Security

All cryptographic operations delegate to [`kxco-post-quantum`](https://www.npmjs.com/package/kxco-post-quantum), which wraps [`@noble/post-quantum`](https://github.com/paulmillr/noble-post-quantum) — audited by Cure53 (2024). Key material is read from files or environment variables; private key bytes are never echoed to stdout. Run rotation events on a hardened, air-gapped or access-controlled machine.

To report a vulnerability, open a [private security advisory](https://github.com/JackKXCO/kxco-pq-cli/security/advisories/new) or email **security@kxco.ai**.

## Funding

Maintained by **Shayne Heffernan** and **John Heffernan** at [KXCO by Knightsbridge](https://kxco.ai).

[Knightsbridge Law](https://knightsbridge.law) · [target150.com](https://target150.com) · [livetradingnews.com](https://livetradingnews.com)

## License

Apache 2.0. The CLI emits artefacts consumable by the (Apache-licensed) webhook package and the (MIT-licensed) `kxco-post-quantum` primitive package. See [LICENSE](./LICENSE).

## Maintainers

Shayne Heffernan · John Heffernan — [KXCO by Knightsbridge](https://kxco.ai)

Deployed in production at [target150.com](https://target150.com), [knightsbridgelaw.com](https://knightsbridgelaw.com), [livetradingnews.com](https://livetradingnews.com).
