# Entity System — Identifier Naming Conventions

**Version**: 1.0
**Status**: Active
**Companion to**: SPECIFICATION-FORMAT.md (binding companion — identifier conformance)
**Enforced by**: `tools/spec-style` (corpus is gate-clean as of the fold)

---

## 1. Purpose

`SPECIFICATION-FORMAT.md` governs how a normative spec *document* is laid out — section ordering, RFC 2119 keywords, type-definition notation. This document governs the **identifiers inside** those specs: entity types, fields, operations, error codes, and value vocabularies that an implementation must reproduce byte-for-byte to interoperate.

These names are part of the wire contract. Once the protocol is released and downloaded, renaming any of them is a breaking change every implementation must absorb. This document exists so those names are **predictable, derivable, and consistent**, and so the linter (`tools/spec-style`) can mechanically verify conformance and reject unformatted contributions before review.

---

## 2. The Governing Rule

The protocol uses **two separators, chosen by what kind of token it is — never by which domain or spec you are in.**

> **kebab-case for the *namespace* (type paths, operations, enum/string values).**
> **snake_case for *data-structure keys* (field / map-key names, error & status codes).**

This split is **domain-invariant**: a field key is snake in *every* spec; a type path is kebab in *every* spec. The author never has to remember a per-context exception — the answer depends only on the locally-visible kind of token. In `{"peer_id": "system/peer/grant-entry"}` the key (left of the colon) is snake; the namespace value is kebab. You can *see* which you are writing; you never have to *recall* it.

**Why this split, and not one separator everywhere.** A single separator is superficially simpler, but the established corpus already follows this split with very high consistency, and unifying would not be free: an audit (`tools/spec-style`, 2026-06) found that forcing kebab everywhere would rename **473 distinct field keys (3,314 occurrences) plus ~174 value / error-code tokens** (`type_ref`, `peer_id`, `content_hash`, `array_of`, `capability_denied`, …) — together ~620 distinct identifiers across ~3,600 occurrences, a total break of the wire data-contract across all implementations, the CBOR corpus, and every conformance vector. Field keys are CBOR **map keys**: canonical CBOR orders map keys by encoded bytes, so renaming one re-orders the map, changes every enclosing `content_hash`, and invalidates every signature and test vector in the corpus. All-kebab is not a string-replace; it is a re-genesis of the entire cryptographic conformance corpus, for zero interop gain (the names are arbitrary identifiers — they could be UUIDs). The split moves only type-*path* values and touches **zero map keys**. Ratifying the existing split instead leaves the wire contract untouched and reduces the whole core+extensions cleanup to a handful of true outliers. The coherence win comes from *writing the rule down and gating it mechanically*, not from churning identifiers. The data is the rationale: we ratify the pattern that is already established and already correct in the overwhelming majority of cases.

The split is **configurable** in the linter (per-axis policy, see §5) so the standard can be retuned by a future proposal without rewriting tooling.

---

## 3. Per-Axis Specification

| Axis | Form | Example | Notes |
|------|------|---------|-------|
| Entity-type path segment | **kebab**, `/` for hierarchy | `system/capability/grant-entry` | gated (high precision) |
| Operation name | **kebab** (lowercase verb; hyphen if multi-word) | `get`, `put`, `read-metadata` | namespace vocabulary |
| Enum / string-value vocabulary | **kebab** | `target-wins`, `no-overwrite`, `k-of-n` | namespace vocabulary |
| Field / map-key name | **snake_case** | `peer_id`, `content_hash`, `request_id` | data-structure key |
| Error / status code | **snake_case** | `capability_denied`, `payload_too_large` | matched as a stable token like a key |
| Wire message-type constant | **SCREAMING_SNAKE** | `HELLO`, `EXECUTE`, `CONTENT_GET` | observed convention; ratify in the V8 proposal |
| Pseudocode terminal outcome | **UPPERCASE** | `DENY`, `ALLOW`, `REJECT` | not a wire identifier (SPECIFICATION-FORMAT.md §3.3) |

A token that is a single word with no separator (`get`, `peer`, `hash`) is trivially conformant on every axis.

### 3.1 Not identifiers — exempt

The rule governs protocol identifiers only. The following are **not** governed and the linter does not flag them:

- **Prose** — ordinary English (the linter only reads code blocks and `code spans`).
- **RFC 2119 keywords** — `MUST`, `SHOULD`, `MAY` (uppercase per SPECIFICATION-FORMAT.md §4).
- **Encoding / hash prefixes** — `ecfv1-sha256:…`, `sha256`, `blake3` (defined by ENTITY-CBOR-ENCODING; the hyphen there is structural, not a kebab identifier).
- **Document filenames** — `SCREAMING-KEBAB-CASE.md`.
- **Section references** — `§3.2`, numbering.
- **Placeholders & example data** — type variables (`A`, `B`, `T`), hex/base64 blobs (`DEADBEEF`, `dGVzdG5vbmNl`), proper nouns (`FastCDC`). These are `other`-cased and never gated.
- **Pseudocode helper-function names** — e.g. `verify_request(...)`. Illustrative, not part of the wire contract; reported separately, never failed.

### 3.2 The constraint-kind double-duty case (worked ruling)

The hardest case the audit surfaced, ruled here because it is the difference between a 9-identifier rename and a 2-identifier one. A standard constraint kind appears in the spec **twice**, on **two different axes**:

```
{name: "system/type/constraint/min_length", fields: {min_length: {type_ref: "primitive/uint"}}}
       └──────────── type-PATH (namespace) ────────────┘        └─ data KEY ─┘
```

The type-path leaf `system/type/constraint/min_length` is a **namespace address** (kebab axis). The parameter `min_length` is a **data-structure key** (snake axis). They are the *same word* but **not the same token** — one names where the constraint type lives in the tree, the other names a field inside the constraint's data.

**Ruling: apply the rule to each on its own axis. The path becomes kebab; the key stays snake.**

```
{name: "system/type/constraint/min-length", fields: {min_length: {type_ref: "primitive/uint"}}}
```

This is **not** an exemption and there is no conflict: the kebab-path / snake-key pair for one concept is the split working exactly as intended. The snake type-paths are themselves the *artifact* of this confusion — they diverged precisely because an author saw the leaf and the key as "the same name" and matched them, collapsing two axes into one. Ratifying the split *un-collapses* them. A reader who passes the linter is compliant; a reader who knows the rule reads `…/min-length` (a path) and `min_length` (a key) and knows which is which on sight. Affects the 7 `system/type/constraint/{min_length, max_length, min_count, max_count, one_of, not_one_of, type_pattern}` leaves; single-word kinds (`min`, `max`, `pattern`) are already conformant.

**Double-duty is exact for 5 of the 7, not all.** The five length/count kinds are genuinely double-duty — the kind word *is* the parameter key (`system/type/constraint/min_length` path leaf and `{min_length: …}` field key are the same word on two axes). `one_of` / `not_one_of` are **not** double-duty: their parameter key is `values`, not `one_of`, so the path rename stands alone with no snake-key counterpart to leave behind. The ruling is identical either way — **rename the path; leave any data key as-is** — but the worked example above (`min_length`) only illustrates the double-duty subset. Do not infer from it that an enum value or a `values` key should also change.

---

## 4. Semantic Naming (the "name follows what it does" principle)

Casing is mechanical; *which word* is chosen is editorial and non-binding:

- An operation name SHOULD describe the action it performs; a reader who knows the handler should be able to guess the operation, and vice versa.
- The protocol does **not** mandate a universal default-operation vocabulary. Domains MAY use different verbs (`get`/`put` for general tree-style access, `create`/`destroy` elsewhere). This is design-space, not a conformance axis, and the linter does not enforce it.

---

## 5. Enforcement

Conformance authority is mechanical, not editorial:

- `tools/spec-style` extracts every identifier from the spec corpus, classifies it by axis and separator, and reports every gated violation with `file:line`. Per-axis rules are config-driven (`--config`), so the standard can change by proposal without code changes.
- A spec is **gate-clean** when the linter reports zero gated violations. Exit code is non-zero on violations, so it composes as a CI / pre-submit gate.
- Contributors proposing a new extension spec or a core change MUST run the linter and submit gate-clean. Unformatted submissions are returned, not reviewed.
- Post-V8, the linter runs as a conformance gate so a non-conforming identifier cannot land.

### 5.1 Gate precision (current tool version)

The linter favors **precision over recall** so the gate never wrongly blocks a contributor. As of v0.2 it gates two high-confidence checks and surfaces the rest for human review:

- **Gated (high precision):** entity-type paths must be kebab (snake = violation); any identifier mixing `_` and `-` is a violation.
- **Report-only (review, not yet gated):** a kebab token used as a map-key cannot yet be distinguished from an operation / type / enum key (all legitimately kebab) without block-context parsing; and a snake *enum value* that should be kebab cannot yet be told from a snake *error code* that is correct. These detections are a v0.3 refinement (parse `fields:` / `operations:` / `one_of:` blocks for context). Until then they are surfaced for human audit, not auto-failed.

---

## 6. Status & open items

The **rule is normative** (the split above), validated against the corpus, and the corpus is **gate-clean** (`tools/spec style` reports 0 violations — the document defining the rule is itself exempt from the corpus walk via `EXCLUDE_FILES`, since it must quote the non-conforming "before" forms to teach the rule). Remaining items:

1. **SCREAMING_SNAKE wire-message-constant axis — RATIFIED** (§3). `HELLO` / `EXECUTE` / `CONTENT_GET` are an intentional, uniform convention; the linter may gate ALL_CAPS message constants rather than treat them as ungoverned. No rename either way.
2. **The adjudication list — DONE.** The audit's 9 true outliers (33 real occurrences) are renamed: the 7-member `system/type/constraint/*` snake family → kebab (ruled in §3.2; EXTENSION-TYPE v1.2); `system/peer_id` → `system/peer-id` (the lone core-spec straggler, a type-ref in SYSTEM-COMPOSITION v1.9); `system/content/frame_limit_respected` → kebab (EXTENSION-CONTENT v3.6 Amdt 4 + its validate-peer check label). 8 of 9 were in *extension* specs; none appeared in generated-peer source. Full impact + work-distribution recorded in the V8 naming proposal.
3. **v0.3 linter precision** — block-context field/operation/enum disambiguation to graduate the report-only checks into the gate (not blocking; all 9 current outliers are caught by the type-path gate today).

**Out of scope for this guide.** The *placement* of `system/peer-id` (vs `system/identity/peer-id`) is a namespace-model question about how the identity extension relates to core — not an identifier-style question. Casing is settled here (`peer_id` → `peer-id`); placement routes to a separate identity-namespace review and is not a style-guide concern.
