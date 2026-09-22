# Changelog

All notable changes to this project are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project aims to follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

**Three documents, three version lines.** Each specification's `**Version**:` header is a claim
about *that document's* content and moves when that document's obligations do.
`ENTITY-CORE-PROTOCOL` carries this repository's release number; `ENTITY-CBOR-ENCODING` (`1.x`) and
`ENTITY-NATIVE-TYPE-SYSTEM` (`4.x`) are on independent ladders and have never tracked it. Read each
header rather than inferring one from another.

**Why the entries below carry a fourth component.** Development between releases runs under a
trailing component on `ENTITY-CORE-PROTOCOL`'s version — `0.8.2.1`, `0.8.2.2`, and so on — whose
only job is to tell implementers that core text moved without inventing a release number between
releases. Each is a landed fold. **The component is stripped when a release is cut**, the same way
`0.8.0.x` was stripped at `0.8.2`; the per-revision marks stay in these notes because
implementations cite them.

**The public surface these notes are measured against** is the normative text of the three
specifications in `specs/` and the conformance corpora in `specs/test-vectors/`. **The locked wire
core is untouched** — no renumber, no new opcode ([ADR-0002]). Several refusal *codes* are newly
declared in the status vocabulary, which is the point of the entries that do it: the rule already
existed and the code it named appeared nowhere a reader would look. Everything below is a rule
stated where the specification had nowhere for it to live, a rule corrected against a neighbouring
section that contradicted it, or a rule withdrawn.

### Changed in ways that can break an existing caller

**A peer conformant to the last published text can be non-conformant under this one.** Six items,
each a thing an implementation must now do differently; the sections further down carry the full
reasoning and the section numbers. The ordering rule and the authority intersection are the two
that change an answer already on the wire.

- **`501 unsupported_operation` is reachable only after the permission check passes.** A request
  that is both unauthorized and unimplemented now answers **`403`**, not `501`. A peer that tests
  operation-existence first must reorder the two — and until it does, its `501`-versus-`403`
  responses are a two-valued oracle that lets any caller holding *any* grant on a path enumerate
  that handler's whole operation set. The order was drawn in the dispatch chain and asserted in a
  parenthesis, and stated as an obligation in neither; an implementation was measured on the other
  reading. *(0.8.2.30)*
- **A path derived inside a caller's request is authorized by an intersection.** The caller's
  verified capability **and** the executing handler's own grant must **both** pass — derivation is
  not the discriminator; serving a live caller's request is. The conformance floor published the
  handler grant alone as the authority for eight revisions, so a peer built from that list grants
  strictly more than it should: a narrow caller can reach anywhere the tree handler can, and the
  listing filter is vacuous. *(0.8.2.22, swept through the floor at 0.8.2.31)*
- **An unmatchable scope pattern is fail-closed on both sides.** A capability carrying one is
  **invalid** at mint, delegate and verify, and an unmatchable `exclude` excludes **everything**.
  Previously an exclusion that matched nothing silently produced a grant wider than written, with
  no error anywhere. Capabilities that used to verify are now refused. *(0.8.2.21, ordered at
  0.8.2.22)*
- **An inbound request naming another peer's namespace is `400 invalid_request`.** The refusal
  itself is unchanged and always was `400`; what changed is that the `404 handler_not_found` row —
  the only code the specification previously explained at the point an implementer reads — now
  says so explicitly. A peer answering `404` here was never conformant and is now measurably
  non-conformant. *(0.8.2.2)*
- **Preserving content a peer did not model went `SHOULD` → `MUST`**, at the section that declares
  itself the canonical home of the entity-fidelity contract, and at the two unknown-format-code
  lines that are the same rule one noun over. It had said `SHOULD` while three other homes said
  `MUST`, so the conformance floor stated one rule at two strengths at once. Stripping an
  unmodelled field republishes a different content hash for the same entity, which is a
  correctness claim about the network and does not belong under a `SHOULD`. A peer that took the
  `SHOULD` literally is now non-conformant; a peer that implemented what the text meant is
  unaffected. **Deliberately authoring a derived entity is still not a fidelity violation** — the
  section now carries the frame that separates relaying and re-encoding from transforming.
  *(0.8.2.10)*
- **The ECF conformance corpus is de-versioned.** `conformance-vectors-v1.{cbor,diag}` are now
  `conformance-vectors.{cbor,diag}`. **No vector value changed** — the `.cbor` is byte-identical,
  `sha256:9695b1f1d939cfdfdd4297f8ad32122d424b1ec180cfae74c92d509d88f7c6dc`, 71 vectors, measured
  on both sides — but anything pinning the corpus **by filename** stops resolving. Verify a
  vendored copy by digest; a filename mismatch makes a vendor check report *could not look*, which
  is indistinguishable from a pass. The corpus changelog beside the vectors carries both artifacts'
  digests.

**Also withdrawn from the published tree**, each with its own section below:
`ENTITY-CORE-MACHINE-SPEC` (retired — it was a derived restatement and had drifted from the specs
it restated), and the two authoring standards `SPECIFICATION-FORMAT` and
`STYLE-NAMING-CONVENTIONS`, which are now single-homed in `entity-system-architecture` where they
are written. If you hold a link to any of the three, `.release-removals` says where it went.

### Fixed — the conformance floor's restating rows did not name their authorities (0.8.2.32)

`0.8.2.31` added to §9.1 the rule that **a row here that restates a rule stated elsewhere names that
section as its normative home** `[MUST]`, and then applied it only to the two rows it was
investigating. This is the sweep of the rest: **26 rows edited — 25 in §9.1 and one in §1.11 Boundary
Conformance**, which is a second normative floor. Every named home was confirmed by opening the cited
section.

**No rule changes.** The rows keep saying what they said; what they gain is a resolvable authority, so
the next time one drifts from its source the drift is visible from the copy.

The sharpest instance is a pair whose authorities point in **opposite** directions, while both rows
cited both sections and named neither:

- **`501`** — §6.2 declares itself the restatement and **§3.3** the authority.
- **`404`** — §3.3 declares itself the restatement and **§6.2** the authority.

A reader of the floor could not tell which way either pointed.

Two rows were examined and ruled **not** restatements: the `0.8.2.13` `system/*` withdrawal, whose
only home is that list, and §9.2's `SHOULD` row, which is outside the rule's scope. Whether a floor
row restates a rule or **is** its sole statement is decided by opening the cited section, never by the
row's shape — one candidate inverted on reading, because §6.3's listing-filter *pseudocode* is marked
Informative while the normative paragraph above it is the home.

### Fixed — the conformance floor published the authority rule `0.8.2.22` corrected (0.8.2.31)

§9.1's authority-selection row read *"selected by who named the path … handler-derived → the executing
handler's own grant … the propagated `caller_capability` is never the authority for a derived path."*
**§6.8 corrected that exact discriminator at `0.8.2.22`** — *derivation is not the discriminator* —
and requires the caller's capability **and** the handler grant to **both** pass for a path derived
within the caller's request.

**A peer built to the floor row flattens that intersection.** That makes the §6.3 listing filter
vacuous and lets a narrow caller merge anywhere the tree handler can reach: the hole `0.8.2.22`
closed, still published positively for eight revisions, in the MUST-implement list an implementer
builds from.

A second row was stale in the milder direction — the unmatchable-exclude row carried `0.8.2.21`'s
deny without `0.8.2.22`'s bound.

§9.1 now carries a standing `[MUST]`: **a row that restates a rule stated elsewhere names that section
as its normative home.** Its swept application is `0.8.2.32`.

> A section-granularity check does not find this class. Measured across 21 revisions, §9 already cites
> every section that gained a stamped MUST — so a row can cite §6.8 for one obligation while a second
> obligation in the same section is ungated, and the citation makes both look covered.

### Changed — `501` is reachable only after the permission check, and `ping` is an example (0.8.2.30)

**`501 unsupported_operation` is emitted only after `check_permission` passes.** A request that is
both unauthorized and unimplemented is **`403`**. Three homes already drew the order this way —
§6.5's chain, §6.7's assertion, §3.3's code row — and §6.2's *"the caller's authority is irrelevant"*
was the outlier: it was generalized at `0.8.2.6` to fix a **code** defect, and was then readable as an
ordering claim.

**The ordering is a confidentiality property, not a preference.** Testing operation-existence first
makes the response a two-valued oracle over a handler's manifest for any caller holding any grant on
that path, so the operation set becomes enumerable without authorization for it. §6.7 names that
expansion and refutes it **on the strength of this ordering** — a peer that reverses the two re-opens
the leak while every other rule still reads as satisfied. **At least one implementation changes.**

⚠ **A check driving the `501` row MUST use a grant that covers the probed operation**, or it measures
the ordering instead of the row.

**`ping` is an EXAMPLE.** Core defines no ping operation — no params type, no result type, no manifest
entry, no floor row, no code row — and `EXTENSION-NETWORK` is its sole owner; §4.2's `MUST` is
conditional on implementing it. A peer that does not implement it refuses correctly, and the
conformant code for that refusal is **`501`**, not `400 invalid_request`.

§4.7's half-open row keeps its `409` and loses its premise. It justified the status with *"the
pre-authorized-connect exception is scoped to an established connection"*, contradicting §4.2's *"in
any connection state"* at the same level. The `409` is right and the premise was not — **being
pre-authorized is not being in-order** — and the premise is the half a reader carries to the next
case.

### Fixed — the `system/peer` identity entity had eight homes and the last sweep took two (0.8.2.29)

`0.8.2.15` corrected the `system/peer` hashable basis at §4.5a item 1a and §4.6's pseudocode — the two
homes its proposal named — and **six further sites across two documents kept the retired shape**,
declaring three distinct shapes for one entity, including a `{peer_id}` form at §3.13 that no section
ever specified. The worst site is §4.4's block labelled **Normative wire example**: it shares none of
the rule's vocabulary, so neither a search by subject nor a search for the literal reaches it, **and
it is the site an implementer copies.**

Also in this revision:

- **§3.6 `peers:` patterns are canonicalized at comparison** (new rule 4). The prose wrote those
  values path-shaped, predating the id-scope grammar that matches them literally — so an include
  granted nothing and an exclude excluded nobody, **silently**. Dispositions are asymmetric: an
  unresolvable **include is dropped**, an unresolvable **exclude refuses the grant**. The pseudocode
  was already correct and the prose was the drifted half; `resolve_peer_scope` is now its executing
  arm.
- **A per-connection peer-id form cache, and a `MUST NOT` on auto-correlating peer-id forms.** The
  latter had been carried under a heading reading *"six normative SHOULDs"* and was mis-levelled at
  source.
- **§4.6 step 3 and §1.5 gave opposite keyworded outcomes on one hello** — step 3 tests the binding,
  §1.5 governs the spelling. A strict refusal had no conformant code and now has one.
- **`MUST accept the mint` is withdrawn.** It contradicted §1.5's `MAY` refuse and made every strict
  deployment non-conformant.
- **§4.5a item 1a's *"every consumer derives its hash"* is a property, not a prohibition on
  fetching**, and its exemption now reaches §4.5's transmission ban — without which no non-floor
  connection could be established.
- **Every restatement of the tag member carried §6.3's four-word shorthand and none carried the scope
  sentence defining it**, so the narrow reading was available and described no implementation. This is
  `0.8.2.28`'s enumeration defect inverted: there a home omitted a member, here the homes dropped its
  bound.
- **§4.11 calls the close a choice while its own Conformance clause constrains it four paragraphs
  away**, with no cross-reference. Continuing is the only disposition conformant on every connection.

### Fixed — the root-entity-hash refusal had a code in the pseudocode and no row in the table (0.8.2.28)

Which received entities §1.8's validate-on-receipt binds, and the code for a root/`included` mismatch.

`included` got `400 hash_mismatch` at `0.8.2.24`. **The root arm got the same code at `0.8.2.25` — but
only in §6.5's dispatch pseudocode.** §4.11's cause table, which is the prose home a reader consults
for pre-admission refusal codes, carried an `included` row and **no root row**. It does now.

**That gap is measurable rather than untidy, and it produced a confident false absence.** A careful
reader consulted §4.11's table, correctly reported that it assigns no code for the root arm, and
correctly declined to invent one — while §6.5 had assigned it three revisions earlier. **The table is
where that answer belongs**, so the fix is the row, not the reading.

> The known form of this defect is a rule that **executes** in a code block and is missed by a search
> for the words that state it. This is its mirror: the **enumeration of the class omits a member**, so
> a reader consults the *right* home and it answers *wrongly*. **After changing a pseudocode arm,
> check the enumeration of its class.**

`params`/`result` is not a `MUST`/`SHOULD` conflict. §3.4 already states the reason: the outer hash
covers the `params` bytes, so `params` is not a separately received entity. What §3.4 did not say —
and now does, in one sentence, at no change of strength — is that **§1.8's resolution-integrity
`MUST` governs the moment a handler resolves through an embedded hash for an authority decision.**
That rule lives under a different noun, so a reader looking up the field never reached it.

### Fixed — three fossils in one encoding specification, and a refusal class with no code (0.8.2.27)

`ENTITY-CBOR-ENCODING` restates rules whose authority lives elsewhere, and three of those
restatements had outlived the design they restate.

- **The `content_hash_format` registry is pinned to `ENTITY-CORE-PROTOCOL` §1.2 as its single
  normative home.** ECF §4.3/§4.4/§4.6 had `0x03` and `0x04` **transposed** against it, so the corpus
  bound `ecfv1-blake3` to two codes. **Latent, not live** — `0x00` is the only production code.
- **The format code is a varint, not a fixed octet.** §4.5 specified an OCTET; ECF's own Appendix E
  already required the varint reading. The tell that ties the two together is `0xFF`: §4.3 reserved it
  *"for an extension mechanism if >254 formats needed"*, an escape hatch only a one-byte field needs,
  in a registry whose own rule is that **255 is never allocatable, because a `0xFF` byte cannot
  terminate a varint.**
- **ECF §9.2's *"Decoders MUST accept any valid CBOR (not just deterministic)"* and *"preserve unknown
  tags"* are withdrawn.** That is pre-Option-B text contradicting §1.11 at `MUST` level, in the
  section an implementer reads while writing a decoder. Two adjacent sections were checked and do
  **not** conflict — §10.3 is the refusal's detection mechanism — which shrank the fix from three
  sections to one.

**`400 non_canonical_ecf` was over-narrowed, and the class it is named for had no code.** `0.8.2.24`
and `0.8.2.26` correctly ruled the code off the framing arm and off a mis-keyed `included` entry,
arguing that the code selects the caller's remedy — and wrote the scope as *"§6.3 defines that code
for tag-policy violations specifically"*, narrower than that argument supports. **Both exclusions
stand; only the over-narrowing is corrected.** §4.7 had no row for the code at all, and §3.3's
enumeration — which §8.3 declares authoritative — was missing both it and `hash_mismatch`. Verified
strictly additive against independently built implementations.

### Fixed — the signature message had five normative homes and the executable one disagreed (0.8.2.26)

**A signature is computed over the target entity's FULL `content_hash` — format code ‖ digest**
`[MUST]`. §7.3 is the single normative home; every other statement restates it and now names it.

The question was filed against three documents. **There are five homes**, and the two nobody had
enumerated — §4.6's `authenticate` pseudocode and §3.5's invariant-pointer path width — both already
state the hash reading. §10.2 was never a third position: it cites §7.3 as its authority in the same
sentence that drops the format code.

**The sole outlier is a test-vector category, and it declares itself downstream of §7.3 in its own
text.** The artifact contains its own correct input one category above: `content_hash.1` produces
exactly the value §7.3 names as the message, and the signature category three rows down does not use
it. Every live implementation implements §7.3; **the only implementation of the raw-bytes reading was
the fixture.**

Three grounds, and the third is why this is not a coin flip: **a verifier reaching a signature through
the §3.5 invariant pointer holds the hash and may hold no copy of the entity**, and the format code is
the domain separator §7.3 exists to bind.

⚠ **The vectors are NOT regenerated here.** The corpus changelog records the artifact as
known-divergent so nobody reads the current digest as blessed.

The rest of this revision withdraws, demotes, permits or partitions; **none of it adds an obligation
to a conformant peer**:

- **§4.11 arm (a) splits into a1/a2** — the close is a *choice* where the frame was consumed whole and
  is **forced** where the stream is desynchronized. Arm (f) obliged the impossible on the truncated
  half.
- **`non-canonical` is dropped from §4.11's framing row.** The framing arm and §5.4's tag policy now
  **partition** the input, instead of both `MUST`ing opposite codes for one tagged data field.
- **A uniform verdict on an *unreferenced* `included` entry MUST NOT be required.** A list
  representation has no wire key to mis-key, so requiring one verdict would make a conformant
  mechanism illegal.
- **§1.8 item 1's two mechanisms are distinguished by cost**: binding the key is one check at one
  site; discarding the key and addressing by validated `content_hash` is *N* ingresses, and *N−1* of
  *N* is wire-indistinguishable from *N* of *N*.

### Fixed — the pre-admission refusal is one invariant, not five patches (0.8.2.24 → 0.8.2.25)

**One un-parseable frame produced three different caller-observable answers across three independently
built implementations** — a bare close, a silent drop, and a coded `400`.

The same invariant was already stated in **five places at four strengths**, three of them giving the
same reason in nearly the same words and **none cross-referencing another**: §4.6 (connect-auth,
`MUST` emit coded `401`, *"a bare close is non-conformant"*), §5.2a (hash-binding, `MUST` emit; drop
**and** bare close both non-conformant), §4.10a (oversize, `SHOULD` … otherwise `MAY` close), §3.3
(wrong root type, `MUST` close, with no coded frame at all), and the **framing arm, unstated**.

**New §4.11 states it once:** a coded frame is **mandatory**, the close is **optional**, and a silent
drop and a bare close are **two distinct non-conformances**. The frame belongs to the **class**; the
**code** belongs to the **cause** — framing and wrong-root-type are `400 invalid_request`.

Four landed sites contradicted it and **three of the four are not prose**: §6.5's dispatch-chain
pseudocode (*"Other type? → Invalid. Close connection."*, with no failure arm on any of its
decode/validate steps), §9's conformance inventory — **which GATED the bare close** — and §3.3's prose
`MUST`. **The dispatch chain is the block an implementer copies**, which is the likeliest origin of
two of the three divergent behaviours.

Checked and **exempt**, stated rather than skipped: `EXTENSION-SIGNALING` §9.2 (a separate three-verb
mailbox transport) and `EXTENSION-NETWORK` §10 (a deliberate shutdown).

Two further rules, both forced by measurement:

- **A narrowing MUST NOT be lossy about its own emptiness, at every seam.** `0.8.2.20`'s
  dispatch-boundary fix deletes this rule's input, and a guard green on 108 in-tree rows was dead at
  the wire. Ordering is *one* way to satisfy non-lossiness and does not reach the second seam.
- **The empty-result rule binds where the absent case is WIDER than the request.** A broad result
  refuses `400 path_required`; an optional **filter** answers the empty result at `200`, because
  `path_required` means *supply a resource* and this caller supplied one. **Operations must now
  declare which shape they are** — one sentence was censused and bound to 2, 5 and 13 operations by
  three different readers, each inferring a per-operation property no specification declared.

**Cost, recorded before the run rather than after it: 21 of 34 measured peers move PASS → FAIL on the
§4.10(a) strengthening.** That is the only direction the rest of the corpus already points.

Two new conformance requirements. `CORE-PREADMISSION-REFUSAL-1` arm (f) — a refusal on a multiplexed
connection carrying an admitted request — cannot be inferred from the others and has never been
driven. `CORE-RESOURCE-TWO-EMPTIES-1` **must cross a socket**: an in-tree test that builds the handler
context directly is structurally blind to the failure it exists for.

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

### Fixed — an unmatchable scope pattern is fail-closed in an include and fail-OPEN in an exclude (0.8.2.21)

**`0.8.2.20` ruled that `matches_pattern` MUST return false when either operand is the unmatchable
sentinel, and argued its safety from the include side only.** Matches-nothing is fail-closed in an
include and **fail-open in an exclude**: a granter writing `exclude: ["*/secret"]` gets an exclusion
that excludes nothing and a grant **silently wider than written**, with no error anywhere, because the
sentinel is designed not to raise.

Measured, it is **three sites**, not the one reported — `matches_scope`'s exclude loop (every
dimension of every grant), `check_resource_scope`'s concrete arm, and its pattern arm, which was
fail-closed **by accident** via a negated test. Fixed in both directions:

- **A capability carrying an unmatchable scope pattern is INVALID at mint, delegate and verify**
  `[MUST]`, not `MAY` — two conformant peers would otherwise reach different authorization decisions
  on the same bytes.
- **An unmatchable exclude excludes EVERYTHING at evaluation.**

Stated at the scope layer, which already knows the position, so `matches_pattern` stays uniform over
its operands and remains transcribable into any language.

Four further deltas:

- **`effective_targets` returns the RAW survivor and decides the skip canonically.** `0.8.2.20`'s
  formulation contradicted §6.13 **in the same revision** — that section derives the install pattern
  against the `system/handler/` prefix and its own worked example is peer-relative. **Undetectable by
  the existing check**: every arm passes under either reading, so an implementation reading the
  pseudocode literally goes conformance-green and then answers `path_required` where §6.13 expects a
  pattern.
- **Path validation is a property of the BOUNDARY, not of the channel.** `tree:extract` and
  `tree:merge` derive paths from `params`, which no resource-target pre-validator ever sees, and the
  store boundary **asserted** under a comment claiming pre-validation — a wire-reachable remote
  denial of service. §5.4 and §6.7 already named `params`; **a rule stated over an enumeration of
  channels invites an implementation that enumerates channels.**
- **The handler-level check's authority is selected by who NAMED the path, not by who initiated the
  chain.** Caller-named paths take the caller's capability, handler-derived paths take the handler's
  own grant, peer-root takes no check. §6.8's *"voluntarily"* is struck, and §6.3's parameter is
  renamed **`authority`** — two implementations filled a parameter named `capability` from
  attribution. ⚠ **`0.8.2.22` supersedes the discriminator itself**: derivation is not the test, and
  the two authorities can **intersect**.
- **The §6.7 read carve-out is CLOSED**, with the region searched named so the negative is reviewable.
  The listing filter already checks every returned entry individually, so a performance rationale for
  exempting reads would have had to exempt that too.

Two new conformance requirements, both wire-observable and both security vectors:
`CORE-EXCLUDE-UNMATCHABLE-1` — **with** the well-formed-exclude control, without which the row cannot
tell a working exclusion from a peer that denies everything — and `CORE-PARAMS-PATH-TOTAL-1`, which
cannot be a false pass, because a peer that pre-validates only the resource target either answers
`200` or dies.

### Fixed — the authorization subject is the effective set, and a discarded verdict is a class (0.8.2.20)

Six deltas. The first three close an authorization bypass.

- **`effective_targets` is a named function in §5.2**, called by the authorizer and by the handler. It
  was **three** layers, not two — §9.1's conformance floor derived the set a **third** time, in prose.
  That row now cites the function and **MUST NOT restate its derivation**. The subject rule is stated
  generally — **subject ⊆ `effective_targets`** — with `effective[0]` as its single-resource
  specialization, so the first set-valued operation does not meet a half-rule.
- **A pattern target is `400 malformed_resource` for an operation requiring a concrete path.** §2.1's
  two undeclared codes land in §3.3's enumeration.
- **§6.7 goes act-neutral: reads or writes.**
- **The *"defense-in-depth"* characterization is withdrawn at SEVEN sites, not three.** The four
  missed by the first enumeration include **both pseudocode comments inside the block that implements
  the rule**, and §9.1 again. §5.2's own list says *"Both levels MUST pass"* two lines under the
  withdrawn sentence. ⚠ **The causal story is withdrawn with it** — it measures the other way: of five
  backends, the three that transcribe the phrase all wrote the check, and the two that never mention
  it never wrote it.
- **`canonicalize` is total, and its sentinel is `/never-match`.** Chosen over forms that are
  themselves strings the specification says a path cannot be: `/never-match` violates no §1.4 rule at
  all — it is unreachable as a canonical path **structurally**, needing no new prohibition. **All
  three consumers are ruled**, not one: matcher, validator, storage.
- **A discarded validation verdict is a CLASS, not a function.** `validate_absolute_path`'s verdict was
  discarded at **both** its call sites, one of them under a comment reading *"MUST — reject malformed
  `peer_id` segment"*. **Five sites, two functions, one defect** — and the class was never the search,
  because the finding named a function.

### Fixed — a multi-signature root never relaxes Dimension 4 (0.8.2.19)

**A live over-acceptance in all three ground-up implementations**, and the opposite of the
under-acceptance it was first reported as. The single-signature branch requires the root granter to be
the frame peer; **the multi-signature branch accepted a root when the frame peer was merely among the
signers.** That branch is correct for its original purpose — a peer verifying its own group root,
where the frame is **local** — and `0.8.2.18` repurposed the frame to the **target** and silently
re-scoped it, so a K-of-2 credential merely *including* the target relaxed Dimension 4.

**Neither rule is wrong alone; they composed into a hole.** The general form is folded with it:
**changing what a shared parameter MEANS re-scopes every check that reads it, and those readers are
listed nowhere.**

Second correction: the clause named a case that cannot occur — a peer identity is never a
multi-granter entity — so it read as conditional when its only effect is refusal. Every implementation
had built the effect; the text now states it.

**Wire core untouched.**

### Fixed — the handler frame binds a check whose four inputs were undefined (0.8.2.18)

Seven deltas, every one of them raised by having **built** `0.8.2.17` and driven it three ways.

- **The granter is read at the chain ROOT, the grantee at the leaf.** The leaf reading accepts only
  the unattenuated grant and refuses every narrowing of it — backwards from every other attenuation
  rule here, and it refuses every cross-peer continuation advance.
- **Dimension 1's handler pattern on an outbound sub-dispatch is the target URI's peer-relative
  path.** The check was obliged with no defined input.
- **A presented capability is evaluated in the TARGET's frame.** In the dispatcher's frame, the
  absent-`peers` default and §5.5's root-trust rule each refuse every credential the arm exists to
  accept — so the arm looks implemented and denies everything.
- **The check binds any outbound dispatch originated while a handler body executes**, autonomous
  origination included; a top-level self-origination is out of scope for the ambient arm. The ambient
  arm confines a **delegated** authority, and a peer originating as itself is the root of its own.
- **Absent resource is `path_required`; more-than-one is `ambiguous_resource`.** The register
  paragraph predated `0.8.2.14`'s split and still collapsed them, which is why two readers of landed
  text reached opposite codes. The general form is now stated once, so the next handler specification
  need not re-derive it.
- **§3.3's parenthetical named two operations as needing no resource**, where `EXTENSION-IDENTITY` §6
  gives both a resource target under an architecture-side `MUST`, at two tables. One revision old, and
  ours. Replaced with an operation that genuinely takes none, and with the **test** rather than the
  instance.
- **§6.5 signature ingestion is no longer scoped to the dispatch chain.** As written, a capability
  minted in the connect response had its root signature bound at **no path**, and every chain rooted
  at it was unverifiable locally. §5 already required the result; the only mechanism that produces it
  was unreachable there.

**Four of the seven cost a conformant peer nothing; two move one implementation each; one is an audit.
Nothing takes a currently-green check red. Wire core untouched.**

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

### Changed — `path_required` is a core code, because the condition is core's (0.8.2.14)

An extension `MUST`-ed **`400 path_required`** at two operations while its own error-code appendix
declared itself the **closed set** for that handler and omitted it — and this row, which the
extension-development guide claimed *"sanctions them by name"*, enumerated five spellings rather than
the category. **So a peer obeying the appendix failed the conformance suite, and a peer passing the
suite emitted a spelling its own extension called non-conformant.** Five implementations and two
conformance checks had already picked the second reading.

**The deciding test is whose condition it is.** `path_required` is raised by the **dispatcher**, before
any handler runs, on §3.2's path-as-resource rule. A code belongs in the set owned by the document
that defines the condition raising it — so this one is core's, and locating it in an extension locates
it where it **cannot be complete**. The alternative, a row in each extension appendix, restates a core
condition once per extension.

The row now states the condition, that it is raised **at dispatch** and is therefore available to every
handler, and that **it is not a synonym for `invalid_request`** — the remedy differs, and the code is
what selects it.

**No behaviour change for any implementation.** This makes the specification agree with what five of
them already emit.

### Removed — the `system/*` installation reservation is withdrawn (0.8.2.13)

**Retracts `0.8.2.12`, which narrowed this rule.** The narrowing asked the wrong question: it argued
about the rule's **scope** when the rule should not have existed. **Withdrawn, not re-scoped.** Three
findings, each disqualifying on its own:

1. **It was never authorized.** It entered as **one row in a design-revision migration table** —
   *"System paths | Not reserved | `system/*` reserved for system handlers"* — with no rationale, then
   or in any revision since. **The same revision introduced the structured grant model, in the row
   below it** — so the reservation and the mechanism that makes it redundant arrived together.
2. **It named a party this specification does not define.** *"User-installed handlers."* There is no
   definition of a user, and none distinguishing one from the party running a deployment, the party
   administering it, an extension author, or an ordinary caller — and implementations returned that
   undefined word to callers in the refusal message.
3. **It was not part of the authorization system.** Every engine enforced it as a hardcoded prefix
   match firing **before** authorization is consulted, while `register` already derives its pattern
   from `resource.targets[0]` and the standard dispatch capability check on `resource` already
   decides, per caller and per path, whether that caller may install there. **The prefix rule
   overrode a decision the deployment had deliberately made.**

**What replaces it is the check that was always underneath: install authorization is the capability
check on the install path.** An informative note records the real consideration — a handler bound over
a bootstrapped one substitutes its behaviour — as something a deployment will **often** refuse, not as
a requirement. Whether to permit installation at a `system/*` path is a deployment's risk decision.

**It constrains nothing cross-peer**: no wire form changes, and a caller sees only a refusal it must
already handle.

### Changed — the `system/*` reservation is scoped to the dispatch path (0.8.2.12)

> ⚠ **Superseded by `0.8.2.13`, which withdraws the rule entirely.** Retained because the reading it
> corrected was live in the text for the file's whole history.

§6.2 reserved `system/*` against *"user-installed handlers"* and stated **no purpose**, in every
revision since the file's first commit. **Every standard extension lives under `system/*`**, so the
widest reading — the one a reader with no rationale to scope against arrives at — described a protocol
in which **no standard extension can be installed on any peer**.

Scoped to the dispatch path: a handler installed via `system/handler:register` `MUST NOT` be installed
at `system/*`, refused `403 forbidden_pattern`, publishing nothing. A peer's own composition —
bootstrap handlers and standard extensions installed in-process by peer-owner code — installs there by
construction and is not constrained.

The purpose is stated with the rule: **a wire caller that could install at `system/*` could register at
`system/tree` and shadow the peer's own store operations for every subsequent dispatch.**

**No conformant peer's behaviour changes.** §9.1's row followed, because it restated the retired
party-based scoping in different words and would otherwise have been left asserting it.

### Fixed — the admission predicate was a type the specification already had (0.8.2.11)

`EXTENSION-TREE` Appendix A mandated `400 invalid_request` when a submitted entity *"does not decode"*
and **never said what decoding is**. It was already written: `put-request.entity` is typed
`core/entity`, and §8.1 declares three fields with **no `optional` marker on any of them**.

- **§6.3 gains the admission ladder** — receipt, not authoring; structure, then hash. **The ordering is
  a data dependency, not a convention**: step two's inputs are exactly what step one establishes.
- **§9.1 gains the row**, naming the discriminating input as the one carrying **both** faults, since
  that is the input a row-scoped author does not write.
- ⛔ **The one home that disagreed is the one that mattered.** `ENTITY-NATIVE-TYPE-SYSTEM` §2.8 — the
  only table in either document describing what is on the wire at a `core/entity` slot — carried
  **`content_hash?`**, optional. Six other homes carry the strong form. Corrected, and the row now
  names §8.1 as its authority, so the next divergence is visible from the copy.

**The receipt-versus-authoring dichotomy is false: both layers exist**, and `SDK-EXTENSION-OPERATIONS`
§3.2 already assigns authoring to the SDK — `put(path, type, data) -> hash` cannot return that hash
without computing it.

### Changed — preserving content a peer did not model is a MUST, not a SHOULD (0.8.2.10)

**One rule, thirteen homes, two strengths — and the canonical home carried the weak one.**
`ENTITY-CORE-PROTOCOL` §2.10 and `ENTITY-NATIVE-TYPE-SYSTEM` §2.4 said `MUST`; §1.8 item 5 and
`ENTITY-CBOR-ENCODING` §5.4 item 5 said `SHOULD` — **and §5.4 is the section that declares itself the
canonical home of the entity-fidelity contract.** §9.1's MUST-implement list carried the rule twice,
once through each side, so the floor stated it **at both strengths at once**.

The argument that settles it is §2.10's own: **content hashing covers all of `{type, data}`, so a peer
that strips a field it did not model publishes a different hash for the same entity and breaks content
addressing for every downstream peer.** That is a correctness claim about the network, and a
correctness claim about the network does not belong under a `SHOULD`. §5.4 also disagreed with itself
four lines apart — the numbered list said `SHOULD` while the governing paragraph made losslessness
precondition (b) of the re-encode mechanism, at `MUST`.

**No implementation changes; this aligns prose with a gate that was already running.**

⭐ **A bare `MUST` would have been the wrong fix, and §5.4 now carries the frame that makes it safe.
There are THREE acts, not two.** *Relaying* an entity and *re-encoding* one are both claims that this
is still the sender's entity, and both must preserve every byte of meaning. ***Transforming* one —
deliberately authoring a derived entity — is neither**: new content hash, publisher signs it as their
own, original intact and independently addressable, and the only loss is dedup against the original,
which is a cost the transformer chose. **A transform is not a fidelity violation. Silently emitting a
transform while claiming a relay is.** Without that paragraph, *"MUST preserve unknown fields"* reads
as *"you may never transform an entity"* — which is false, and worse than the `SHOULD` it replaces.

Two further homes were found at fold time and were **the defect surviving its own fix**:
`ENTITY-CBOR-ENCODING` §4.6 and §9.3 said unknown **format codes** `SHOULD` be preserved when
forwarding — the same rule one noun over.

### Fixed — the code-slot rule stated a permission and a prohibition and never the consequence (0.8.2.9)

**Two implementations independently concluded that the `400` slot had no governing set.** It has one:
§3.3's `400` row names `invalid_request` as the default, enumerates five cross-cutting specifics, and
says in terms that it *"is the generic 400 code an extension handler uses for a structurally invalid
request."* **The `400` row is the most completely specified row in the table** — it is what the `500`
row was brought up to match at `0.8.2.8`, not the reverse.

**When two careful readers reach the same wrong conclusion from a technically correct sentence, the
sentence is the defect.** The cause is visible in the text: `0.8.2.7` stated the **permission** (a
specific code `MAY` be used where defined) and the **prohibition** (an undefined spelling is
non-conformant) and **never the consequence** — what a peer emits when it wanted a specific code and
none is defined. A reader holding an undefined spelling saw a rule that forbids what they have and
does not say what to use instead, so they read it as an unfilled slot.

The consequence is now stated at the point of the prohibition: **an undefined spelling falls back to
that status's default**, the **absence of a table is not an unfilled slot**, and where the condition
genuinely names something a caller would branch on, the peer **holds it as a named divergence and
routes it** rather than minting a site.

Companion fold: `EXTENSION-TREE` v4.4 gains the `put`/`set` Appendix A rows — **the two core data
operations the protocol runs on had no row in the only error-code table their extension has.**

### Fixed — a domain code at a shared status is not a synonym, and the half-open state is named (0.8.2.8)

- ⛔ **§9.1's `501` synonym list wrongly named `unsupported_mode`.** `EXTENSION-REGISTRY` §6a.9.2
  normatively `MUST`s `501 unsupported_mode` for live registration under a stored domain-control
  policy, in a paragraph that says it is pinned deliberately because four codes were plausible.
  **That is not this row**: the registry handler **is** registered and `register` **is** implemented,
  and the refusal is about the mode of a stored policy. **A synonym is a second spelling of the same
  failure; a domain code for a different failure at the same status is not one.** The list now states
  the test — **the failure named, never the status shared.** The `0.8.2.7` list had been built from a
  census of what peers emit, and a census cannot tell you which of those spellings the corpus
  **mandates**.
- **§3.3's `500` row gains an enumerated more-specific set**, symmetric with `400` and `403`.
  `io_error` (an OS I/O operation failed) and `storage_error` (a content-store or tree bind/read
  failed) are **distinct conditions**, and they go in §3.3 rather than a per-extension appendix
  because **nine extensions emit one of them.** Censused by subsystem, the implementations already
  agree on the first condition; the apparent divergence dissolves once the condition boundary is
  drawn.
- **The half-open connection is named in §4.7.** Post-`hello`, pre-`authenticate` is **not
  established**, so the out-of-order row already governs an unauthenticated `ping` there and the
  answer is **`409`**. Stated rather than given a new row, because two adjacent rules each *look* like
  they cover it: the `invalid_nonce` row is scoped to a **pre-`hello`** `authenticate`, and the
  pre-authorized-connect exception is scoped to an **established** connection. All three ground-up
  implementations answer `409` by construction; this pins it by text.

### Changed — the default-code column is mandatory for the generic case (0.8.2.7)

**Ruled `MANDATORY` at every row that names a default**, and the force was already landed one
paragraph below: §3.3's authorization-path code discipline binds exactly this regime for `401`/`403` —
a default for the generic case, a more-specific code only where **defined**, and `MUST NOT` mint a
catch-all. It is now stated for the table it sits under, rather than for the authorization path alone.
**Advisory is refuted**: the generic case is the one with no other information in it, so it is
precisely the case a caller cannot branch on unless the spelling is fixed.

⭐ **The unit is the CODE SLOT, never a spelling.** `0.8.2.6` retired `unknown_operation` from the
`501` slot and reported the class closed; **`not_implemented` survives everywhere, beside
`not_supported`, `unsupported_mode`, `not_available` and `domain_control_unsupported`.** **A
token-scoped sweep reports done while the slot stays divergent.**

⛔ **The `500` row's parenthetical is struck.** It claimed *"three implementations had independently
converged on `internal_error`"*. **Censused by slot, no implementation converged** — one carries 16
spellings at `500`, another 15, another about 18. The evidence had been a count of the expected token
in three trees, published as a distribution, and it was **false about all three**. The default code
stands: it was derived from the table's structure, never from what the cohort emits. *(A build-state
claim inside normative text is what the "specification text is not a log" rule exists to prevent, and
it got past that rule by arriving as rationale.)*

**The `404` row is scoped.** `handler_not_found` was already normative in five homes before `0.8.2.6`
tabulated it. The row now names §6.2 as the defining home and **excludes the in-handler case** — an
absent entity, binding or hash **inside a registered handler** is a domain outcome, so nobody sweeps a
tree handler.

**Satisfaction mode is stated per row, because the `500` row cannot carry one**: a conformant peer
cannot be made to fail internally on demand over the wire, so that row is satisfied by **source
audit**, not by a wire check.

### Fixed — §3.3 named a default code for three statuses and left three bare (0.8.2.6)

Two folds in one revision, so the cohort vendors once.

**The `501` code split.** Implementations divided between `unknown_operation` and
`unsupported_operation` — **and the STATUS differed with it, `400` versus `501`** — so a client keying
on `result.data.code` got a different remedy depending on which peer refused. One implementation's
reachability path already had to accept **both spellings at both statuses**, with a source comment
naming the divergence outright: a measured cross-implementation split, recorded in a comment and never
routed.

**The cause is structural, and it is the half worth fixing.** §3.3 named a default code for
`400`/`401`/`403` and **left `404`/`500`/`501` bare**, so an implementer at a `501` site had nothing to
look up where every other default code lives. They reach for §6.2, read a capability-handler heading,
conclude it is scoped, and **mint**. So: **§3.3's `404`/`500`/`501` rows carry their defaults, §6.2's
sentence is marked general with §3.3 named as its authority, and `unknown_operation` is retired.**
Nothing is invented — `501` and `404` were already in the corpus.

**Two connect-path orderings are pinned:**

1. **A pre-establishment foreign-namespace EXECUTE answers `400 invalid_request`, not `401`.** A `401`
   directs the caller to authenticate and retry, **and that retry cannot succeed at any authentication
   state** — the remedy-selection failure `0.8.2.4`'s own split was made to prevent.
2. **`system/protocol/connect` stays pre-authorized after establishment.** §3.3 excepts the connection
   path from author/capability **with no state qualifier**, and §5.1 is scoped to *"authenticated
   EXECUTE"* — the class that exception defines. §5.1 is **not** the general rule with §4.2 as its
   exception; it is the other way round.

### Fixed — a pre-establishment EXECUTE had a rule nobody could find from the connection vocabulary (0.8.2.5)

**This was reported as an unruled code. It is not unruled.** §4.2's third pre-authorization rule
already governs a non-connect EXECUTE arriving before the handshake completes — *"a missing or
unverifiable author or signature is auth-class `401`"* — and §5.2a gives the code,
**`authentication_failed`**. `0.8.1` put that discriminator there, replacing the bullet's blanket
`403`.

**So the three-way measurement is a conformance gap two releases old, not a design question**: two
implementations emit the blanket `403` that `0.8.1` retired, one emits a `400`, and the two codes
standing in for the rule (`connection_required`, `handshake_failed`) **appear nowhere in either
specification document.**

**The remedy is a note, not a row.** Every row of §4.7 describes a **connect** operation; this input is
not a connection-handshake failure, so a row would put §4.7 and §4.2 into exactly the two-registries
state the precedence clause forbids. **What the note fixes is findability**: §4.7 is the table an
implementer is reading when the input arrives, and §4.2 states the rule in the vocabulary of
**pre-authorization**, which is not searchable from the vocabulary of **connection state**.

**No new obligation and no flag day**: this changes what a peer emits, never what it accepts, and
nothing branches on the code.

### Fixed — the connect surface reconciled, and the absent-`protocols` arm goes the other way (0.8.2.4)

- **Row 3 retired.** `incompatible_key_type` described an intersection model §4.5 no longer uses.
- **Row 1 names its trigger.**
- **Row 5 scoped to §1.2 ingest**, with an explicit *"a conformance check MUST NOT treat this as a
  handshake obligation."*
- **Row 10 split** — a **state** conflict is `409`, matching `connection_already_established` directly
  above it; an **unknown operation** is `400 invalid_request`, now declared at core level with its
  class defined.
- **§4.6's numbering is a normative order**, with the constraint placed on the **emitted pair** rather
  than on the internal sequence, so a peer may still check cheaply first.
- **Four §9.1 rows**, so the new obligations have somewhere a check can read them.
- **§4.5 names §8.4's identifiers inline.** One implementation advertised a stale protocol identifier
  from its first commit, **undetectable for the life of the project because the field was read by
  nothing** — which is the general sentence the edit carries.
- ⛔ **Absent or empty `protocols` is `400 invalid_request`, not unconstrained** — ruled against both
  filing recommendations. **The measurement neither filing had is one tier away**: generated peers
  already require the field before intersecting it, and pass conformance doing it, so the permissive
  reading was never the cohort default and ruling it would have made passing peers **non-conformant**.
  Both filings reached the half that matters on their own — **`incompatible_protocol` cannot be told to
  a caller that named no version.**

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

**No wire renumber, no new opcode** — the locked wire core is untouched ([ADR-0002]). Raised by an
implementation (with two green tests) after `SPECIFICATION-FORMAT.md` §8.4.6 pinned `{peer_id_hex}` to the
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
this surfaced by reading, in an implementation, and would have kept not-failing.

### Changed — spec amendment 0.8.1 (cross-substrate hardening, before-freeze)

Surfaced by the cross-substrate conformance sweep across the generated-peer cohort (findings F31–F48 and the associated hand-offs).
**No wire renumber, no new opcode, no V8 semantic change** — the wire core stays locked ([ADR-0002]). Applied to
`ENTITY-CORE-PROTOCOL.md` (§4.2, §4.4, §4.6, §4.8, §5.2, §6.11, §6.7, §3.5, §3013) and `ENTITY-NATIVE-TYPE-SYSTEM.md`
(§4.4 table, §10.1 / Appendix B refs). **Per the conformance review, these split by validation requirement**
(HANDOFF-TO-ARCH-2026-07-27-0.8.1-ratify-preconditions):

**Bucket A — spec repair (a conformant peer passes unchanged; the spec contradicted itself, peers were already correct):**

- **F32 (§4.2/§4.4):** missing/unverifiable `author` is auth-class **401**, not a blanket 403 (reconciled with the §5.2a discriminator). Surfaced by two generated peers as a spec self-contradiction — peers already mapped `AUTHN_FAIL`→401.
- **UN-b/F48 (§4.6):** key_type-support validation hoisted to an ordered **step 0**, before identity binding → `400 unsupported_key_type`. Existing vectors (`AGILITY-UNKNOWN-1`, `NEGOTIATE-KEYTYPE-1`) already gate the ordering; `unsupported_key_type` present in 38/41 peers.
- **UN-a/F47 (§3013):** disambiguate the policy-path `{peer_pattern}` (hex-closed) from the capability `peers:` scope patterns (Base58); the proposed "add Base58 to §3013" was rejected.
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
cohort re-run. The determination for findings F31–F46 and the named hand-offs is recorded in the Entity system architecture corpus.

### Added — non-interactive freshness knobs, W7 (0.8.1 — same cohort re-run)

Additive, deployment-elected freshness knobs — the §4.10 "declare and enforce a finite bound; the value is the
deployment's call" pattern applied to freshness. No wire renumber, no new opcode. Applied to `ENTITY-CORE-PROTOCOL.md`:

- **Knob 2 (§2881 / §5.1):** a deployment-declared, honored `revocation_propagation_bound` — makes the revocation exposure window `min(TTL_granularity, B)` reason-about-able (ratifies the "sync-latency-bounded" convention).
- **Knob 3 (§2966 / §5.10):** a declared cross-clock skew-tolerance `δ` — the relayed validity window is `[not_before − δ, expires_at + δ]`, declared not silent (`δ = 0` reproduces today's behavior; a *tolerance*, not clock sync).
- **Knob 1 (§210, security consideration):** the informed-deferral note — a relayed cap is authentic + authorized but not proven fresh; replay is bounded by `min(TTL, revocation_bound) ± δ` and defended primarily by handler idempotency. The strong mechanism (challenge-response over a relayed circuit) is a **deferred future extension** (rides RELAY Mode C).

Design: `EXPLORATION-NON-INTERACTIVE-FRESHNESS-AND-ANTI-REPLAY` + `ANALYSIS-NON-INTERACTIVE-FRESHNESS-CRITICALITY`, in the Entity system architecture corpus.

### Changed — cross-peer continuation bound (0.8.1; folds the built+green-3-way continuation/bounds work)

The continuation/network runtime is built and conformance-green three-way; these core-protocol deltas fold the
spec text up to the shipped wire. `chain_depth` is additive (MUST-ignore-unknown, beside `cascade_depth`) — no
wire renumber. Applied to `ENTITY-CORE-PROTOCOL.md`:

- **§3.11 `system/bounds`:** add the `chain_depth` field (all 3 impls ship it) — the deterministic causal-chain-length brake, inherited across the wire like `cascade_depth`, distinct from ttl/budget. Pin `chain_id` to a **single path segment** (was a UUID default with no format; marker path-safety depends on it).
- **§5.9 (Ruling 1):** pin cross-peer TTL — the resource backstop is decremented **once per dispatch (incl. sub-dispatches), never double-counted** at ingress *and* forward. Fixes the 9-vs-64 cross-impl divergence (Rust seeded no TTL; Python double-counted).
- **§5.9 (Ruling 2):** de-confound the magnitudes — the two MUST be **distinct** with `chain_depth` ceiling ≤ `ttl` seed (equal magnitudes let TTL mask the deterministic depth brake). **8× (seed 512) is the recommended default**, measured three-way 2026-07-27; a deployment MAY retune it for its fan-out. The conformance requirement is the *property* (the depth brake, not TTL, terminates a runaway), not the number — per the §4.10 doctrine.
- **§4.10(b) (Ruling 3):** disambiguate the reason codes — `chain_depth_exceeded` (400) is the **capability**-chain limit; the **continuation** causal-depth brake suspends with `bounds_exceeded` (429). Two mechanisms MUST NOT share a reason string; two implementations were colliding on one.

Governing record: `PROPOSAL-CONTINUATION-BOUNDS-PROPAGATION` (rev.2026-07-19), in the Entity system architecture corpus. The CONTINUATION-extension-side reconciliations (§3.6 step-6 refill→decrement, §3.7 resume-roots-depth-0, §3.9 wired framing, §6.2 monotonic clause) fold in place in `EXTENSION-CONTINUATION.md`.
