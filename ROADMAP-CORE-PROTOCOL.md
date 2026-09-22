# Core Protocol — Domain Roadmap (canonical, living)

**Status**: Active

**Target repo:** `entity-core-protocol`.
**What this domain is:** the irreducible protocol every peer must speak — the wire format,
the type system, the identity/capability/dispatch core, and the core tree operations. This is
the **mandatory convergence layer** (`--profile core`): agree on this and any two peers
interoperate at the substrate.

> **Canonical & living.** This roadmap is updated as the protocol moves. It is the
> **forward-looking, domain-scoped view**: the maturity ladder below and where each artifact
> currently sits on it.

**Protocol line:** V8 (Genesis tag `v0.8.0`) · **Floor:** keystone **15 generated peers,
`--profile core` 0-FAIL** + Go/Rust/Python release-green.

> **Versions are not restated here.** Each specification's own `**Version**:` header is
> authoritative for that document, and the three move on independent ladders. The table below
> records *maturity and trajectory*, which are this document's to state; for the number, open the
> specification. A number copied out of a document has nobody to keep it current, and this table's
> copies were a full release cycle stale before anyone checked them.

---

## The maturity ladder (recap)

`M0 Proposed → M1 Ratified → M2 Landed → M3 Implemented → M4 Peer-converged (3-way) →
M5 Validated (vectors) → M6 Keystone-converged (15 generated peers)`. The core domain's
defining property: **everything here is at the strongest gate, M6** — independently
re-derived by 15 languages, not just the three reference impls.

## Artifacts & maturity

| spec | version line | maturity | trajectory | role |
|---|---|---|---|---|
| `ENTITY-CORE-PROTOCOL` | `0.x` — carries the release number | **M6** | 🟢 floor stable | the spec — Layers 0–4; the universal-tree model (§1.4/§1.7) |
| `ENTITY-CBOR-ENCODING` | `1.x` — independent | **M6** | 🟢 byte-stable v7.56→v7.77 | ECF — the wire contract (App-E conformance corpus) |
| `ENTITY-NATIVE-TYPE-SYSTEM` | `4.x` — independent | **M6** | 🟢 | core type system + native types |
| `EXTENSION-TREE` | `4.x` — not in this repo | **M6** | 🟢 | **tree get/put = core** (§2); snapshot/diff/merge/extract/view ride along |
| core `test-vectors/` | pinned by sha256, not numbered | **M6** | 🟢 | ECF + crypto-agility corpora (vendored to keystone) |

~~`ENTITY-CORE-MACHINE-SPEC`~~ — **RETIRED 2026-08-31** and no longer published. **G3 is closed as
*retire*, not *make current*.** Its §1.8 is now `ENTITY-CBOR-ENCODING` §5.4 and its §6.4 was always
`ENTITY-CORE-PROTOCOL` §4.7; `.release-removals` carries the same note.

## Stages / phases

**Stage — Substrate (DONE, M6).** Wire/type/identity/capability/dispatch/tree all converged
across 15 generated peers + the 3 reference impls at 0-FAIL. This is the v1 floor and it is
**closed**: the v7.65→v7.77 amendment line (peer-entity canonicalization, crypto-agility seed
tables, authorization-error-code contract, concurrency gate, resilience + resource-bounds floor,
verdict-determinism errata) all landed and re-converged. The protocol is feature-complete for v1.

**Stage — Release normalization (IN FLIGHT).**
- **Naming normalization** — folded on the V7 line as v7.77 (kebab namespaces / snake keys).
- **v7 → V8 cutover** — Genesis `v0.8.0` is the release marker; behaviorally a near-no-op
  (the naming work is already folded). *Forward item:* author the v7→v8 conversion doc (gap #7).
- **MACHINE-SPEC — RESOLVED 2026-08-31: retired** (gap G3 closed). The skew was never going to be
  paid down: a hand-maintained condensation of a specification ages on every edit to the source, and
  **nothing gates spec-against-spec**, so the drift is only reachable by a human reading both
  documents side by side. Retirement had been decided **2026-08-02** and its one precondition —
  relocating §1.8 to `ENTITY-CBOR-ENCODING` §5.4 — was met **2026-08-10**; the retirement itself then
  sat undone for four weeks and cost a fold (FM-1 found §6.4 and §6.2 both wrong). It is no longer
  published; `.release-removals` says where its two cited sections went.

**Stage — Hardening program (forward, post-release, proposal-first).** The substrate is
correctness-complete; the open frontier is non-functional (resilience, security-under-load,
per-language performance shape, distributed resilience). The P0 resilience floor + P1 resource
bounds landed (v7.75); P2 (perf shape) + P3 (distributed) are deferred, proposal-first.

## What is NOT core (boundary note)

The 22 other extensions, the SDK, applications, and all guides are **not** in this domain — they
are the optional "agree → interoperate" layer in `entity-system-architecture`. The only extension
document here is TREE, because it defines the core get/put surface. See the canonical map §1/§3.

## Notes for the content/papers project

- The headline story: **core is small, frozen, and independently re-derived by 15 languages.**
  That 15-peer `--profile core` 0-FAIL gate is the credibility anchor — use it.
- "Crypto agility" lives here as a *validated* surface (Ed448/SHA-384 seed tables), but the
  dual-algorithm *requirement* is a keystone generator-stability requirement, not a core floor
  raise — don't describe SHA-384 as mandatory for v1 conformance.
