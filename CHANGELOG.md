# Changelog

All notable changes to this project are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project aims to follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Fixed — a capability/identity forgery in the envelope `included` map (0.8.2.23)

**Every authority lookup resolved an entity by a wire-supplied address that nothing verified.**
`envelope.included` is keyed by hash, and §5.2/§5.5/§5.5a resolve the author, the capability, the
chain root granter, each link's signer and each grantee **by key** out of it. An attacker who knows a
victim's identity hash — the **public `grantee` field of any capability the victim presents** — could
file their own `system/peer` entity under that key; their own signature then verified against their
own key while the peer attributed it to the victim. An observer of any capability chain could mint a
leaf off it, up to the parent's scope, without the grantee's key.

Self-consistency validation does not catch it, which is why it survived review: the substitute entity
hashes to its own content and is perfectly valid — it is wrong only *under the key it was filed at* —
and `content_hash` rides the wire, so the entity's own claim is attacker-controlled too.

**The obligation is stated as a property, not a mechanism.** §1.8 item 1 now carries *resolution
integrity* — never resolve an entity used for an authority decision through an address not verified
against its content — satisfied by **binding the key** or by **discarding the key and addressing by a
validated `content_hash`**. Both are conformant; the second is safe only where receipt validation runs
at every ingress, and that precondition is stated with it. §3.1 makes the map's keying normative in
both directions; it had stated the shape in the indicative for seven revisions and obliged nothing.

**Dispositions follow the resolution site, not the check** — three new §5.2a rows: author → `401
authentication_failed`, capability and chain granter/signer → `403 capability_denied`, grantee → the
existing `401 unresolvable_grantee` carve-out, and a decode-boundary refusal → `400 hash_mismatch`
without reaching the table. A uniform verdict would have made a key-discarding implementation
non-conformant for answering the row §5.2a already assigns it.

### Added — the handler frame is the handler that owns the operation (0.8.2.23)

§6.3's `handler_pattern` selects which grants are considered **before** their `resources` scope is
read, and the specification never said who supplies it. It is now `[MUST]`: **the handler that owns
the operation being authorized, never the handler performing the check** — generalizing a rule
`EXTENSION-SUBSCRIPTION` §2.3 already states in the imperative for a non-tree handler. Where owner and
runner coincide the frame is that handler's own pattern, so `system/tree` is not the only legal value.
`handler_pattern` is also **REQUIRED and fail-closed**: an absent value MUST NOT be read as *match all
handlers*. Both directions had been measured in shipped implementations — the wrong frame refuses a
conformant caller with no dimension to attribute the refusal to; the absent frame let a grant scoped to
any handler authorize a tree read.

### Fixed — the authority table flattened an intersection, and a sentinel arm was unreachable (0.8.2.22)

- **§6.8** — the handler-level authority is selected by **whether the access serves a live caller's
  request**, not by who derived the path. A path the handler derived is still the caller's access when
  its existence, content or effect reaches the caller: a listing entry, an extract or snapshot binding,
  a merge expansion, a subscription payload. For those, **the caller's capability and the handler's own
  grant MUST both pass**. The previous three-row exclusive table could not express an intersection, so
  it made a handler grant — broad by construction — the authority for listing entries, which makes the
  listing filter vacuous, and for merge expansions, which let a narrow caller merge anywhere the tree
  handler can reach.
- **§6.3** — the listing filter binds **any handler returning a multi-entry result whose entries are
  tree paths**, not only the tree handler; the rule was already general and the sentence was not.
  `filter_listing`'s parameter is renamed `authority`, matching the function it calls.
- **§5.2** — `check_resource_scope`'s pattern arm takes the unmatchable-exclude sentinel **first**. The
  arm's coverage test was correct in isolation and unreachable: `patterns_overlap` `continue`s on a
  sentinel, skipping it. §5.4's consumer table gains the site and states that a sentinel arm is a
  **control-flow obligation**, not a line.
- **§6.3** — a **pattern subject** routed into `check_path_permission` is authorized as §5.2 authorizes
  a pattern target, with an empty caller-exclude set: every overlapping grant exclude is uncovered, so
  the check MUST DENY.
- **§5.2 / §5.6** — the scope type is a property of the **dimension**, supplied by the call site, never
  read from a received entity; a `scope` whose declared `type` contradicts its dimension is refused
  `403 capability_denied`. `matches_scope` and `scope_subset` take it as a parameter, which is the
  signature every conformant implementation already has — this corrects the specification toward the
  implementations rather than the reverse.

> Both revisions landed in one editing pass and no peer observed an intermediate `0.8.2.22` document.
> Each delta carries its own revision marker inline so either number resolves to the rules it named.

### Added — outbound sub-dispatch authorization, and the authority it runs against (0.8.2.17)

**PD-2 lands.** `check_permission` now MUST run **before a locally-originated sub-dispatch leaves the
peer**, all four dimensions applied, `target_peer = extract_peer(uri, local_peer_id)`. The rule that
took nine months to state is that *which authority the check runs against* depends on what the
sub-dispatch spends, and the two cases are different questions:

- **Ambient authority** — no capability presented; the sub-dispatch rides the executing handler's
  grant. **Dimension 4 binds that grant.** A handler with no `peers` scope cannot reach a foreign
  peer. This is the confused-deputy ceiling the dimension exists for.
- **Presented authority** — a capability that is **not** the propagated `caller_capability` but a
  distinct credential minted **by the target peer** naming **this peer** as `grantee`. **Its own four
  dimensions authorize the sub-dispatch; the dispatching handler's `peers` scope is not consulted.**
  The party that decides what may happen at a peer is that peer, and it already has.

The presented arm MUST verify `granter` is the target, `grantee` is the local peer, validity
(chain-verified, unexpired, unrevoked), and coverage. **A capability failing any of these is not
presented authority and falls back to the ambient arm.** All four checks already existed; this
composes them.

**This is disjoint from §6.2's confused-deputy prohibition, not an exception to it.** That rule
forbids re-spending the *propagated caller capability* at a target the caller chose. Presented
authority is the opposite shape — minted for this purpose, by the party being accessed. A caller can
steer the handler only toward peers that have already granted this peer something.

**Two readings were rejected.** Keying the exemption on *the connection the request arrived on* makes
an authority question turn on a transport predicate, and fails at both edges — a fresh dial back to
the same peer is the same authority question and would be refused, while reusing an inbound
connection to reach a **third** peer would be wrongly exempted. Requiring per-connection `peers`
scopes on handler grants makes a bootstrap-time grant depend on who later connects.

**§9.1 carries the two-arm conformance row**, which is what makes this checkable: the negative arm
(ambient at a foreign peer → refused) is new and drivable today; the positive arm is already driven.

### Changed — `path_required` is raised by the handler, not the dispatcher (0.8.2.17)

`0.8.2.14` declared the code and stated it was raised *"at dispatch, before the handler runs"* for a
*"directly-callable"* operation. **The dispatch-level reading is withdrawn: no conformant
implementation could execute it.** `system/handler/operation-spec` declares `input_type` and
`output_type` and nothing else, so a dispatcher — which holds the resolved manifest and nothing else
— has no field against which to decide whether an operation requires a `resource`. §3.2 states the
complementary rule directly, and the two sections disagreed.

**`path_required` stays a core-declared code** — the condition is core's, since `resource` is a §3.2
EXECUTE field — and **the raising site is the handler**, with each operation's own specification
saying whether it requires a `resource`. `EXTENSION-CONTENT` §6.2/§6.3 are the worked examples;
operations for which no resource is legitimate simply do not carry the requirement. **A manifest
field declaring the requirement is a coherent design and is a separate, wire-visible proposal** — it
should be argued on its merits, not landed as a defect fix.

### Fixed — the id-scope pin never reached the pseudocode (0.8.2.16)

§5.2 has said since `0.8.1` that `matches_scope` matches each dimension **by its scope type** —
`path-scope` (handlers, resources) canonicalized, `id-scope` (operations, peers) compared as literal
identifiers — and that **"an id dimension canonicalized is a conformance defect."** The id-scope
pattern grammar paragraph restates it normatively.

**Both normative code blocks did exactly what that prose forbids.** `matches_scope` wrapped value
and pattern in `canonicalize(…)` unconditionally, in both the include and exclude arms, with no
scope-type branch anywhere — under a comment reading `; Uniform scope check for all grant
dimensions`, which is the defect declaring itself. §5.5a's `scope_subset` did the same. Between them
they are reached by `check_permission`, `check_grant_covers` and `check_path_permission` on
`operations` and `peers`, and by `grant_subset` on `operations` and `peers` during **delegation** —
where a child grant wrongly judged a subset of its parent widens authority down a chain nobody
re-checks.

An implementation reading the prose was conformant; one reading the pseudocode was not, **and the
pseudocode is what gets transcribed.** The measured instance is a generated peer that failed in both
directions at once — an `include` over-granted and an `exclude` over-denied — and neither is visible
while both are present, because they partially mask each other. That is the `ALLOW`-bug class §5.2's
own parenthetical names.

Both functions now dispatch on the scope entity's own `type`, so no call site changes. The literal
arm recognizes exactly the two wildcard forms the grammar paragraph allows — bare `*`, and a
trailing `/*` matching by literal segment-prefix — and applies none of the §5.4 path transforms.
`scope_subset` additionally rejects a child/parent pair whose scope types differ, which is a
malformed grant. **No new conformance row: §9.1's F40 row already states the rule.** No wire change.

Found by the enumeration this class earned: **a rule ruled in prose whose normative pseudocode was
never swept** — the third instance in one week, after `0.8.2.15` and `0.8.2.10`. A code block shares
none of its rule's vocabulary, so neither a subject enumeration nor a term grep reaches it. The
sweep that found this one was bounded and complete: **14 `canonicalize` call sites in this
specification, of which exactly these two were wrong.**

### Fixed — two sections computed a different `system/peer` hash than the type system defines (0.8.2.15)

§3.5 defines `system/peer` as `{public_key, key_type}` and states outright that **`peer_id` MUST NOT
appear in the hashable basis**; §1's envelope example and §3.5's prose both agree, and both cite the
revision that moved the field out. **§4.5a item 1a and §4.6's `peer_entity` pseudocode still carried
the pre-v7.65 shape.**

That is not editorial. `peer_hash` from §4.6 is what goes into `signature.signer`, and the same value
is a capability's `grantee` and `granter` (§3.6) and the `{peer_id_hex}` path segment. **Two
implementers, each conformant to a section they read, computed different bytes for one identity**, so
§5.2's `signer == author` and `grantee == author` equalities — byte-wise by §5.3 — fail across that
pair at connect, on a correct signature with a correct key.

The retired shape also defeated the property the carrying sections were asserting. §4.5a item 1a pins
the entity to the ECFv1-SHA-256 floor so the identity hash is *"the same bytes on every connection in
the network."* A basis containing `peer_id` cannot have that property: §4.5's wire-acceptance
carve-out lets one key be presented in more than one `peer_id` wire form, so the two forms hash to two
identities — the second-form manufacture **item 4 of the same section prohibits**. Item 1a's argument
was sound and its field list contradicted its conclusion; only the field list changed.

**§4.6's `authenticate_entity` is untouched and still carries `{peer_id, public_key, key_type, nonce}`**
— it is a different type, the challenge payload, and `peer_id` there is a signed assertion of the wire
form being presented. A comment now says so, because stripping the field from both blocks is the
obvious wrong sweep.

Also corrected: the crypto-agility `SEEDS.md` restatement. **No wire change and no new requirement** —
§3.5's `MUST NOT` was already landed; four consumers of it stopped contradicting it. Raised by
conformance measurement of the connect handshake across a population of independently built peers.

### Fixed — the default handler grant spans the whole local store, not just the peer's own prefix (0.8.2.3)

`0.8.2.2` pinned the default per-handler self-grant's `resources` to `["/{local_peer_id}/*"]`, reasoning
that a bootstrap grant should reach only its own peer. **That was wrong, and it would have broken a
common case.**

A peer's store is one local address space keyed by peer id: `/{remote_peer_id}/…` names a **local**
region holding that peer's cached or mirrored data, so writing there is a local write and not a remote
reach. §6.3 already said this in terms — *"a grant with `peers` absent … may include resource paths like
`/{remote_peer_id}/data/*` for cached copies."* The network bound is carried entirely by the `peers`
dimension, which this grant omits and which therefore defaults to the local peer and is still checked.

Narrowing `resources` closed no hole that `peers` did not already close, and it broke handlers that
declare no scope and write a follow-mirror or a cached foreign site into their own store: because the
handler grant is the Dimension 3 ceiling on the in-process sub-dispatch path, those writes returned
`403`. The default is now `["/*/*"]` — the whole local store — and §6.2 states the reasoning so the
narrower reading is not re-derived.

The `peers`-omitted half of `0.8.2.2` stands unchanged, and it is the half that carries the security
property.

### Fixed — a foreign-namespace request had a MUST with no code anyone could find (0.8.2.2)

§1.4 has always required a peer to reject an inbound EXECUTE naming another peer's namespace, with
status **400 `invalid_request`**. But `invalid_request` appeared **exactly once in the entire
specification — inside that MUST itself**. It was not in §3.3's status vocabulary, not in §8.3's
status table, not in §5.2a's verdict enumeration, not in §9.1's conformance list, and no conformance
check read it. Meanwhile §6.2 defined `404 handler_not_found` as *"no handler is registered at the
dispatch path"* — a description a foreign-namespace path satisfies on its face.

So an implementer reading §1.4 met a code used nowhere else in the document, and an implementer
reading §6.2 met a code whose definition fit. They picked the one the specification explained.
**This was not a divergence from a clear rule; the rule had nowhere for the agreement to live.**

`invalid_request` is now declared in §3.3 and §8.3, enumerated as a pre-dispatch row in §5.2a, named
in §6.5's dispatch chain and in §9.1's conformance list, and `handler_not_found`'s definition in §6.2
is scoped so it can no longer absorb this input. §1.4 and §3.6 additionally state **where the `peers`
capability dimension is evaluated** — it is unreachable on the inbound path by construction and does
its work on internal sub-dispatch, a distinction whose absence had led to the reasonable-but-wrong
conclusion that the dimension was inert.

Also pinned: the **default per-handler self-grant**. §6.2's `or []` fallback was under-specified — a
handler that declares no scope still needs its own namespace to do anything impure — so the default is
now `resources: ["/{local_peer_id}/*"]` with `peers` omitted. **This is a default, not a ceiling:** a
handler needing broader authority declares it, and the declaration is what the grant is built from.

Version header moves `0.8.2.1` → **`0.8.2.2`**.

### Fixed — a pre-hello `authenticate` had two answers in one table (0.8.2.1)

`ENTITY-CORE-PROTOCOL` §4.7 answered one input twice: row 6 pinned an `authenticate` arriving before
any hello nonce to **401 `invalid_nonce`**, while row 10's parenthetical claimed the same input as an
out-of-order operation at **400 `connection_sequence_error`**. Two readings, each defensible from the
text, mutually non-conformant — so *"follow §4.7"* was not a well-defined position.

**Ruled 401 `invalid_nonce`**: a captured `authenticate` replayed onto a fresh connection is exactly
this input, so it is an authentication failure and not a malformed request — the same status already
pinned for a replayed `authenticate` on an established connection. Row 10's example was narrowed, §4.2's
ordering MUST was given the status and code it had always lacked, §5.2a's connect-time rows were
completed, the precedence order between §4.6, §4.7 and §9.1 is now stated once, and §9.1's conformance
list names the status.

Version header moves `0.8.2` → **`0.8.2.1`**.

### Changed — the ECF conformance corpus is de-versioned

`conformance-vectors-v1.{cbor,diag}` → **`conformance-vectors.{cbor,diag}`**. A corpus is identified by
its name and its artifact's sha256, never by a version stamp in its filename; the stamp was a second
identity that could disagree with the bytes.

The 0.8.2 entry below records that the `-v1` stamp was deliberately kept, because Appendix E of
`ENTITY-CBOR-ENCODING` stated the corpus-version citation rule normatively and no proposal covered
retiring it. **That amendment has now landed** — Appendix E's five citation sites were rewritten, and
§E.6's MUST no longer demands a citation form that no longer exists.

**The artifact is byte-identical across the rename** — sha256
`9695b1f1d939cfdfdd4297f8ad32122d424b1ec180cfae74c92d509d88f7c6dc`, 71 vectors. Anyone vendoring this
corpus should **verify by digest, never by filename**: a rename does not merely make a vendored copy
stale, it makes a filename-matching vendor check unable to look at all, and that reports as a warning
rather than as a failure.

### Removed — `ENTITY-CORE-MACHINE-SPEC`

Retired. It was a *derived* document — a hand-maintained condensed restatement of the three real specs —
and it had drifted unchecked from 0.8.0 while carrying a canonical declaration. When two of its sections
were finally examined, both were wrong: one was missing an error row entirely, the other still carried a
blanket `403` the real spec had corrected two releases earlier.

A derived document is not repaid by re-synchronizing it. The maintenance burden is unbounded, nothing
mechanically checks one spec against another, and the drift is reachable only by a human reading both
documents side by side. **No replacement is planned, and a new "condensed" or "implementation" edition
of any spec here would be the same defect.** Its §1.8 content lives at `ENTITY-CBOR-ENCODING` §5.4; its
error table was always `ENTITY-CORE-PROTOCOL` §4.7.

### Removed — the two authoring standards are no longer duplicated here

`SPECIFICATION-FORMAT` and `STYLE-NAMING-CONVENTIONS` are now **single-homed in
`entity-system-architecture`**, where they are authored. They are authoring standards — how a normative
spec document is written — so the reader who needs them is a spec author rather than an implementer of
this protocol. Specs here cite them by document name.

Both copies had drifted, in opposite directions, which is why they were consolidated rather than
re-synchronized: this repo's `SPECIFICATION-FORMAT` was a strict stale subset missing an entire
subsection family, and its `STYLE-NAMING-CONVENTIONS` had forked **both** ways, so neither side was a
superset and there was no clean one to sync from.

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
