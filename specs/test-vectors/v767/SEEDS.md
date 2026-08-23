# v7.67 Conformance Corpus — Seed Pins (ratified)

**Status**: Ratified architecture-side 2026-06-10. Ratifies the Go cohort's `V767-FIXTURE-SEED-RECONCILIATION-2026-06-10.md` §§2.1–2.4 verbatim for Phase 1 and §3.1 (with three arch clarifications below) for Phase 2.
**Source**: V7 §1.2 (`content_hash_format` seed table) + §1.5 (`key_type` seed table) + §1.3 (ECF canonical encoding) + §4.5/§4.5a (negotiated format semantics per v7.69) + §1.2/§1.2a (home format vs active format per v7.70).
**Cohort confirmation**: Phase 1 values byte-equal cross-impl 2026-06-09 (Go `9c56d5d` × Rust `2c81a20` × Python `614cb0f`/`f231406`). Phase 2 seeds ratified here; cohort regression-runs M2/M3/M6 with these pinned seeds to produce byte pins in the next round.

This document is the SINGLE SOURCE OF TRUTH for the v7.67 corpus seeds. The sibling `conformance-vectors-v1.diag` carries the same values inline per the ECF-corpus convention; if anything diverges, `SEEDS.md` wins and `.diag` is corrected.

---

## §1 Phase 1 — byte-pinned (cohort-validated)

### §1.1 Ed448 fixture — `KEY-TYPE-ED448-1`

| Pin | Value |
|---|---|
| Secret seed | 57 bytes, every byte `0x42`. Hex: `42` × 57. |
| Library call | `cloudflare/circl/sign/ed448::NewKeyFromSeed(seed)` (Go reference); equivalent direct-seed APIs in Rust + Python. **No pre-expansion**: the 57 bytes are passed verbatim to the library; the RFC 8032 SHAKE256 expansion happens inside. |
| Fixture sign-message | ASCII, 45 bytes, no trailing newline: `v7.67 Phase 1 cohort cross-impl Ed448 fixture`. Hex: `76372e3637205068617365203120636f686f72742063726f73732d696d706c2045643434382066697874757265`. |
| `public_key` (57 B) | `2601850dc77aaf141e065b2fe83ecfe08b6c15ba930886e9f111b6f0fd8f9f246b167e0398f957df61c9cead939cdf5bc9fe43c9432f3b0e00` |
| Canonical `(key_type, hash_type)` | `(0x02, 0x01)` SHA-256-form per V7 §1.5 size-cutoff rule (Ed448 pubkey > 32 bytes → not identity-multihash). |
| `peer_id` (Base58) | `3dR1gAppfHXSGMvPRuAfYkkt4P2C1fvnFYpxPBSQP8RLs4` |
| `signature` over the fixture message (114 B) | `0aff7a36b2b5e7502f9a133bc9ed39316284f0be738e2485546b33fda60966b19ac0e3424ed549072af7ac5caa6d695c3e1e6412207cecaf8085444fbf062cb5271ea6d127c6c87327e1e20793f2b10341d04bd4bed32e220eca1b2255cc8aa4d2a0c8304d67e6f20e814b90411049b33400` |

**Derived `system/peer` entity (ECF-canonical CBOR)**:

| Field | Value |
|---|---|
| `entity.Type` | `system/peer` |
| `entity.Data` raw CBOR (data map, ECF-deterministic field order `key_type` then `public_key`) | `a2686b65795f747970656565643434386a7075626c69635f6b657958392601850dc77aaf141e065b2fe83ecfe08b6c15ba930886e9f111b6f0fd8f9f246b167e0398f957df61c9cead939cdf5bc9fe43c9432f3b0e00` |
| `content_hash` (33 B wire = `0x00` SHA-256 format byte + 32 B digest) | `002785b314436a82503829339cb2519b4efe795712406ea19ac185e31ae8c70748` |

### §1.2 SHA-384 fixture — `HASH-FORMAT-SHA-384-1`

This vector **inherits the v7.66 `AGILITY-ENTITY-1` corpus fixture**; it is *not* a fresh seed.

> **Inverted `[FOLDED]` — the fixture is a `system/peer`, and a `system/peer` has exactly one content_hash.** This vector was written to re-hash the fixture under `content_hash_format = 0x01` and pin the result, on the theory that the same canonical bytes should produce a spec-required digest under a second hash family. **`ENTITY-CORE-PROTOCOL` §4.5a item 1a (v7.77) retired the construction it was pinning**: the `system/peer` identity entity is authored at the ECFv1-SHA-256 floor unconditionally, so there is no SHA-384 form of it to assert. `.2.rehash` now asserts the refusal instead, and the verifier MUST route through the pinned constructor — it stayed green for a week only because it hand-built the entity and bypassed the code that would have refused it.
>
> **Coverage consequence, stated because inverting loses something real:** this was the corpus's only *positive* SHA-384 content-hash vector, and the corpus no longer exercises SHA-384 digest computation over an entity in the affirmative. **A replacement positive vector belongs on a non-`system/peer` type** (any entity that legitimately carries a home format) and is **owed** — it is not created here, because retargeting this vector is what §4.4 explicitly ruled against. Until it lands, `0x01` is exercised only by refusal.

| Pin | Value |
|---|---|
| Fixture entity | `system/peer({ public_key: 0xAA × 64, key_type: "experimental-test" })` (the v7.66-allocated `0xFE` test-only stub; substrate floor forces SHA-256-form / `hash_type=0x01` regardless of content_hash_format). |
| Public-key seed | 64 bytes, every byte `0xAA`. |
| ECF-encoded `{data, type}` input length | 128 bytes |
| SHA-256 content_hash (33 B wire, inherited v7.66 `AGILITY-ENTITY-1` pin) | `003d0c34b508c5bf9eca5f086f09aac10f44bd43fca1a091b6aa55a096ca8fcd45` |
| SHA-384 form | **None — does not exist.** `system/peer` is floor-pinned by §4.5a item 1a; authoring it under `0x01` MUST be refused (`.2.rehash`). The retired pin was `012e64bbde…3eef5a69`; it is recorded here as history, not as an expectation. |

### §1.3 Varint probes — `VARINT-MULTIBYTE-1` / `VARINT-RESERVED-FF-1` / `FORMAT-CODE-INTERPRETATION-1`

These are **probe vectors** — they pin REJECTION behavior, not byte outputs. No keypair or message seeds required.

| Vector | Probe input | Expected behavior |
|---|---|---|
| `VARINT-MULTIBYTE-1` | `system/hash` with format-code bytes `80 01` (LEB128 of integer 128; multi-byte decode path) | Impl decodes the varint; resolves to integer 128; integer 128 is not in §1.2 seed table → surface `unsupported_content_hash_format` error. **Pins that the multi-byte LEB128 decoder exists and the unsupported-code error fires from it (not from a single-byte short-circuit).** |
| `VARINT-RESERVED-FF-1` | (a) `system/peer` mint with `key_type` integer 255 (varint `FF 01`). (b) `system/hash` with format-code integer 255. | Both refused. 255 is reserved per V7 §1.5 / §1.2. |
| `FORMAT-CODE-INTERPRETATION-1` | `system/hash` with an unallocated format-code byte (e.g. `0x42`). Renamed from v7.66 `PREFIX-DISPATCH-1`; semantics unchanged. | Surface `unsupported_content_hash_format`. Pins that an unknown code is an INTERPRETATION error (not a routing dispatch issue) — V7 §1.2 reframe per v7.68. |

---

## §2 Phase 2 — matrix vector seed convention (ratified; cohort byte-equality round-trip pending)

Matrix vectors `MATRIX-M2`, `MATRIX-M3`, `MATRIX-M6` exercise the cross-key / cross-hash agility surface end-to-end: handshake → tree-get → cap grant A→B → cap use B→A. The cohort's Phase-2 cross-impl pass used **ephemeral keypairs** (process-start `crypto/rand`); seeds were never pinned. To make Phase 2 a byte-pinned corpus item rather than only a wire-flow assertion, architecture ratifies the Go-proposed §3.1 scheme below, with three arch clarifications added to nail down cross-format semantics that the bare scheme leaves implicit.

### §2.1 Per-peer keypair seeds (ratified from Go §3.1 Table)

| Matrix vector | Peer A keypair seed | Peer B keypair seed |
|---|---|---|
| `MATRIX-M2` | Ed448 from seed `0x42` × 57 (same as Phase-1 `KEY-TYPE-ED448-1` — intentionally reuses the Phase-1 Ed448 identity) | Ed25519 from seed `0x43` × 32 |
| `MATRIX-M3` | Ed25519 from seed `0x44` × 32 — peer home `content_hash_format = 0x01` | Ed25519 from seed `0x45` × 32 — peer home `content_hash_format = 0x00` |
| `MATRIX-M6` | Ed448 from seed `0x46` × 57 — peer home `content_hash_format = 0x01` | Ed25519 from seed `0x47` × 32 — peer home `content_hash_format = 0x00` |

Each seed is the RFC 8032 secret-seed input: 32 bytes for Ed25519, 57 bytes for Ed448, every byte the indicated value.

### §2.2 Active format — arch clarification #1 (negotiation outcome pinned)

Per V7 §4.5/§4.5a (v7.69), the connection's *active* `content_hash_format` is determined by single-active-value first-match from the initiator's preference list. **The matrix vectors fix the initiator and the active format as follows**, so all three impls converge:

| Matrix vector | Initiator | A `hash_formats` (preference order) | B `hash_formats` | Active format |
|---|---|---|---|---|
| `MATRIX-M2` | A | `[0x00]` | `[0x00]` | `0x00` (SHA-256) |
| `MATRIX-M3` | A | `[0x01, 0x00]` (initiator prefers home, falls back to standard) | `[0x00]` | `0x00` (SHA-256) — first-match is B's only supported format |
| `MATRIX-M6` | A | `[0x01, 0x00]` | `[0x00]` | `0x00` (SHA-256) |

**Net**: in all three matrix vectors the connection's active format is SHA-256, and cap-token entities carry SHA-256 content_hashes. Mixed-home (M3/M6) is the experimental annex per v7.70 §1.2a; the corpus pins the standard-compliance fallback path (active = SHA-256) because that is what a conformant peer MUST handle, not the experimental cross-format-on-wire reading.

### §2.3 Cap-token payload (ratified from Go §3.1)

Each matrix vector exercises a single root cap grant A→B with the payload below. Fixed-zero timestamps make the cap-token entity's content_hash deterministic from the seed inputs alone.

```
CapabilityTokenData {
  grantee:    content_hash(B's system/peer)           // see §2.4 below
  granter:    SingleSig {
                hash: content_hash(A's system/peer)   // see §2.4 below
              }
  grants:     [
                GrantEntry {
                  resources: CapabilityScope {
                    include: ["system/validate/matrix/*"]
                  }
                }
              ]
  parent:     null     // root cap
  created_at: 0        // fixed; deterministic content_hash
  expires_at: 0        // 0 = no expiry (the matrix vectors don't exercise expiry)
}
```

**Notes**:
- `grantee` is the wire content_hash of B's `system/peer` entity. Per v7.69 §1.8, references to identities use the identity's home-format content_hash (NOT re-derived under the active format).
- `granter.hash` is the wire content_hash of A's `system/peer` entity, under A's home format.
- The cap-token entity ITSELF has a content_hash under the active format (SHA-256 in all three matrix vectors per §2.2).
- `expires_at: 0` is the matrix-vector convention for "no expiry"; impls that gate on `expires_at <= now()` MUST treat 0 as "infinite past" only for the impl-defined sentinel, not for these corpus vectors. The `AUTHZ-EXPIRED-1` vector in GUIDE-CONFORMANCE §9 (v7.71) covers the expiry behavior separately.

### §2.4 Identity references — arch clarification #2 (home-format pin)

For M3 and M6, Peer A's home format is SHA-384 and Peer B's is SHA-256. The cap-token references each peer by THAT peer's home-format content_hash:

- `grantee` (refers to B): SHA-256 content_hash of B's `system/peer` entity.
- `granter.hash` (refers to A): ~~SHA-384 content_hash of A's `system/peer` entity (for M3/M6)~~ — **superseded, see the correction below**; SHA-256 for M2.

The cap-token entity itself is authored under the active format (SHA-256 — see §2.2), so the wire cap-token content_hash is SHA-256-form even when it references a SHA-384 identity. **This is the v7.69/v7.70 invariant**: content travels in active format, references to identities use the identity's home format.

> **Correction `[2026-08-12]` — item 1a carved `system/peer` out of this clarification, and this text was not swept.** The invariant in the paragraph above still holds **generally**: content travels in active format, references to identities use the identity's home format. But **`ENTITY-CORE-PROTOCOL` §4.5a item 1a (v7.77) made `system/peer` the one type with no home format** — it is authored at the ECFv1-SHA-256 floor unconditionally, because its data (`{peer_id, public_key, key_type}`) is wholly recoverable from the public peer-id and so an entity nobody fetches to learn its hash cannot be hold-and-fetch. **Every identity reference in this corpus therefore resolves floor-form, including `granter.hash` for M3 and M6.**
>
> **This is the root of all six stale M3/M6 assertions, and the chain is worth stating once:** `peer_a`'s content hash moves to the floor (assertions 1 and 4) → the root cap references it via `granter.hash`, so the cap's own content hash moves (2 and 5) → the signature is over that content hash, so it moves too (3 and 6). One cause, six symptoms, and the proof it is one cause is that **every `peer_b` SHA-256 assertion stays green** — peer_b was already at the floor, and 1a did not move it.
>
> **The failure worth recording is that this clarification is where the sweep should have started and did not.** §2.4 is a *ratified normative statement* that 1a contradicted; the fixture is only its downstream consequence. `entity-core-go` found the six failing expectations by running the verifier and correctly derived replacements consistent with 1a — but the derivation contradicted this section's text, and **the disagreement between the fixture and the clarification went unnoticed by everyone, because a red fixture is visible and a stale sentence is not.** When a pinned primitive moves, sweep the prose that pins it, not only the artifacts that failed.

### §2.5 Signature target — arch clarification #3 (RFC 8032 deterministic)

Per V7 §5.2 cap-chain verification, A signs the root-cap content_hash. Concretely:

```
signature = ed_sign(
  secret_key = A's_keypair,      // from §2.1 seed
  message    = content_hash(root_cap)   // the cap-token entity's wire content_hash, see §2.3
)
```

`message` is the full wire content_hash (algorithm byte + digest), 33 bytes for SHA-256-form. Both Ed25519 and Ed448 sign deterministically per RFC 8032, so the signature bytes are byte-equal across any conformant library given the same seed + message.

### §2.6 Expected byte pin shape (cohort produces in round-trip)

For each matrix vector, the cohort produces (and the next-round corpus update pins) the tuple:

```
{
  "id": "matrix.M2",  // or M3 / M6
  "peer_a_pubkey":   hex,
  "peer_a_peer_id":  Base58,
  "peer_a_content_hash":  hex (ECFv1-SHA-256 floor — see correction below),
  "peer_b_pubkey":   hex,
  "peer_b_peer_id":  Base58,
  "peer_b_content_hash":  hex (ECFv1-SHA-256 floor — see correction below),
  "root_cap_cbor":          hex (cap-token entity {type, data} CBOR, active format),
  "root_cap_content_hash":  hex (active format = SHA-256),
  "root_cap_signature":     hex (A's signature over root_cap_content_hash)
}
```

The cohort regression-runs M2/M3/M6 with the §2.1 seeds substituted for ephemeral keys, captures these tuples, and ships them to architecture in a one-line "byte-equal vs the §2.1+§2.3+§2.5 derivation ✅" plus the captured pins. Architecture then folds them into `conformance-vectors-v1.diag` Phase-2 vector entries.

> **Correction `[2026-08-12]` — `peer_*_content_hash` is floor-form, not home-form.** The two annotations above read `hex (home-format)` until this correction. **`ENTITY-CORE-PROTOCOL` §4.5a item 1a (v7.77) pins the `system/peer` entity to the ECFv1-SHA-256 floor unconditionally** — every connection, whatever the active format, whatever the peer's home format. `system/peer`'s data is `{peer_id, public_key, key_type}`, wholly recoverable from the public peer-id, so it is derive-to-meet and never home-form. The `_sha384` rows of M3 and M6 are therefore floor-form (`00…`, 33 B), not SHA-384-form (`01…`, 49 B).
>
> **This annotation and the M3/M6 corpus expectations are the same defect, and it is arch's, not an implementer's** — 1a moved a value and the artifacts that pinned it were not swept. `entity-core-go` found the corpus half by running the verifier (`49 PASS, 6 FAIL` at `419a715`); this half was found by reading §2.6 while ruling on that report. **Anything that pinned a `system/peer` content hash before v7.77 is suspect until re-derived** — that is the sweep 1a owed and did not get.
>
> **The field name is unchanged and was always `peer_a_content_hash`.** The corpus's `expected_peer_a_content_hash_sha384` was a divergence from this ratified shape before 1a existed; renaming it back is conformance to §2.6, not a new decision. Under 1a the `_sha384` suffix additionally names a knob that can no longer exist.

---

## §3 Deferred (not pinned)

| Phase | Vector(s) | Status |
|---|---|---|
| **3a** | `HASH-FORMAT-BLAKE3-1`, `MATRIX-M5` | Deferred per v7.67 §13.7. No seeds. |
| **3b** | `KEY-TYPE-ML-DSA-65-1`, `MATRIX-M4`, `MATRIX-M7`, `MATRIX-M8` | Deferred per v7.67 §13.7. No seeds. |

When PQ pickup happens (post-release, on external signal), seeds get pinned then. The §2 scheme is the model — pick a seed-byte offset, document the per-peer home format, fix the active format outcome, list the cap-token payload with zeroed timestamps, and run the cohort round-trip.

---

## §4 Corpus encoding (`.cbor` build artifact)

The sibling `conformance-vectors-v1.cbor` is the deterministic ECF-canonical encoding of `conformance-vectors-v1.diag` per ENTITY-CBOR-ENCODING.md v1.5 Appendix E (same procedure as `ecf-conformance/`). Build step:

```
# Produced by the cohort during the next regression round. Bytes MUST match the
# inline canonical: h'...' values in the .diag. Build owner: Go (cohort
# test-vector convention); Rust + Python regression-confirm.
```

Architecture does NOT bless a specific encoder binary; the `.diag` is the spec-derived ground truth and the `.cbor` is its deterministic build output. Any conformant ECF encoder (Go `ecf.Encode`, Rust `ecf::encode`, Python `entity_core_codec.encode_ecf`) MUST produce byte-identical output. The Phase-1 entity bytes inlined in §§1.1–1.2 above are the lock against which the encoder is regression-checked.

### §4.1 Division of labour — architecture sets fields, the encoder settles bytes `[MUST]` `[RULED 2026-08-13]`

**Architecture edits the `.diag` source: vector `id`s, `description`s, `kind`s, inputs, and which assertion a vector makes. Architecture does NOT hand-derive, hand-edit, or hand-inspect encoded bytes.** No byte-run measuring, no hex-literal eyeballing, no `.cbor` edits, and no byte pin transcribed from a report into this corpus by hand. **A claim about what bytes an artifact contains is made by running a tool and quoting its output, never by reading the file.** The `.cbor` is a build artifact and architecture is not its build owner (§4).

**Rationale, from the two ways this went wrong in one week.** Values hand-carried into the corpus is how the M3/M6 rows went stale and how the F16 width correction reached the artifact but never the source. Both were then *found* by hand-inspection, which felt like diligence and is not a process: it does not run, it does not gate, and it does not survive the reviewer's attention span. **Every check below is mechanical and re-runnable; that is the whole point of writing them down.**

### §4.2 Two gates, and they assert different things `[MUST]` `[RULED 2026-08-13]`

Both are required. Either alone leaves the class the other catches invisible.

| Gate | Asserts | Catches |
|---|---|---|
| `v767-corpus-verify` | **artifact-is-expected** — decode the `.cbor`, re-derive the crypto, check structural invariants | a corrupted or wrongly-valued artifact |
| `v767-corpus-build -check` | **source-produces-artifact** — re-encode both `.diag`s, compare to the committed `.cbor`s, write nothing | **source and artifact having drifted apart** |

> **The second gate exists because its absence cost two months.** The June F16 correction was applied to the `.cbor` and never swept back to the `.diag`. `v767-corpus-verify` reported **52 PASS / 0 FAIL** throughout — correctly, because the artifact *was* what it was expected to be. **Nothing asserted that the source still produced it**, so a corpus verifying green against itself carried a source that could no longer rebuild it. `-check` writes nothing and is safe to run against a tree its runner does not own.
>
> **`-check` deliberately does not recommend rebuilding on failure, and that is the correct behaviour.** When source and artifact disagree, **which side is right is a judgement, not a default** — in the F16 case the `.cbor` was the correct side, so a blind rebuild would have destroyed the good copy and silently re-introduced the bad widths. **Deciding which side is right is exactly the step the June regen skipped.**

### §4.3 The legacy encoder proof MUST be pinned to a frozen source `[MUST]` `[RULED 2026-08-13]`

`-verify-legacy` proves the encoder still reproduces a known-good historical artifact. Its input pair — **the `.diag` frozen at `56d4de4` plus the six §4-width corrections, reproducing `8e7c5232…` at 9236 B** — **MUST NOT be re-pinned to a moving source.** Once the live `.diag` legitimately moves (as it did under the M3/M6 re-stamp), re-encoding it cannot reproduce the June artifact, and re-pointing the proof at HEAD would make it assert only that today's source produces today's output — **a tautology wearing the costume of an encoder proof.**

### §4.4 Sequence for any corpus change `[MUST]`

1. **Architecture** edits the `.diag` **fields** — both copies, encoded content identical (§5 de-versioning rule).
2. **Build owner** (Go, §4) rebuilds both `.cbor` with byte-identity enforced **before** anything is written.
3. **`-check` green** — source produces artifact, both copies identical.
4. **`v767-corpus-verify` green** — artifact decodes and re-derives.
5. **Architecture commits the artifacts** into `entity-core-protocol`. The build owner writes the files; it does not run git in a tree it does not own.

**Never hand-edit a `.cbor`. Never transcribe a byte pin by hand. Never re-pin the legacy proof.**

---

## §5 Round-trip workflow (what closes this cycle)

| Step | Owner | Output |
|---|---|---|
| 1 | Architecture (this doc) | `SEEDS.md` ratified + `conformance-vectors-v1.diag` with Phase 1 byte pins inline + Phase 2 vector definitions referencing §2 seeds + Phase 2 byte pins marked `TBD-COHORT-ROUND-TRIP` |
| 2 | **Go** | Re-run M2/M3/M6 with §2.1 keypair seeds substituted for ephemeral keys + capture §2.6 tuples + ship as `V767-PHASE2-BYTE-PINS-COHORT-2026-MM-DD.md`. **Also**: produce `conformance-vectors-v1.cbor` from the `.diag` using the Go ECF encoder. |
| 3 | **Rust + Python** | Regression-confirm Phase 1 byte-equal (already validated 2026-06-09; one-line ✅) + run Phase 2 with §2.1 seeds + confirm §2.6 tuples byte-equal Go's pin + confirm `.cbor` byte-equal. |
| 4 | Architecture | Fold the Phase-2 byte pins into `conformance-vectors-v1.diag` from Go's note. Cohort regression-confirms one more time. Corpus lockable. |
| 5 | **Keystone (C#)** | Vendor `agility-vectors-v1` = `v767/` + `GUIDE-CONFORMANCE` §9 `AUTHZ-*` matrix; run resync work block; re-run S4. |

> **M3/M6 re-stamp status `[2026-08-12]` — the round-trip re-opens at step 3, and it is not blocked on architecture.** §4.5a item 1a (v7.77) moved `system/peer` to the floor **after** the M3/M6 rows were stamped, so the six SHA-384-row expectations went stale (`entity-core-go`, `cmd/v767-corpus-verify -full-hashes` → `49 PASS, 6 FAIL`, read live at `419a715`; proposed replacements in their `spec-issues/2026-08-11-d-v767-m3-m6-restamp-proposal.md`). Go's proposal is **step 2, complete and correct in form** — derived values with a per-row reason, nothing hand-edited, and the root cause proven singular by what still passes (every `peer_b` SHA-256 assertion is green, because peer_b was already at the floor and 1a did not move it).
>
> **It was filed as "awaiting arch ratification"; that misreads this table.** Step 4 folds pins that step 3 has already confirmed byte-equal — **architecture does not ratify a single implementation's derivation, by design.** Go said so themselves (*"No cross-impl claim. These values were derived by Go only"*) and offered cohort confirmation as a gate rather than assuming it away. The gate is granted, and it is the standing rule here, not a new condition: a corpus is an oracle, and an oracle produced and checked by one implementation is that implementation's output with extra steps. Phase 1 is byte-pinned precisely because three impls produced it independently.
>
> **What actually blocked it was a stale build-state belief, not a missing ruling.** Go declined to request step 3 because they recorded rust and py as last-moved 2026-08-10 and unprobed. **Both have moved since** — `entity-core-rust` `21eb223` and `entity-core-py` `2c1aa1b`, both 2026-08-11, both pushed and clean (read live 2026-08-12). Step 3 is runnable now.
>
> **The re-stamp MUST cover both copies of this corpus, and the second copy is a standing hazard `[flagged 2026-08-12 — decision owed]`.** `specs/test-vectors/crypto-agility/` is the de-versioned **public release form** of this directory (`agility-SEEDS.md` / `agility-vectors-v1.{diag,cbor}`), cut at the v0.8.0 release (`cf3c436`, 2026-06-21) and **untouched since**, while `v767/` has kept moving. It carries the **same four stale `expected_peer_a_content_hash_sha384` entries** and the **same pre-1a §2.4 home-format pin** — verified 2026-08-12. Step 5 above has Keystone vendor `agility-vectors-v1`, so **the copy an external implementer consumes is the stale one**, and it announces itself as `Status: Active` and *"the SINGLE SOURCE OF TRUTH for the crypto-agility corpus seeds"* while this file makes the same claim for the same content. Two files, one content, both claiming canonicity, diverging silently — the drift `AGENTS-STANDARD`'s one-canonical-home rule exists to prevent, and go's verifier cannot see it because it pins the `v767/` path. **RESOLVED `[2026-08-12]` — re-stamp both together; there was no frozen-history dilemma.** This note first called it *"a call about frozen history."* **That framing was wrong, and two checkable facts retire it** — raised by `entity-core-go` and verified here:
>
> 1. **The corpora are byte-identical where it counts.** `conformance-vectors-v1.cbor` and `agility-vectors-v1.cbor` are the same bytes (`sha256 8e7c5232…e31f982e`, both files, 2026-08-12) — the sha the corpus pin already verifies. The `.diag` files differ only in **de-versioning cosmetics**: header dates, `V7 §` → `§`, and sibling filenames. **No vector value differs**, which is precisely why the encoded artifacts match. So **the divergence would be *created* by re-stamping one copy — it is not a pre-existing state that leaving them alone protects.**
> 2. **The published artifact is already immutable, and not by way of this file.** `v0.8.0` is a git tag and it contains `specs/test-vectors/crypto-agility/` in full. **The tag is the archival record.** Correcting the working tree does not rewrite anything published; it only changes what the *next* release ships — which is an ordinary fixture correction, not a history edit.
>
> **The dilemma was self-inflicted: "published" was conflated with "immutable in the working tree."** A tagged release is what makes an artifact frozen; a directory that a tag happens to contain is not. Both copies re-stamp under the same cohort round-trip (§5 steps 2–4), and the byte-identity above is the invariant to preserve — **if the two `.cbor` shas ever differ after this, that is the defect, and it is worth a check.** Making `crypto-agility/` a generated view of `v767/` remains the stronger long-term shape and is no longer urgent once both are corrected together.
>
> **Phase-1 input widths — the June F16 regen was never swept back to the `.diag` `[FIXED 2026-08-12]`.** Six literals in **each** copy were wrong: the **Ed448 secret seeds carried 58 bytes** where RFC 8032 `SeedSize` is **57** (`key-type-ed448.1.pubkey`, `.4.signature`, `matrix.M2`, `matrix.M6`), and the **experimental-test `public_key` carried 63 bytes** where v7.66 §4.2 pins **64** (`hash-format-sha-384.1.inherited_sha256_pin`, `.2.rehash`).
>
> **The `.cbor` was right and the `.diag` was wrong** — the inverse of the usual direction, and the reason it survived: F16 corrected the build artifact in June and nobody swept the source. Confirmed by measuring the artifact rather than re-deriving from the spec: longest byte-runs in **both** `.cbor` files are `0x42`→57, `0x46`→57, `0xAA`→64, against 58/58/63 in both `.diag`.
>
> **The file documented its own defect while carrying it.** This `.diag`'s build comment already read *"Supersedes prior sha `4d8dfced…` which carried 58-byte Ed448 seeds, a 63-byte experimental pubkey"* — describing, as superseded history, the exact state its own body was still in. **A note saying a defect was fixed is not evidence the file was fixed**; the only check that would have caught it is the one `entity-core-go` ran — measure the artifact, compare to the source. Worth a build-time assertion rather than a reader's diligence.
>
> **The de-versioning rule `[MUST]` `[RULED 2026-08-12]` — it applies to comments, never to encoded content.** `crypto-agility/` is the de-versioned publish form of this directory, and the two `.cbor` artifacts MUST be byte-identical. **So de-versioning may only touch what the encoder does not see**: `/ … /` comment blocks, the header, filenames, dates, repo paths. **An `"id"` or `"description"` is encoded, so a de-versioned variant of one is not cosmetic — it is a guaranteed `.cbor` divergence at the next rebuild.**
>
> **`v767/` is the source; the publish copy takes its encoded content verbatim.** Where the two disagree on an encoded field, `v767/` wins by definition rather than by review. On the fully-qualified-citation question specifically (`V7 §1.5` vs `§1.5`), **the qualified form is canonical in both** — a citation that does not name its spec is exactly what `AGENTS-STANDARD` says not to ship, and the published copy is the one that most needs to be self-contained for a reader with no other context.
>
> *Found by `entity-core-go` at rebuild-planning time: **5 of 13 shared descriptions had diverged** — two still carrying the pre-1a `granter.hash is SHA-384` claim this file swept, three by de-versioning alone. **Diverged sources would have produced two different `.cbor` files at the next rebuild**, which is why refusing to rebuild before reconciling was protecting a live invariant rather than a hypothetical one. All five reconciled to the `v767/` strings 2026-08-12, and verified afterwards that every remaining differing line between the two files sits inside a comment block.*
>
> **FOLDED `[2026-08-13]` — `hash-format-sha-384.2.rehash` is inverted in both `.diag` sources; the `.cbor` rebuild is owed by the build owner (§4.4 step 2).** The paragraph below is the record of why it was owed for four handoffs, kept because the shape recurs. What it describes is now done on the source side: `kind` is `construct_reject`, the pre-inversion `canonical_content_hash` pin is gone, and the vector carries an explicit `verifier_requirement` that the refusal be observed through the pinned constructor rather than a hand-built entity. **Until step 2 lands, source and artifact disagree by construction and `-check` is expected red — that is the sequence working, not a defect.** Item (a) of the same pass, the `peer_a_content_hash` field rename, landed earlier at `8d38e62`.
>
> *(Historical, as written when it was outstanding.)* **the `hash-format-sha-384.2.rehash` inversion is ruled and NOT folded.** The vector still carries its pre-inversion pin (`canonical_content_hash` `012e64bbde3c494cf7cd…`), i.e. it still asserts that a `system/peer` **can** be authored under `content_hash_format = 0x01` — the construction item 1a forbids. `v767-corpus-verify` names it on every run as a NOTE, which is the only reason it is not lost. **This is the third ruled-not-folded item in this corpus from the same author in one week**, and it is a `.diag` **field** change (the vector's `kind` and its expected assertion), not a byte change — so it is architecture's under §4.1 and sequences through §4.4 like any other: edit both sources, Go rebuilds, `-check`, verify, commit. **It is deliberately not bundled into the artifact-landing commit**, because folding it now would immediately invalidate the `.cbor` that was just proven consistent — the churn is the reason to sequence it, not a reason to defer it indefinitely.
>
> **Also owed in the same pass, ruled here:** (a) the field reverts to §2.6's `peer_a_content_hash` — the `_sha384` suffix was always a divergence from the ratified shape and under 1a names a knob that cannot exist; (b) `hash-format-sha-384.2.rehash` is **inverted, not retargeted** (see `HASH-FORMAT-SHA-384-1` in the definition source) — it currently asserts a `system/peer` under `content_hash_format = 0x01`, which 1a forbids, and stays green only because the verifier hand-builds the entity instead of going through the pinned constructor. **A vector that exercises a forbidden construction and passes by routing around the code that would forbid it certifies the opposite of the rule** — the `GUIDE-CONFORMANCE` §2.4a failure shape, in a fixture. Inverting it (authoring a `system/peer` under `0x01` MUST be refused) turns the vector into the guard for the rule that retired it; the verifier MUST go through the pinned constructor so the bypass cannot recur.

---

*Companion: `entity-core-go/docs/validation/V767-FIXTURE-SEED-RECONCILIATION-2026-06-10.md` (Go cohort hand to architecture; the source of every Phase-1 byte pin above and the §2 scheme). Architecture clarifications added in §2.2 (initiator + hash_formats list → active format), §2.4 (home-format identity reference vs active-format content), §2.5 (RFC 8032 deterministic signature target). v7.71 `AUTHZ-*` matrix lives in `GUIDE-CONFORMANCE.md` §9, NOT this file — codes are domain-scoped per V7 §3.3 + GUIDE-EXTENSION-DEVELOPMENT §4.X.*
