# v7.67 Conformance Vectors — Crypto-Agility

**Status (2026-06-10):** vector **definitions** landed 2026-06-09 (v7.67 spec catch-up merge); **Phase 1 byte fixtures + Phase 2 seed convention LANDED 2026-06-10** in `conformance-vectors-v1.diag` + `SEEDS.md` (this directory). Phase 1 is byte-pinned cross-impl (Go × Rust × Python byte-equal 2026-06-09; cohort confirmation in `entity-core-go/docs/validation/V7.67-PHASE1-COHORT-CROSS-IMPL-PIN-2026-06-09.md`). Phase 2 has the seed convention ratified architecture-side (`SEEDS.md` §2, ratifying Go's `V767-FIXTURE-SEED-RECONCILIATION-2026-06-10.md` §3.1 with three arch clarifications); byte pins for M2/M3/M6 are produced by the cohort round-trip per `SEEDS.md` §5 step 2. Phase 3a/3b stay deferred per v7.67 §13.7. The sibling `conformance-vectors-v1.cbor` is the deterministic ECF-canonical encoding produced from `.diag` by any conformant encoder (cohort produces in the round-trip pass; bytes MUST match the inline canonical: h'...' values in `.diag`).

Normative definition source: `proposals/implemented/PROPOSAL-V7-V7.67-CRYPTO-AGILITY-SEED-TABLES.md` §6 (matrix) + §7 (vectors). Spec homes: V7 §1.2 (`content_hash_format` seed table), §1.5 (`key_type` seed table).

---

## Vector inventory

### Phase 1 — allocation + machinery (LANDED cross-impl: Go × Rust × Python byte-equal)

| Vector | Validates | key_type / hash_format |
|---|---|---|
| `KEY-TYPE-ED448-1` | `system/peer({public_key, key_type="ed448"})` → canonical `(0x02, 0x01)` peer_id; `content_hash` byte-equal cross-impl; sign/verify round-trip on a fixed seed | Ed448 `0x02` / SHA-256-form |
| `HASH-FORMAT-SHA-384-1` | `content_hash` under `content_hash_format=0x01` byte-equal cross-impl for a canonical fixture entity; store + retrieve round-trip | Ed25519 `0x01` / SHA-384 `0x01` |
| `VARINT-MULTIBYTE-1` | Impl decodes a `system/hash` with a multi-byte LEB128 format code (fixture uses `0x80`, first two-byte encoding) and rejects with `unsupported_content_hash_format` (`0x80` unallocated). Confirms the multi-byte decode path exists. | n/a |
| `VARINT-RESERVED-FF-1` | Impl rejects constructing a `system/peer` with `key_type` value 255 (`0xFF 0x01`) and a `system/hash` with format-code value 255 | n/a |
| `FORMAT-CODE-INTERPRETATION-1` | (renamed from v7.66 `PREFIX-DISPATCH-1`; semantics unchanged) impl receiving a `content_hash` with an unsupported format code returns `unsupported_content_hash_format` | n/a |

### Phase 2 — cross-key / cross-hash matrix, classical (LANDED cross-impl; 11/11 mixed-key pairs convergence-flat)

| Vector | Peer A | Peer B | Cap chain depth |
|---|---|---|---|
| `MATRIX-M2` | Ed448 / SHA-256 | Ed25519 / SHA-256 | ≥ 1 |
| `MATRIX-M3` | Ed25519 / SHA-384 | Ed25519 / SHA-256 | ≥ 1 |
| `MATRIX-M6` | Ed448 / SHA-384 | Ed25519 / SHA-256 | ≥ 1 |

Each matrix vector exercises: handshake (`hello` + `authenticate`, signature scheme + peer_id canonicalization per allocation) → tree-get on a small entity (content-store dispatch by format code) → cap grant A→B (mixed-`key_type` cap-pattern peer references) → cap use B→A (sig verification + cap-chain walk + format-code freeze).

### Phase 3a — DEFERRED to W2 backlog (BLAKE3 family)

| Vector | Notes |
|---|---|
| `HASH-FORMAT-BLAKE3-1` | `content_hash` under `content_hash_format=0x03` (BLAKE3-256) |
| `MATRIX-M5` | Ed25519/BLAKE3 ↔ Ed25519/SHA-256; format-code freeze across chain links |

### Phase 3b — DEFERRED to W2 backlog (ML-DSA-65 / PQ)

| Vector | Notes |
|---|---|
| `KEY-TYPE-ML-DSA-65-1` | `key_type="ml-dsa-65"`; canonical `(0x03, 0x01)`; sign/verify round-trip on a fixed seed |
| `MATRIX-M4` | ML-DSA-65/SHA-256 ↔ Ed25519/SHA-256; **cap chain depth ≥ 3** (PQ size accumulation) |
| `MATRIX-M7` | ML-DSA-65/BLAKE3 ↔ Ed25519/SHA-256; **cap chain depth ≥ 3** (combined PQ size + non-SHA-2) |
| `MATRIX-M8` | ML-DSA-65/SHA-384 ↔ Ed448/BLAKE3; single-permutation regression check (does NOT replace a fuzz-over-pairs harness — descoped) |

---

## Fixture byte status

**Phase 1 — LOCKED.** All five Phase-1 vectors are byte-pinned in `conformance-vectors-v1.diag` with the seed convention ratified in `SEEDS.md` §1. Cohort cross-impl byte-equality validated 2026-06-09 (Go `9c56d5d` × Rust `2c81a20` × Python `614cb0f`/`f231406`); arch-side authoring 2026-06-10 reconciles Go's `V767-FIXTURE-SEED-RECONCILIATION-2026-06-10.md` §§2.1–2.4 with V7 §§1.2/1.3/1.5 algorithm text. Per the corpus-authoring discipline (proposal §7.6 / v7.66 §7.2): pinned bytes derive from the spec algorithm; the inline canonical values in `.diag` are the lock.

**Phase 2 — LOCKED 2026-06-10.** The three matrix vectors (`MATRIX-M2`, `MATRIX-M3`, `MATRIX-M6`) are byte-pinned in `conformance-vectors-v1.diag` from a 3-way cohort round-trip (Go post-fix `3cfb353`, Rust `d38d1f8`, Python `a2463be`). All 7 gates per matrix vector byte-equal across all three impls: pubkeys + peer_ids + home content_hashes + cap-data CBOR + active cap content_hash + signature. The round-trip surfaced one latent cross-impl divergence (Go's `CapabilityScope` was emitting `{include: null}` `0xf6` for unconstrained scope dimensions via an fxamacker default; Rust + Python emit `{include: []}` `0x80`); Go's fix at `3cfb353` lands all three on the spec-canonical `[]`. Architecture ruled on the underlying convention in `reviews/RULING-EMPTY-SCOPE-INCLUDE-2026-06-10.md` (no spec change — both halves already bound by ENTITY-CBOR-ENCODING §232 + V7 §3.6 list-of(pattern) typing). Cohort closeouts: `entity-core-go/docs/validation/V767-PHASE2-BYTE-PINS-COHORT-2026-06-10.md` (v2 post-fix); `entity-core-rust/docs/validation/V767-PHASE2-BYTE-PINS-RUST-2026-06-10.md`; `entity-core-py/docs/validation/V767-PHASE2-BYTE-PINS-PYTHON-2026-06-10.md`.

**Build-artifact `.cbor` regenerated 2026-06-10 (PM) under F16.** The prior `.cbor` sha `4d8dfced…c6d0ae` (7913 B, locked by the cohort round-trip at step 4) was decoded end-to-end by Keystone during its FFI agility-surface bring-up and found internally inconsistent with its own `.diag`: Ed448 `secret_seed` fields were written 58 B (RFC says 57), the `experimental-test` `public_key` was written 63 B (should be 64), and every Phase-2 `expected_*` field was still the literal text `"TBD-COHORT-ROUND-TRIP"` (12 fields). The 3-way "byte-equal" agreed on the same `.cbor` sha because the round-trip compared crypto outputs + the file's sha, never decoded the file. Cohort cryptographic outputs (the `expected_*` byte values folded into `.diag`) are unaffected. Architecture re-emitted the `.cbor` from the `.diag` via `tools/regen-v767-cbor.py` (canonical ECF: definite-length, minimal integer encoding, length-then-lexicographic map keys per ENTITY-CBOR-ENCODING §4.1). **New sha256 = `8e7c5232f64bee83d628679f930c771e4e49f2f1e37d19e41e0d7838e31f982e`** (9236 B). The prior sha `4d8dfced…c6d0ae` is superseded. Full close-out: `reviews/CLOSEOUT-F16-AGILITY-CORPUS-CBOR-REGEN-2026-06-10.md`. Cohort action: re-verify against the regenerated bytes (no crypto/output change expected; only the fixture-file's input-side seed/pubkey byte-widths and Phase-2 `expected_*` field types change).

**Phase 3a/3b — DEFERRED.** BLAKE3 + ML-DSA-65 seeds not pinned (v7.67 §13.7).

Pinned seeds (cross-reference for impl authoring):
- **Ed448** — 57-byte secret seed (RFC 8032), byte `0x42` × 57 for `KEY-TYPE-ED448-1` / `MATRIX-M2`-A; `0x46` × 57 for `MATRIX-M6`-A. Library expansion (SHAKE256) happens inside the signing library; no pre-expansion.
- **Ed25519** — 32-byte secret seed (RFC 8032), `0x43` × 32 for `MATRIX-M2`-B; `0x44`/`0x45` for `MATRIX-M3` A/B; `0x47` × 32 for `MATRIX-M6`-B.
- **SHA-384** — re-hash of the v7.66 `AGILITY-ENTITY-1` fixture (`system/peer({pub=0xAA×64, key_type='experimental-test'})`) under `content_hash_format=0x01`. Same source entity as the inherited SHA-256 corpus pin.
- **ML-DSA-65 / BLAKE3** — to be pinned at Phase-3b/3a pickup.
