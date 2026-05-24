# Changelog

## 0.1.5 — 2026-05-24

Maintenance release. No breaking changes.



## 0.1.4 — 2026-05-24

Maintenance release. No breaking changes.



## 0.1.3 — 2026-05-24

Maintenance release. No breaking changes.



## 0.1.2 â€” 2026-05-24

Fix bin entry that was being stripped during publish (removed `./` prefix).

## 0.1.1 â€” 2026-05-24

Maintenance release. No breaking changes.

## 0.1.0 â€” 2026-05-22

Initial release.

### Added
- `kxco-pq keygen` â€” derive ML-DSA-65 keypair from master secret + label via HKDF
- `kxco-pq fingerprint` â€” compute 16-hex KID from a public key hex string or file
- `kxco-pq rotate` â€” generate signed rotation manifest + well-known JSON for key transitions
- RFC 8785 JCS-canonical signing of rotation manifests using the old identity key
- `--master` / `--new-master` / `--old-secret` accept inline 64-hex or `@/path/to/file`
- Output: `secret-key.hex`, `public-key.hex`, `kid.txt` (keygen + rotate)
- Output: `well-known.json`, `manifest.json` (rotate only)
- Deterministic keygen matches `kxco-post-quantum` library derivation pattern
- 20+ tests: JCS canonicalisation, manifest signing, keygen, fingerprint, rotate
