# Changelog

All notable changes to this project are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project aims to follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

*(Nothing yet. Post-release work lands here.)*

---

## [0.8.2] — 2026-08-22

> **Not yet tagged.** The version headers were cut on 2026-08-21; the published surface then moved
> again on 2026-08-22 (the corpus de-version below), so this section describes the tree at that
> later point and is dated accordingly. `v0.8.2` does not exist as a tag — a tag is the release
> ([ADR-0015]) — and whatever commit carries it is what these notes must match.

**The research-preview release.** `ENTITY-CORE-PROTOCOL.md` **`Version: 0.8.2`**; the encoding and
type-system companions ship at their own labels, **`ENTITY-CBOR-ENCODING.md` 1.5** and
**`ENTITY-NATIVE-TYPE-SYSTEM.md` 4.2.1**, which are unchanged in normative content since the last
snapshot and differ only by correction and cross-reference repair.

**The locked wire core is untouched** — no renumber, no new opcode, no new status code ([ADR-0002]).
Everything below is either a rule that was always implied and is now stated, or a correction to text
that contradicted a neighbouring section.

**Versioning note, so the sequence reads correctly.** Development after `0.8.0` ran under an
arch-managed fourth component (`0.8.0.1`) whose only job is to signal to the implementation cohort
that core text moved without an agent inventing a release number. **That component is stripped at
release, which is this.** Three normative rules in §5.2's dispatch pseudocode carry inline
`(normative, 0.8.2)` tags — those were written ahead of the version, not left behind by it, and they
are correct as of this release.

- Initial public research-preview release.

### Changed — the `system/peer` identity entity is pinned to the ECFv1-SHA-256 floor (v7.77)

**No wire renumber, no new opcode** — the locked wire core is untouched ([ADR-0002]). Routed by core-go
(with two green tests) after arch's `SPECIFICATION-FORMAT.md` §8.4.6 pinned `{peer_id_hex}` to the
floor while leaving the identity entity on its author's home format. **Those two halves are unsatisfiable off
the floor**: `{peer_id_hex}` is simultaneously a path segment and an identity-reference equality operand, and
§1.8 / §4.5a ruled the two roles in opposite directions. Pinning only the path segment produces **two
`content_hash`es for one identity** — the state §1.8 exists to prevent — and turns capability-grant
verification into `401` on any non-floor connection.

- **§4.5a item 1a (new, normative).** The `system/peer` entity a peer presents is authored under ECFv1-SHA-256
  unconditionally, whatever the connection's active format and whatever the peer's home format. It is the one
  wire/identity-surface entity with **no author-chosen content** — its data is wholly recoverable from the
  public peer-id — so every consumer derives its hash rather than fetching it.
- **§4.5a item 1** no longer lists the identity entity among entities following the active format; **item 4**
  now states that identity-equality holds *across* connections, not only within one, and that one derivation
  function is the conformant shape.
- **§1.8** — the authored identity hash and the floor-derived hash are now the same bytes by construction, so
  "MUST NOT recompute" stops being a coincidence preserved per-connection and becomes an identity. **The
  prohibition is unchanged and unweakened**; it simply can no longer contradict §8.4.6's derivation.
- **§1.2** — one named exception to "a peer's persistent state is uniformly its home format."

**What this deliberately does not do:** it does not make the floor mandatory for peers. A non-floor **home**
format stays fully runnable — §1.2 still governs all stored content, §4.5a still governs envelope framing,
capabilities and signatures. Exactly one entity type, of three public fields, is pinned. The alternative
(promote §8.4.6's "effectively mandatory" prose to a `[MUST]` on peers) was **rejected**: it retires
`hash_formats` negotiation as a live wire surface and makes any non-floor conformance arm illegal by
construction rather than merely divergent.

### Changed — the test-vector corpus is de-versioned; the crypto-agility corpus has one copy

`specs/test-vectors/v767/` is **deleted**. The crypto-agility corpus had two committed copies — a
working copy stamped for spec revision v7.67 and a de-versioned publish form — and they collapse
into `specs/test-vectors/crypto-agility/`. `agility-SEEDS.md` → `SEEDS.md`;
`agility-vectors-v1.{diag,cbor}` → `agility-vectors.{diag,cbor}`.

**No vector value changed and the artifact did not move** —
`b5484e84dd2cddfa7d3cc8a041deba92cb29615aedb2180e31d8b6910ac5b648`, 10874 B, before and after. A
corpus is now identified by **its name and its artifact's sha256**, never by a version stamp; the
history that a filename integer pretended to carry lives in `crypto-agility/CHANGELOG.md`, added
here. The corpus-process rules that had accumulated in the working copy's `SEEDS.md` land as
`GUIDE-CONFORMANCE` §5.1a–§5.1d, where procedure belongs.

The second copy was the drift surface, not protection against it: the two copies diverged four
separate ways in one week. What replaces the `.cbor` byte-identity invariant is the
source-produces-artifact gate, which is the check that actually catches drift.

**`test-vectors/ecf-conformance/conformance-vectors-v1.*` deliberately keeps its `-v1` stamp** in
this release. Appendix E of `ENTITY-CBOR-ENCODING.md` states the corpus-version citation rule
normatively and independently, so retiring the stamp there is a normative core-spec edit that no
proposal yet covers. Tracked; it does not ship half-done.

### Fixed — the §3.9 type registry carried two superseded type strings

`ENTITY-CORE-MACHINE-SPEC.md` §3.9 still defined `system/protocol/inbox/{delivery,notification}` after both
renames were ratified 2026-08-10 in `EXTENSION-INBOX.md` §2.1 / `EXTENSION-SUBSCRIPTION.md` §2.2. Renamed, and
the block is now marked as a **reproduction** with its canonical home named — nothing gates spec-to-spec, so
this surfaced by reading, in core-go, and would have kept not-failing.

### Changed — spec amendment 0.8.1 (keystone cross-substrate hardening, before-freeze)

Surfaced by the `entity-core-keystone` cross-substrate conformance sweep (findings F31–F48 + the RT-/W hand-offs).
**No wire renumber, no new opcode, no V8 semantic change** — the wire core stays locked ([ADR-0002]). Applied to
`ENTITY-CORE-PROTOCOL.md` (§4.2, §4.4, §4.6, §4.8, §5.2, §6.11, §6.7, §3.5, §3013) and `ENTITY-NATIVE-TYPE-SYSTEM.md`
(§4.4 table, §10.1 / Appendix B refs). **Per keystone's 2026-07-27 review, these split by validation requirement**
(HANDOFF-TO-ARCH-2026-07-27-0.8.1-ratify-preconditions):

**Bucket A — spec repair (a conformant peer passes unchanged; the spec contradicted itself, peers were already correct):**

- **F32 (§4.2/§4.4):** missing/unverifiable `author` is auth-class **401**, not a blanket 403 (reconciled with the §5.2a discriminator). Surfaced by Nim/Julia as a spec self-contradiction — peers already mapped `AUTHN_FAIL`→401.
- **UN-b/F48 (§4.6):** key_type-support validation hoisted to an ordered **step 0**, before identity binding → `400 unsupported_key_type`. Existing vectors (`AGILITY-UNKNOWN-1`, `NEGOTIATE-KEYTYPE-1`) already gate the ordering; `unsupported_key_type` present in 38/41 peers.
- **UN-a/F47 (§3013):** disambiguate the policy-path `{peer_pattern}` (hex-closed) from the capability `peers:` scope patterns (Base58); keystone's "add Base58 to §3013" rejected.
- **RT-8 (§6.7):** optional MAY — return `404` (not `403`) for authz-denied to mask handler existence (must be consistent).

**Bucket B — new requirements (these change what a conformant peer MUST do; need cohort vectors — the ratify gate cannot certify them until vectors exist):**

- **F40 (§5.2):** scope matching is typed per-dimension (path-scope canonicalized, id-scope literal); the two MUST NOT be interchanged. **The cohort implements the semantics 0.8.1 now forbids** (id-scope dims routed through canonicalization, confirmed structurally in Go + Python) — symmetric-latent today, a live cross-peer ALLOW divergence where the two sides canonicalize asymmetrically. **Needs an accept-path vector.**
- **RT-13b (§6.11 a′):** new MUST — whole-frame write atomicity on a shared connection (no byte interleave). Unverified cohort-wide; a reachable class (Io, `A-IO-002`).
- **RT-6 (§4.6):** connect-handshake nonce single-use / freshness elevated SHOULD→MUST (scoped to the interactive handshake). Behavioral; compliance unverified cohort-wide.
- **RT-13a (§4.8):** store-safety explicitly includes reference counts / shared per-entity metadata. Behavioral **only on manual-memory substrates** (use-after-free class; C fixed, rest unverified); inert on GC/ARC peers.
- **RT-14 (§3.5):** lowercase-hex in **any** tree path segment is a general MUST (path segments case-sensitive). Generalizes a rule that previously bound only chain-participating caps; needs-check cohort-wide.

**F37 (type-system) — RESOLVED 2026-07-27 to the core name `system/peer-id` (spec repair; a conformant peer passes
unchanged).** F37's first pass (the cross-substrate hardening fold) had repointed the two dangling `system/peer.peer_id` refs to
`system/identity/peer-id`, matching the type system's own (mis-placed) definition — but `system/identity/` is the
**EXTENSION-IDENTITY** namespace, and `peer-id` is a **core** protocol primitive (bootstrap type #14, used in the
connect handshake before any extension loads). Placing a core type under an extension's namespace was the error;
the deferred "identity-namespace review" (`STYLE-NAMING-CONVENTIONS.md`) is now resolved. The type system is
reconciled to `system/peer-id` across all 19 sites (definition §4.8, bootstrap table, address-primitive lists,
type_refs); `ENTITY-CORE-PROTOCOL.md`, the cohort (36 peers), and the oracle core-type floor **already used the
core name** — so **no oracle change, no core-gate fingerprint move, no peer regen** (the F32/F48 pattern: the
cohort had converged correctly and the *spec* needed the fix). The 14-title-vs-15-row bootstrap-table mismatch stays
reconciled (bare `entity` un-numbered as the primordial co-arising root).

**Held / deferred:** **W6** (mint-time resource absolutization, §5.5/§5.5a) is a behavioral change gated on its own
§PR-8 conformance demonstration — authored, not folded. Appendix-B systemic regen (F37 `[K]`) and the Group-E
editorial one-liners (F36, format_code=128, F45, F33, F46, RT-10) are tracked follow-ons. Ratifies on the 28-peer
cohort re-run. Determination: `entity-system-architecture/docs/research/reviews/ABSORPTION-keystone-findings-F31-F46-and-named-handoffs.md`.

### Added — non-interactive freshness knobs, W7 (0.8.1 — same cohort re-run)

Additive, deployment-elected freshness knobs — the §4.10 "declare and enforce a finite bound; the value is the
deployment's call" pattern applied to freshness. No wire renumber, no new opcode. Applied to `ENTITY-CORE-PROTOCOL.md`:

- **Knob 2 (§2881 / §5.1):** a deployment-declared, honored `revocation_propagation_bound` — makes the revocation exposure window `min(TTL_granularity, B)` reason-about-able (ratifies the "sync-latency-bounded" convention).
- **Knob 3 (§2966 / §5.10):** a declared cross-clock skew-tolerance `δ` — the relayed validity window is `[not_before − δ, expires_at + δ]`, declared not silent (`δ = 0` reproduces today's behavior; a *tolerance*, not clock sync).
- **Knob 1 (§210, security consideration):** the informed-deferral note — a relayed cap is authentic + authorized but not proven fresh; replay is bounded by `min(TTL, revocation_bound) ± δ` and defended primarily by handler idempotency. The strong mechanism (challenge-response over a relayed circuit) is a **deferred future extension** (rides RELAY Mode C).

Design: `entity-system-architecture/docs/research/explorations/EXPLORATION-NON-INTERACTIVE-FRESHNESS-AND-ANTI-REPLAY.md` + `ANALYSIS-NON-INTERACTIVE-FRESHNESS-CRITICALITY.md`.

### Changed — cross-peer continuation bound (0.8.1; folds the built+green-3-way continuation/bounds work)

The continuation/network runtime is built and conformance-green three-way; these core-protocol deltas fold the
spec text up to the shipped wire. `chain_depth` is additive (MUST-ignore-unknown, beside `cascade_depth`) — no
wire renumber. Applied to `ENTITY-CORE-PROTOCOL.md`:

- **§3.11 `system/bounds`:** add the `chain_depth` field (all 3 impls ship it) — the deterministic causal-chain-length brake, inherited across the wire like `cascade_depth`, distinct from ttl/budget. Pin `chain_id` to a **single path segment** (was a UUID default with no format; marker path-safety depends on it).
- **§5.9 (Ruling 1):** pin cross-peer TTL — the resource backstop is decremented **once per dispatch (incl. sub-dispatches), never double-counted** at ingress *and* forward. Fixes the 9-vs-64 cross-impl divergence (Rust seeded no TTL; Python double-counted).
- **§5.9 (Ruling 2):** de-confound the magnitudes — the two MUST be **distinct** with `chain_depth` ceiling ≤ `ttl` seed (equal magnitudes let TTL mask the deterministic depth brake). **8× (seed 512) is the recommended default**, measured three-way 2026-07-27; a deployment MAY retune it for its fan-out. The conformance requirement is the *property* (the depth brake, not TTL, terminates a runaway), not the number — per the §4.10 doctrine.
- **§4.10(b) (Ruling 3):** disambiguate the reason codes — `chain_depth_exceeded` (400) is the **capability**-chain limit; the **continuation** causal-depth brake suspends with `bounds_exceeded` (429). Two mechanisms MUST NOT share a reason string (Rust/Py were colliding).

Governing record: `entity-system-architecture/docs/proposals/PROPOSAL-CONTINUATION-BOUNDS-PROPAGATION.md` (rev.2026-07-19). The CONTINUATION-extension-side reconciliations (§3.6 step-6 refill→decrement, §3.7 resume-roots-depth-0, §3.9 wired framing, §6.2 monotonic clause) fold in-place in `entity-system-architecture/specs/extensions/EXTENSION-CONTINUATION.md`.
