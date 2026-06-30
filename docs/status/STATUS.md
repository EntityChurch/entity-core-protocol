# entity-core-protocol — status

_Updated: 2026-06-30 · public: v0.8.0 (master)_

## Where it is

entity-core-protocol is the **canonical specification** of the Entity Core Protocol —
the spec text itself, not an implementation. It defines the **mandatory convergence
layer** (`--profile core`): the Entity Canonical Form (ECF) wire format, the
entity-native type system, the identity / capability / dispatch core, and the core
tree `get`/`put` operations. Two peers that agree on this surface interoperate at the
substrate; everything above it (the other extensions, the SDK, applications) is
optional and is published in the sibling `entity-system-architecture` repo.

Maturity is high. The v1 substrate is **feature-complete, frozen, and closed**: wire /
type / identity / capability / dispatch / tree have all converged. The published
release line is **V8, Genesis `v0.8.0`** (tag `v0.8.0` on `master`). The README's
convergence claim — the core profile is independently re-derived by a cohort of
**15 generated peers at `--profile core` 0-FAIL**, plus the Go / Rust / Python
reference implementations — is the credibility anchor; state it as cohort + reference
convergence, not as 15 fully-independent derivations.

The normative texts live in `specs/`, each carrying its own authoritative
`**Version**:` header (which is the source of truth for that document, not the release
tag):

| `specs/` file | version | role |
|---|---|---|
| `ENTITY-CORE-PROTOCOL.md` | 0.8.0 | the protocol — Layers 0–4; the universal-address-space / tree model |
| `ENTITY-CBOR-ENCODING.md` | 1.5 | the Entity Canonical Form (ECF) wire contract |
| `ENTITY-NATIVE-TYPE-SYSTEM.md` | 4.2.1 | the core type system and native types |
| `ENTITY-CORE-MACHINE-SPEC.md` | 0.8.0 | condensed, machine-readable implementation reference (no rationale, no history) |
| `SPECIFICATION-FORMAT.md` | 1.1 | the normative-spec authoring format |
| `STYLE-NAMING-CONVENTIONS.md` | 1.0 | identifier naming — these names are part of the wire contract |

The real contract surface — the conformance corpora — lives under
`specs/test-vectors/`: `ecf-conformance/` (the ECF byte contract,
`conformance-vectors-v1.{diag,cbor}`) and `crypto-agility/` (the cross-key /
cross-hash matrix, `agility-vectors-v1.{diag,cbor}` with the seed convention in
`agility-SEEDS.md` and `README.md`). `.diag` is the human-readable source; the sibling
`.cbor` is the deterministic ECF-canonical encoding; bytes MUST match. These corpora
are vendored downstream and are treated as pinned.

`CHANGELOG.md` frames `v0.8.0` as the **initial public research-preview release**.

## Where we left off

Stable at the v0.8.0 "Genesis" research-preview line; no normative spec text has changed.
Next substantive work is reconciling `ROADMAP-CORE-PROTOCOL.md` to the published `0.8.0`
headers and making the `ENTITY-CORE-MACHINE-SPEC.md` keep-or-retire decision.

One framing correction vs. the older roadmap text: **the published specs are all
version-string-consistent at `0.8.0`** (including `ENTITY-CORE-MACHINE-SPEC.md`, whose
header reads `0.8.0`). The "MACHINE-SPEC reads 7.8 against the protocol line" *version
skew* that earlier notes describe **no longer applies to the shipped artifacts** — it
was resolved when the V8 publish normalized every version string. What remains is (a)
editorial drift in `ROADMAP-CORE-PROTOCOL.md`, whose artifact table still shows V7-era
strings (`ENTITY-CORE-MACHINE-SPEC` at "7.8", "byte-stable v7.56→v7.77", and an
`EXTENSION-TREE` row), and (b) the substantive maturity question of whether to keep the
condensed machine reference current with the protocol or retire it.

## Backlog

- **`ROADMAP-CORE-PROTOCOL.md` reconciliation** — bring its artifact table in line with
  the published `0.8.0` headers: drop the stale "7.8 / version skew" and "v7.56→v7.77"
  strings, and clarify that `EXTENSION-TREE` (listed as a core artifact) is referenced
  here but its full document is published downstream in `entity-system-architecture`.
- **`ENTITY-CORE-MACHINE-SPEC.md` keep-current-or-retire** — it is the condensed,
  generate-a-peer reference that "travels with" the protocol. Decide whether to maintain
  it against the protocol line going forward or fold/retire it. (Distinct from the now-
  resolved version-string skew above.)
- **v7 → v8 conversion note** — still owed. The cutover is behaviorally a near-no-op
  (the identifier-naming normalization that closed the V7 line is already folded), so
  this is a short migration note, not a behavioral delta.
- **Hardening program** (post-release, proposal-first, non-functional only): the P0
  resilience floor (responsive / bounded / deliver-or-signal / no-crash / recover) and
  the P1 resource-bounds floor (max payload, max chain depth, connection bound with safe
  defaults) have landed and re-converged. P2 (per-language performance shape) and P3
  (distributed resilience — delivery semantics, partition/restart catch-up) remain
  deferred.

## Waiting on

- **Operator decision** on the MACHINE-SPEC keep-or-retire question, and on how to
  reconcile the roadmap artifact table.
- **Cross-domain pointers** authored in `entity-system-architecture`, not here — the
  release-surface / maturity map, the full `EXTENSION-TREE` document, and the
  conformance guide (`--profile core` definition, profiles, vector index). They are
  referenced from this repo but owned downstream, so updates to them are out of this
  repo's hands.
- **Process gate, by design:** any normative change is **proposal-first** — the wire
  core is locked and closed for v1, so new substrate work goes through the RFC-style
  proposal in `CONTRIBUTING.md` (motivation → bounded discussion → **conformance-corpus
  demonstration** → ratify and fold), not a direct edit. Wording-only / editorial
  hygiene can go direct.

## Done recently

- Shipped the **initial public research-preview release `v0.8.0`** (V8 line): the full
  six-document normative spec set plus the ECF and crypto-agility conformance corpora.
- Closed and folded the entire **V7 amendment line**, re-converging the cohort after
  each: peer-entity canonicalization (identity is a pure function of `(public_key,
  key_type)`); crypto-agility seed tables (Ed25519/Ed448 × SHA-256/SHA-384, validated;
  PQ slots allocated, deferred); the home-format vs connection-active-format split for
  cross-format identity; the authorization-error-code contract (no catch-all on authz
  DENY); confused-deputy / foreign-granter authorization hardening; the core
  extensibility-boundary and peer-authority-bootstrap rules; the concurrency gate; and
  the P0 resilience + P1 resource-bounds floor. Then the identifier-naming normalization
  closed the V7 line, and the **V8 cutover** marked it as `v0.8.0`.
- **Publish hygiene:** normalized all spec version strings to `0.8.0`, de-versioned the
  spec bodies, stripped stray dates and unpublished-doc references, and replaced
  placeholder examples with neutral ones.

## Next

1. Reconcile `ROADMAP-CORE-PROTOCOL.md` to the published `0.8.0` headers and make the
   MACHINE-SPEC keep-or-retire decision.
2. Author the short v7 → v8 conversion note.
