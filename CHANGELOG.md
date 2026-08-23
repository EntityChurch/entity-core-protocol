# Changelog

All notable changes to this project are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project aims to follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

- Initial public research-preview release.

### Changed — spec amendment 0.8.1 (keystone cross-substrate hardening, before-freeze)

Non-functional hardening + prose/status reconciliations surfaced by the `entity-core-keystone` cross-substrate
conformance sweep (findings F31–F48 + the RT-/W hand-offs). **No wire renumber, no new opcode, no V8 semantic
change** — the wire core stays locked ([ADR-0002]). A conformant peer passes unchanged; the value is gating the
next generated peer and pinning impls on a stale reading. Applied to `ENTITY-CORE-PROTOCOL.md` (§4.2, §4.4, §4.6,
§4.8, §5.2, §6.11, §6.7, §3.5, §3013) and `ENTITY-NATIVE-TYPE-SYSTEM.md` (§4.4 table, §10.1 / Appendix B refs):

- **RT-13a (§4.8):** store-safety explicitly includes reference counts / shared per-entity metadata (manual-memory use-after-free class).
- **RT-13b (§6.11 a′):** new MUST — whole-frame write atomicity on a shared connection (no byte interleave).
- **F32 (§4.2/§4.4):** missing/unverifiable `author` is auth-class **401**, not a blanket 403 (reconciled with the §5.2a discriminator).
- **UN-b/F48 (§4.6):** key_type-support validation hoisted to an ordered **step 0**, before identity binding → `400 unsupported_key_type`, not `401 identity_mismatch`.
- **RT-6 (§4.6):** connect-handshake nonce single-use / freshness elevated SHOULD→MUST (scoped to the interactive handshake).
- **F40 (§5.2):** scope matching is typed per-dimension (path-scope canonicalized, id-scope literal); the two MUST NOT be interchanged.
- **RT-14 (§3.5):** lowercase-hex in **any** tree path segment is a general MUST (path segments case-sensitive).
- **RT-8 (§6.7):** optional MAY — return `404` (not `403`) for authz-denied to mask handler existence (must be consistent).
- **UN-a/F47 (§3013):** disambiguate the policy-path `{peer_pattern}` (hex-closed) from the capability `peers:` scope patterns (Base58); keystone's "add Base58 to §3013" rejected.
- **F37 (type-system):** two dangling `system/peer.peer_id` refs repointed to `system/identity/peer-id`; the 14-title-vs-15-row bootstrap-table mismatch reconciled (bare `entity` un-numbered as the primordial co-arising root).

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
- **§5.9 (Ruling 2):** de-confound the magnitudes — `ttl` default 64 → **512** (= 8 × the `chain_depth` ceiling 64); equal magnitudes let TTL mask the deterministic depth brake. The **ratio 8×** is the pinned, cohort-retunable quantity.
- **§4.10(b) (Ruling 3):** disambiguate the reason codes — `chain_depth_exceeded` (400) is the **capability**-chain limit; the **continuation** causal-depth brake suspends with `bounds_exceeded` (429). Two mechanisms MUST NOT share a reason string (Rust/Py were colliding).

Governing record: `entity-system-architecture/docs/proposals/PROPOSAL-CONTINUATION-BOUNDS-PROPAGATION.md` (rev.2026-07-19). The CONTINUATION-extension-side reconciliations (§3.6 step-6 refill→decrement, §3.7 resume-roots-depth-0, §3.9 wired framing, §6.2 monotonic clause) fold in-place in `entity-system-architecture/specs/extensions/EXTENSION-CONTINUATION.md`.
