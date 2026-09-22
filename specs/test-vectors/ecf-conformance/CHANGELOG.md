# ECF Conformance Corpus — Changelog

**This file is the corpus's version.** The directory is named for what the corpus tests and the
artifacts carry no version stamp; what changed, when, why, and who re-blessed it lives here instead.
An integer in a filename says only that something moved — and in practice it did not even say that,
because it was never once incremented while the corpus grew from 69 vectors to 71.

A conformance citation names `(spec-version, corpus-name, artifact sha256)`. The sha is the exact
identifier; this file is the narrative behind it.

**Vectors are never removed.** A landed vector stays a conformance criterion. A vector that is wrong
is corrected in place and the correction is recorded here.

---

## 2026-09-16 — ⛔ `signature.1`–`signature.3` are KNOWN-DIVERGENT and the correction is OWED, not landed

**The artifact has NOT moved. `conformance-vectors.cbor` is still
`9695b1f1d939cfdfdd4297f8ad32122d424b1ec180cfae74c92d509d88f7c6dc`, and this entry exists so that
nobody reads that digest as still-blessed on the `signature` category.**

`ENTITY-CORE-PROTOCOL` **0.8.2.26** ruled what a signature signs: **the target entity's full
`content_hash`** — `varint(format_code) ‖ SHA256(ECF({type, data}))`, 33 bytes under ECFv1-SHA-256 —
and §7.3 is its single normative home. `ENTITY-CBOR-ENCODING` Appendix E's `signature` category was
corrected in the same revision.

**The three landed vectors sign `ECF({type, data})` instead** — the message one SHA-256 and one prefix
short of the ruled one. They were generated to match the category's prose, which was wrong.

⭐ **The artifact contains its own correct input, one category above.** `content_hash.1`'s `canonical`
is `h'005f3139…'` — the 33-byte value §7.3 names as the message — and the `signature` category three
rows down does not use it. Nothing in the fixture related the two categories, which is why a
construction defect sat inside one artifact unreadable from either side.

**How it surfaced:** `entity-system-conformance` executed the category — the first time any Class-B
category had been executed rather than read — and reproduced `signature.1–3` over the ECF bytes with a
decoder and an RFC 8032 Ed25519 of their own writing, while the §7.3 reading completes the live
handshake on 8 of 8 peers and `entity-core-go` refuses the fixture's reading `401
authentication_failed`. Two independent codecs, both directions.

### What is owed, and by whom

| # | item | owner |
|---|---|---|
| 1 | Recompute `signature.1`, `.2`, `.3` — **`input` maps unchanged**, only `canonical` moves | a seat that runs code; `entity-core-go` as core lead |
| 2 | Add **`signature.4`**, signing an entity that also appears in the `content_hash` category, so the two cross-check by inspection (Appendix E now requires this) | same |
| 3 | Update the `.diag` header comment for the category — it states the old construction informally | same, in the same commit as 1–2 |
| 4 | **Cross-bless with a second independent codec** before the digest is published | `entity-system-conformance` s1-py is the natural second: independent decoder, Ed25519 from the RFC, and it is the instrument that found this |
| 5 | Record the correction here with the new sha256 | whoever lands 1–3 |
| 6 | Re-pin every published conformance report citing the superseded digest | each seat |

**Arch rules the text and does not author the signatures** (`GUIDE-EXTENSION-DEVELOPMENT` §7 Stage 4).
Owning this repo changes who edits it, never how.

⚠ **Until item 5 lands, a conformance report citing this corpus SHOULD state that the `signature`
category is under correction.** The other 68 vectors are unaffected — the defect is in the Class-B
`sign` construction and reaches nothing else.

---

## 2026-08-31 — de-versioned

**No vector value changed and the normative artifact did not move.**

| Artifact | sha256 | Note |
|---|---|---|
| `conformance-vectors.cbor` | `9695b1f1d939cfdfdd4297f8ad32122d424b1ec180cfae74c92d509d88f7c6dc` | **Unchanged.** Measured on both sides of the rename; identical. **This is the number to cite** |
| `conformance-vectors.diag` | `da521d67aa8193a3bf9acd232088d8515d50333f0294c65a5df3b45f46a1a87b` | was `71015b729b205f39e29750e632a136844fe7da3f9e37800e870d14bf87086544` — **two comment lines**, see below |

**71 vectors, unchanged.**

The `.diag` moved because two lines of its header comment named the old filename. Both edits sit
inside the file's opening `/ … /` span; the complete diff is those two lines, no `id`, `kind`,
`input`, `description` or `h'…'` value moved, and the `.cbor` digest confirms the encoding did not
change. **The `.diag` is non-normative source; the `.cbor` is the artifact impls load and the one a
conformance report cites.**

- `conformance-vectors-v1.{cbor,diag}` → `conformance-vectors.{cbor,diag}`.
- `ENTITY-CBOR-ENCODING` Appendix E amended: the citation rule no longer names a corpus version.
  §E.1's fixture reference, §E.3's load step, and §E.5's report rule now cite the corpus by name and
  the artifact by sha256.

**Why this half landed nine days after the crypto-agility half.** The de-versioning proposal scoped
the rule change to `GUIDE-CONFORMANCE` §5.1 and did not notice that **`ENTITY-CBOR-ENCODING`
Appendix E states the same rule independently, normatively, in a different repo's core spec.**
Renaming the artifact while §E still required *"cite the version of `conformance-vectors-v{N}.cbor`"*
would have left a live core `[MUST]` demanding a form that no longer existed. The ECF half waited for
the amendment that covers §E; the crypto-agility half, which no core spec named, did not have to.

That is the whole reason this corpus stayed stamped for nine days while `spec corpus` reported the
error, and it is recorded here rather than in the spec because a spec is not a log.

## Vector history prior to de-versioning

| When | Change |
|---|---|
| 2026-07-12 | **69 → 71.** `nested.5` and `nested.6` added (F29/F30) — CBOR head-length boundaries, on the path of every encode. The filename did not move, which is the concrete case against version stamps |
| — | Earlier history predates the byte-pin and is not reconstructible from artifacts; treat the 2026-07-12 sha as the first firm anchor |

**Consumers that vendor this corpus** must re-vendor against the new name. A vendored copy under the
old filename does not merely go stale — it makes the vendor gate report `vendor-unmatched`, which is
*could-not-look*, not a pass. Verify by sha256, never by filename.
