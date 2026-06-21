# Entity Core Protocol — Implementation Specification

**Version**: 0.8.0
**Status**: Active
**Source**: ENTITY-CORE-PROTOCOL.md, ENTITY-NATIVE-TYPE-SYSTEM.md, ENTITY-CBOR-ENCODING.md, EXTENSION-*.md
**Purpose**: Complete machine-readable specification for generating a conforming peer. No rationale. No history.

---

## 1. Primitives

### 1.1 Entity

```
Entity := { type: string, data: any, content_hash: bytes }
```

`content_hash` always present. Computed from `{type, data}` only.

### 1.2 Content Hash

```
system/hash := bytes  ; format_code(1) + digest(N)

content_hash(entity):
  encoded = ecf_encode({type: entity.type, data: entity.data})
  digest = SHA256(encoded)
  return bytes([0x00]) + digest    ; 33 bytes

hash_equals(a, b):
  return a == b                    ; byte-wise
```

Format codes: `0x00` = ECFv1-SHA-256 (33 bytes, REQUIRED). `0x01` = ECFv1-SHA-384 (49 bytes). `0x02` = ECFv1-SHA-512 (65 bytes). Display: `"ecfv1-sha256:" + hex(hash[1:])`.

### 1.3 ECF (Entity Canonical Form)

Deterministic CBOR per RFC 8949 §4.2:

1. Map keys sorted by encoded byte length, then lexicographically
2. Minimal integer encoding
3. Definite lengths only
4. Shortest float preserving value
5. No duplicate map keys
6. All field values preserved — no omissions

CBOR tags: Protocol messages MUST NOT use CBOR tags on data fields. Tags cause hash divergence — tagged and untagged values produce different ECF bytes, different content hashes. If a received entity contains tags, MUST preserve them when forwarding (entity fidelity).

Optional fields: SHOULD be absent (key not present) rather than null. Absent ≠ null (different bytes, different hashes). MUST NOT reject null on optional fields. MUST preserve null when forwarding.

### 1.4 Identity

```
KeyPair := { private_key: bytes[32], public_key: bytes[32] }  ; Ed25519

derive_peer_id(public_key):
  digest = SHA256(public_key)
  data = bytes([0x01, 0x01]) + digest    ; key_type=Ed25519, hash_type=SHA256
  return base58_encode(data)             ; Bitcoin alphabet, 46 chars

is_peer_id(segment):
  return len(segment) >= 46 and all chars in BASE58_ALPHABET  ; 46 = minimum (Ed25519+SHA256)
```

Base58 alphabet: `123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz`

### 1.5 Signatures

```
sign_entity(entity, private_key):
  return ed25519_sign(private_key, entity.content_hash)   ; sign full 33 bytes

verify_entity_signature(entity, public_key, signature_bytes):
  return ed25519_verify(public_key, entity.content_hash, signature_bytes)
```

Signature model: signatures point TO content. Scan `included` for `system/signature` where `data.target == target_hash`. `data.signer` MUST match expected identity hash.

### 1.6 Wire Frame

```
Frame := [4 bytes: length (big-endian u32)] [length bytes: CBOR payload]
```

Default frame limit: 16 MiB SHOULD. CBOR processing limits (SHOULD, configurable): max nesting depth 64 levels, max map keys 65536.

### 1.7 Storage

```
Entity Tree:   path → hash    ; mutable location index
Content Store: hash → entity  ; immutable, deduplicated
```

Path may be simultaneously bound to entity AND have child paths. No file-or-directory constraint.

### 1.8 Entity Fidelity

1. Validate hash on receipt (compute from {type, data}, compare)
2. Trust validated hash thereafter — MUST NOT recompute
3. Store original bytes
4. Forward original — MUST NOT re-serialize
5. SHOULD preserve unknown fields

**Property vs. mechanism (clarification).** The cross-impl-observable property is: *a forwarded entity re-presents the exact bytes that hash to its validated content hash, and all received content — known fields, unknown fields, CBOR tags, null-vs-absent — survives the round-trip.* Steps 3–4 (store original bytes, forward without re-serializing) are the **unconditionally robust** mechanism for guaranteeing it. An implementation MAY instead carry the validated hash (step 2) and re-encode canonically on forward **iff** (a) receipt validation is a strict ECF re-encode-and-compare — so an accepted entity's canonical encoding provably equals its received bytes — and (b) its parsed representation is lossless at every nesting level, so the re-encode reproduces unknown content. Under (a)+(b) the re-encode is byte-faithful and the property holds. What an implementation MUST NOT do is recompute the hash from a *lossy* parse without carrying the validated hash: that silently diverges the moment encoding is non-deterministic across impls/versions or an unmodeled (forward-compatibility) field appears — the exact hazard this contract exists to prevent. Carrying the validated hash (step 2) is required either way; raw-byte storage vs. lossless-parse-plus-canonical-re-encode is an implementation choice.

Conformance of the mechanism (whichever an implementation chooses) is verified by the test-vector appendix in `ENTITY-CBOR-ENCODING.md` Appendix E. Re-encode-and-compare implementations MUST additionally verify that round-tripping each `encode_equal` vector's `canonical` bytes through their decoder and re-encoder produces byte-identical output — the (a) precondition above made testable.

---

## 2. URI and Path Model

Absolute paths start with `/`: `/{peer_id}/rest`. Peer-relative paths (no leading `/`) resolve to `/{local_peer_id}/rest`. Reserved: `./` and `../` (MUST reject). Storage: absolute paths only.

```
URI := "entity://" peer_id "/" path

normalize(uri):
  if uri starts with "entity://": return "/" + uri[9:]
  return uri

dispatch_path(uri, local_peer_id):
  normalized = normalize(uri)
  canonical = canonicalize(normalized, local_peer_id)
  validate_absolute_path(canonical)                    ; MUST
  return canonical

canonicalize(path, local_peer_id):
  if path starts with "./" or path starts with "../": return error("reserved")
  if path starts with "*/": return error("ambiguous: use /*/rest")
  if path starts with "/": return path
  return "/" + local_peer_id + "/" + path

validate_absolute_path(path):
  ; MUST be called on all tree paths. NOT called on patterns.
  if not starts_with(path, "/"): return error("not absolute")
  if not is_peer_id(split(path[1:], "/")[0]): return error("invalid peer_id")
  return ok
```

Path rules: UTF-8, no null bytes, no empty segments, no trailing `/` on bindings. Absolute paths MUST start with `/`. Peer-relative MUST NOT.

Pattern forms in entity data (peer-relative, canonicalized to absolute before matching):
- `*` → `/{local}/*` (local peer, all paths)
- `pattern/*` → `/{local}/pattern/*` (local peer, subtree)
- `pattern` → `/{local}/pattern` (local peer, exact)
- `/*/*` (all peers, all paths)
- `/*/pattern/*` (all peers, subtree)
- `/{P}/pattern/*` (specific peer, subtree)

Bare `*/rest` is rejected. Peer wildcard MUST use `/*/rest`.

```
matches_pattern(path, pattern):
  ; Both MUST be canonicalized (absolute) before calling.
  if pattern == "*": return true
  if pattern starts with "/*/":
    remainder = pattern[3:]
    second_slash = index_of(path[1:], "/")
    if second_slash < 0: return false
    path_rest = path[second_slash + 2:]
    return matches_pattern(path_rest, remainder)
  if pattern ends with "/*":
    prefix = pattern without trailing "*"
    return path starts with prefix
  return path == pattern
```

Dispatch routing: fully qualified URI → normalize (strips `entity://`, prepends `/`) → canonicalize → absolute path. Check peer_id segment = local → dispatch. Non-local → reject 400 (inbound) or route outbound (internal). Peer-relative → canonicalize → dispatch.

---

## 3. Type Registry

### 3.1 Bootstrap Types (14)

```
primitive/string  := { name: "primitive/string" }                    ; CBOR major type 3
primitive/bytes   := { name: "primitive/bytes" }                     ; CBOR major type 2
primitive/uint    := { name: "primitive/uint" }                      ; CBOR major type 0
primitive/int     := { name: "primitive/int" }                       ; CBOR major type 0 or 1
primitive/float   := { name: "primitive/float" }                     ; CBOR major type 7 float
primitive/bool    := { name: "primitive/bool" }                      ; CBOR true/false
primitive/null    := { name: "primitive/null" }                      ; CBOR null
primitive/any     := { name: "primitive/any" }                       ; any CBOR

system/hash       := { name: "system/hash", extends: "primitive/bytes" }  ; 33 bytes (0x00 + SHA-256)
system/tree/path  := { name: "system/tree/path", extends: "primitive/string" }  ; tree location
system/type/name  := { name: "system/type/name", extends: "primitive/string" }  ; type identifier
system/identity/peer-id := { name: "system/identity/peer-id", extends: "primitive/string" }  ; peer identity

system/type := {
  name: "system/type",
  fields: {
    name:        {type_ref: "system/type/name"},
    extends:     {type_ref: "system/type/name", optional: true},
    fields:      {map_of: {type_ref: "system/type/field-spec"}, optional: true},
    layout:      {array_of: {type_ref: "primitive/string"}, optional: true},
    type_params: {array_of: {type_ref: "primitive/string"}, optional: true},
    type_args:   {map_of: {type_ref: "system/type/name"}, optional: true}
  }
}

system/type/field-spec := {
  name: "system/type/field-spec",
  fields: {
    type_ref:   {type_ref: "system/type/name", optional: true},
    optional:   {type_ref: "primitive/bool", optional: true},
    array_of:   {type_ref: "system/type/field-spec", optional: true},
    map_of:     {type_ref: "system/type/field-spec", optional: true},
    union_of:   {array_of: {type_ref: "system/type/field-spec"}, optional: true},
    type_param: {type_ref: "primitive/string", optional: true},
    type_args:  {map_of: {type_ref: "system/type/name"}, optional: true},
    default:    {type_ref: "primitive/any", optional: true},
    key_type:   {type_ref: "system/type/name", optional: true},
    byte_size:  {type_ref: "primitive/uint", optional: true}
  }
}
```

Field-spec invariant: exactly one of `type_ref | array_of | map_of | union_of | type_param`. Recursive.

Type inheritance: single parent via `extends`. Child inherits all fields. MUST NOT redefine parent fields. MAY add new fields. `extends` determines encoding — if extends primitive, encodes as that primitive.

Open types: unknown fields MUST be preserved. MUST NOT reject.

### 3.2 Protocol Types

```
system/protocol/envelope := {
  fields: {
    root:     {type_ref: "primitive/any"},
    included: {map_of: {type_ref: "primitive/any"}, key_type: "system/hash"}
  }
}

system/protocol/execute := {
  fields: {
    request_id: {type_ref: "primitive/string"},
    uri:        {type_ref: "system/tree/path"},
    operation:  {type_ref: "primitive/string"},
    resource:   {type_ref: "system/protocol/resource-target", optional: true},
    params:     {type_ref: "primitive/any"},
    bounds:     {type_ref: "system/bounds", optional: true},
    deliver_to: {type_ref: "system/delivery-spec", optional: true},
    author:     {type_ref: "system/hash", optional: true},
    capability: {type_ref: "system/hash", optional: true}
  }
}

system/protocol/execute/response := {
  fields: {
    request_id: {type_ref: "primitive/string"},
    status:     {type_ref: "primitive/uint"},
    result:     {type_ref: "primitive/any"}
  }
}

system/protocol/error := {
  fields: {
    code:    {type_ref: "primitive/string"},
    message: {type_ref: "primitive/string", optional: true}
  }
}

system/protocol/resource-target := {
  fields: {
    targets: {array_of: {type_ref: "system/tree/path"}},
    exclude: {array_of: {type_ref: "system/tree/path"}, optional: true}
  }
}
```

`params` and `result` are entities: `{type, data, content_hash}`, not raw values. `author` and `capability` REQUIRED on all EXECUTE except `system/protocol/connect`. Only 2 wire message types: EXECUTE + EXECUTE_RESPONSE. Anything else → close connection.

### 3.3 Identity & Security Types

```
system/peer := {
  fields: {
    peer_id:    {type_ref: "system/peer-id"},
    public_key: {type_ref: "primitive/bytes"},
    key_type:   {type_ref: "primitive/string"}
  }
}

system/signature := {
  fields: {
    target:    {type_ref: "system/hash"},
    signer:    {type_ref: "system/hash"},
    algorithm: {type_ref: "primitive/string"},
    signature: {type_ref: "primitive/bytes"}
  }
}

system/capability/path-scope := {
  fields: {
    include: {array_of: {type_ref: "system/tree/path"}},
    exclude: {array_of: {type_ref: "system/tree/path"}, optional: true}
  }
}

system/capability/id-scope := {
  fields: {
    include: {array_of: {type_ref: "primitive/string"}},
    exclude: {array_of: {type_ref: "primitive/string"}, optional: true}
  }
}

system/capability/grant-entry := {
  fields: {
    handlers:    {type_ref: "system/capability/path-scope"},
    resources:   {type_ref: "system/capability/path-scope"},
    operations:  {type_ref: "system/capability/id-scope"},
    peers:       {type_ref: "system/capability/id-scope", optional: true},
    constraints: {type_ref: "primitive/any", optional: true}
  }
}

system/capability/delegation-caveats := {
  fields: {
    no_delegation:        {type_ref: "primitive/bool", optional: true},
    max_delegation_depth: {type_ref: "primitive/uint", optional: true},
    max_delegation_ttl:   {type_ref: "primitive/uint", optional: true}
  }
}

system/capability/multi-granter := {
  fields: {
    signers:   {array_of: {type_ref: "system/hash"}},
    threshold: {type_ref: "primitive/uint"}
  }
}
; Multi-signature granter for K-of-N root caps (§3.6).
; Inline in the cap data; root-only (parent: null) per M3.

system/capability/token := {
  fields: {
    grants:             {array_of: {type_ref: "system/capability/grant-entry"}},
    granter:            {union_of: [
                           {type_ref: "system/hash"},                     ; single-sig
                           {type_ref: "system/capability/multi-granter"}  ; multi-sig root (M1/M2)
                         ]},
    grantee:            {type_ref: "system/hash"},
    parent:             {type_ref: "system/hash", optional: true},        ; null when granter is multi-granter (M3)
    created_at:         {type_ref: "primitive/uint"},
    expires_at:         {type_ref: "primitive/uint", optional: true},
    not_before:         {type_ref: "primitive/uint", optional: true},
    delegation_caveats: {type_ref: "system/capability/delegation-caveats", optional: true},
    resource_limits:    {type_ref: "system/resource-limits", optional: true}
  }
}

system/capability/grant := {
  fields: { token: {type_ref: "system/hash"} }
}

system/capability/request := {
  fields: {
    grants: {array_of: {type_ref: "system/capability/grant-entry"}},
    ttl_ms: {type_ref: "primitive/uint", optional: true}
  }
}

system/capability/revocation := {
  fields: {
    token:  {type_ref: "system/hash"},
    reason: {type_ref: "primitive/string", optional: true}
  }
}
```

Optional field defaults: `parent` absent = root capability. `expires_at` absent = no expiration. `not_before` absent = immediately valid. `delegation_caveats` absent = unrestricted. `peers` absent = `{include: [local_peer_id]}`.

### 3.4 Handler Types

```
system/handler := {
  fields: {
    pattern:        {type_ref: "system/tree/path"},
    name:           {type_ref: "primitive/string"},
    operations:     {map_of: {type_ref: "system/handler/operation-spec"}},
    max_scope:      {array_of: {type_ref: "system/capability/grant-entry"}, optional: true},
    internal_scope: {array_of: {type_ref: "system/capability/grant-entry"}, optional: true}
  }
}

system/handler/operation-spec := {
  fields: {
    input_type:  {type_ref: "system/type/name", optional: true},
    output_type: {type_ref: "system/type/name", optional: true}
  }
}

system/handler/interface := {
  fields: {
    pattern:    {type_ref: "system/tree/path"},
    name:       {type_ref: "primitive/string"},
    operations: {map_of: {type_ref: "system/handler/operation-spec"}}
  }
}

system/handler/register-request := {
  fields: {
    manifest:        {type_ref: "system/handler"},
    types:           {map_of: {type_ref: "system/type"}, optional: true},
    requested_scope: {array_of: {type_ref: "system/capability/grant-entry"}, optional: true}
  }
}

system/handler/register-result := {
  fields: {
    pattern: {type_ref: "system/tree/path"},
    grant:   {type_ref: "system/capability/token"}
  }
}

system/handler/unregister-request := {
  fields: { pattern: {type_ref: "system/tree/path"} }
}
```

### 3.5 Connection Types

```
system/protocol/connect/hello := {
  fields: {
    peer_id:      {type_ref: "system/identity/peer-id"},
    nonce:        {type_ref: "primitive/bytes"},
    protocols:    {array_of: {type_ref: "primitive/string"}},
    timestamp:    {type_ref: "primitive/uint"},
    hash_formats: {array_of: {type_ref: "primitive/string"}, optional: true},
    key_types:    {array_of: {type_ref: "primitive/string"}, optional: true},
    compression:  {array_of: {type_ref: "primitive/string"}, optional: true},
    encryption:   {array_of: {type_ref: "primitive/string"}, optional: true}
  }
}

system/protocol/connect/authenticate := {
  fields: {
    peer_id:    {type_ref: "system/identity/peer-id"},
    public_key: {type_ref: "primitive/bytes"},
    key_type:   {type_ref: "primitive/string"},
    nonce:      {type_ref: "primitive/bytes"}
  }
}
```

Defaults: `hash_formats` absent → `["ecfv1-sha256"]`. `key_types` absent → `["ed25519"]`.

### 3.6 Tree Types

```
system/tree/listing-entry := {
  fields: {
    hash:         {type_ref: "system/hash", optional: true},
    has_children: {type_ref: "primitive/bool"}
  }
}

system/tree/listing := {
  fields: {
    path:      {type_ref: "system/tree/path"},
    entries:   {map_of: {type_ref: "system/tree/listing-entry"}},
    count:     {type_ref: "primitive/uint"},
    offset:    {type_ref: "primitive/uint"},
    next_page: {type_ref: "system/hash", optional: true}
  }
}

system/tree/get-request := {
  fields: {
    tree_id: {type_ref: "primitive/string", optional: true},
    mode:    {type_ref: "primitive/string", optional: true},
    limit:   {type_ref: "primitive/uint", optional: true},
    offset:  {type_ref: "primitive/uint", optional: true}
  }
}

system/tree/put-request := {
  fields: {
    entity:  {type_ref: "primitive/any", optional: true},
    tree_id: {type_ref: "primitive/string", optional: true}
  }
}

system/type/validate-request := {
  fields: {
    entity:    {type_ref: "primitive/any"},
    type_name: {type_ref: "system/type/name"}
  }
}

system/type/validate-result := {
  fields: {
    valid:  {type_ref: "primitive/bool"},
    errors: {array_of: {type_ref: "primitive/string"}, optional: true}
  }
}
```

### 3.7 Bounds & Resource Types

```
system/bounds := {
  fields: {
    ttl:     {type_ref: "primitive/uint", optional: true},
    budget:  {type_ref: "primitive/uint", optional: true},
    chain_id:{type_ref: "primitive/string", optional: true},
    visited: {array_of: {type_ref: "system/tree/path"}, optional: true}
  }
}

system/delivery-spec := {
  fields: {
    uri:       {type_ref: "system/tree/path"},
    operation: {type_ref: "primitive/string", optional: true}
  }
}

system/resource-limits := {
  fields: {
    max_budget:         {type_ref: "primitive/uint", optional: true},
    max_ttl:            {type_ref: "primitive/uint", optional: true},
    max_visited_length: {type_ref: "primitive/uint", optional: true}
  }
}
```

### 3.8 Tree Extension Types

```
system/tree/snapshot := {
  fields: {
    prefix:   {type_ref: "primitive/string"},
    bindings: {map_of: {type_ref: "system/hash"}}
  }
}

system/tree/snapshot-request := {
  fields: {
    prefix:  {type_ref: "primitive/string", optional: true},
    tree_id: {type_ref: "primitive/string", optional: true}
  }
}

system/tree/diff := {
  fields: {
    base:      {type_ref: "system/hash"},
    target:    {type_ref: "system/hash"},
    added:     {map_of: {type_ref: "system/hash"}},
    removed:   {map_of: {type_ref: "system/hash"}},
    changed:   {map_of: {type_ref: "system/tree/diff/change"}},
    unchanged: {type_ref: "primitive/uint"}
  }
}

system/tree/diff/change := {
  fields: {
    base_hash:   {type_ref: "system/hash"},
    target_hash: {type_ref: "system/hash"}
  }
}

system/tree/diff-request := {
  fields: {
    base:   {type_ref: "system/hash"},
    target: {type_ref: "system/hash"}
  }
}

system/tree/merge-request := {
  fields: {
    source:        {type_ref: "system/hash"},
    target_tree:   {type_ref: "primitive/string", optional: true},
    strategy:      {type_ref: "primitive/string", optional: true},
    source_prefix: {type_ref: "primitive/string", optional: true},
    target_prefix: {type_ref: "primitive/string", optional: true},
    dry_run:       {type_ref: "primitive/bool", optional: true}
  }
}

system/tree/merge-result := {
  fields: {
    applied:   {type_ref: "primitive/uint"},
    skipped:   {type_ref: "primitive/uint"},
    conflicts: {map_of: {type_ref: "system/tree/merge-result/conflict"}},
    strategy:  {type_ref: "primitive/string"}
  }
}

system/tree/merge-result/conflict := {
  fields: {
    existing_hash: {type_ref: "system/hash"},
    incoming_hash: {type_ref: "system/hash"},
    resolution:    {type_ref: "primitive/string"}
  }
}

system/tree/extract-request := {
  fields: {
    prefix:  {type_ref: "primitive/string"},
    tree_id: {type_ref: "primitive/string", optional: true},
    paths:   {array_of: {type_ref: "primitive/string"}, optional: true}
  }
}

system/tree/config := {
  fields: {
    tree_id:        {type_ref: "primitive/string"},
    root_structure: {type_ref: "primitive/string"},
    purpose:        {type_ref: "primitive/string", optional: true},
    ephemeral:      {type_ref: "primitive/bool", optional: true},
    source:         {type_ref: "primitive/string", optional: true},
    capability:     {type_ref: "system/hash", optional: true}
  }
}
```

Merge strategies: `no-overwrite` (default), `source-wins`, `target-wins`. Merge is additive — does not remove absent paths.

### 3.9 Inbox & Subscription Types

```
system/protocol/inbox/delivery := {
  fields: {
    original_request_id: {type_ref: "primitive/string"},
    status:              {type_ref: "primitive/uint"},
    result:              {type_ref: "primitive/any"}
  }
}

system/protocol/inbox/notification := {
  fields: {
    subscription_id: {type_ref: "primitive/string"},
    event:           {type_ref: "primitive/string"},
    uri:             {type_ref: "system/tree/path"},
    hash:            {type_ref: "system/hash", optional: true},
    previous_hash:   {type_ref: "system/hash", optional: true}
  }
}

system/subscription := {
  fields: {
    subscription_id:     {type_ref: "primitive/string"},
    pattern:             {type_ref: "system/tree/path"},
    events:              {array_of: {type_ref: "primitive/string"}},
    deliver_uri:         {type_ref: "system/tree/path"},
    deliver_operation:   {type_ref: "primitive/string"},
    subscriber_identity: {type_ref: "system/hash"},
    deliver_token:       {type_ref: "system/hash"},
    created_at:          {type_ref: "primitive/uint"},
    limits:              {type_ref: "system/subscription/limits", optional: true}
  }
}

system/subscription/request := {
  fields: {
    events:         {array_of: {type_ref: "primitive/string"}, optional: true},
    deliver_to:     {type_ref: "system/delivery-spec"},
    deliver_token:  {type_ref: "system/hash"},
    limits:         {type_ref: "system/subscription/limits", optional: true}
  }
}
; Subscription pattern comes from EXECUTE resource.targets, not params.

system/subscription/limits := {
  fields: {
    max_events:          {type_ref: "primitive/uint", optional: true},
    max_duration_ms:     {type_ref: "primitive/uint", optional: true},
    rate_limit:          {type_ref: "primitive/uint", optional: true},
    notification_budget: {type_ref: "primitive/uint", optional: true}
  }
}

system/subscription/cancel := {
  fields: { subscription_id: {type_ref: "primitive/string"} }
}
```

Extension field on EXECUTE: `deliver_token: {type_ref: "system/hash", optional: true}` — MUST be present when `deliver_to` is present.

Events: `created`, `updated`, `deleted`.

### 3.10 Compute Types

```
compute/literal   := { fields: { value: {type_ref: "primitive/any"} } }

compute/lookup    := { fields: { path: {type_ref: "primitive/string"} } }

compute/apply     := {
  fields: {
    path:      {type_ref: "system/tree/path", optional: true},
    operation: {type_ref: "primitive/string", optional: true},
    fn:        {type_ref: "system/hash", optional: true},
    args:      {map_of: {type_ref: "system/hash"}, optional: true}
  }
}

compute/if := {
  fields: {
    condition: {type_ref: "system/hash"},
    then:      {type_ref: "system/hash"},
    else:      {type_ref: "system/hash", optional: true}
  }
}

compute/let := {
  fields: {
    bindings: {array_of: {type_ref: "primitive/any"}},
    body:     {type_ref: "system/hash"}
  }
}

compute/lambda := {
  fields: {
    params: {array_of: {type_ref: "primitive/string"}},
    body:   {type_ref: "system/hash"}
  }
}

compute/closure := {
  fields: {
    params: {array_of: {type_ref: "primitive/string"}},
    body:   {type_ref: "system/hash"},
    env:    {type_ref: "system/hash", optional: true}
  }
}

compute/scope := {
  fields: { bindings: {map_of: {type_ref: "primitive/any"}} }
}

compute/arithmetic := {
  fields: {
    op:    {type_ref: "primitive/string"},
    left:  {type_ref: "system/hash"},
    right: {type_ref: "system/hash"}
  }
}

compute/compare := {
  fields: {
    op:    {type_ref: "primitive/string"},
    left:  {type_ref: "system/hash"},
    right: {type_ref: "system/hash"}
  }
}

compute/logic := {
  fields: {
    op:    {type_ref: "primitive/string"},
    left:  {type_ref: "system/hash"},
    right: {type_ref: "system/hash", optional: true}
  }
}

compute/field := {
  fields: {
    name:   {type_ref: "primitive/string"},
    entity: {type_ref: "system/hash"}
  }
}

compute/construct := {
  fields: {
    entity_type: {type_ref: "system/type/name"},
    fields:      {map_of: {type_ref: "system/hash"}}
  }
}

compute/result := {
  fields: {
    value:      {type_ref: "primitive/any"},
    expression: {type_ref: "system/hash"}
  }
}

compute/error := {
  fields: {
    code:       {type_ref: "primitive/string"},
    message:    {type_ref: "primitive/string"},
    at:         {type_ref: "primitive/string", optional: true},
    expression: {type_ref: "system/hash", optional: true}
  }
}
```

Arithmetic ops: `add`, `sub`, `mul`, `div`, `mod`. Compare ops: `eq`, `neq`, `lt`, `gt`, `lte`, `gte`. Logic ops: `and`, `or`, `not`. `compute/apply` MUST have `path` xor `fn`. `compute/if` MUST NOT evaluate non-taken branch. `compute/lookup` evaluates compute expressions found in tree (spreadsheet semantics).

### 3.11 Content Types

```
system/content/blob := {
  fields: {
    total_size:  {type_ref: "primitive/uint"},
    chunk_size:  {type_ref: "primitive/uint"},
    chunking:    {type_ref: "primitive/uint"},
    chunks:      {array_of: {type_ref: "system/hash"}}
  }
}

system/content/chunk := {
  fields: { payload: {type_ref: "primitive/bytes"} }
}

system/content/content-response := {
  fields: {
    found:   {array_of: {type_ref: "system/hash"}},
    missing: {array_of: {type_ref: "system/hash"}}
  }
}
```

Chunking values: `0` = fixed-size, `1` = FastCDC/NC2. Default chunk size: 4 MiB. No separate content hash — entity hashing provides integrity. No content-level compression — handled by wire/storage layers.

---

## 4. Algorithms

### 4.1 Request Verification (CONFORMANCE)

```
verify_request(envelope, local_peer_id):
  execute = envelope.root
  included = envelope.included
  if not hash_equals(content_hash(execute), execute.content_hash): DENY
  signature = find_signature(execute.content_hash, included)
  if signature is null: DENY
  if not hash_equals(signature.data.signer, execute.data.author): DENY
  author = included[execute.data.author]
  if author is null: DENY
  if not verify_signature(signature, author): DENY
  capability = included[execute.data.capability]
  if capability is null: DENY
  if not hash_equals(capability.data.grantee, execute.data.author): DENY
  if not verify_capability_chain(capability, included, local_peer_id): DENY
  return ALLOW

find_signature(target_hash, included):
  for hash, entity in included:
    if entity.type == "system/signature" and hash_equals(entity.data.target, target_hash):
      return entity
  return null
```

### 4.2 Permission Check (CONFORMANCE)

```
check_permission(execute, capability, handler_pattern, local_peer_id):
  operation = execute.data.operation
  target_peer = extract_peer(execute.data.uri, local_peer_id)
  resource_target = execute.data.resource

  for grant in capability.data.grants:
    if not matches_scope(operation, grant.operations, local_peer_id): continue
    if not matches_scope(handler_pattern, grant.handlers, local_peer_id): continue
    peers_scope = grant.peers or {include: [local_peer_id]}
    if not matches_scope(target_peer, peers_scope, local_peer_id): continue
    if resource_target is not null:
      if not check_resource_scope(resource_target, grant.resources, local_peer_id): continue
    return ALLOW
  return DENY

matches_scope(value, scope, local_peer_id):
  matched = false
  for pattern in scope.include:
    if matches_pattern(canonicalize(value, local_peer_id), canonicalize(pattern, local_peer_id)):
      matched = true; break
  if not matched: return false
  if scope.exclude is not null:
    for pattern in scope.exclude:
      if matches_pattern(canonicalize(value, local_peer_id), canonicalize(pattern, local_peer_id)):
        return false
  return true

check_resource_scope(resource_target, grant_resources_scope, local_peer_id):
  caller_exclude = resource_target.exclude or []
  grant_include = grant_resources_scope.include
  grant_exclude = grant_resources_scope.exclude or []
  for target in resource_target.targets:
    ct = canonicalize(target, local_peer_id)
    if not is_pattern(ct): validate_absolute_path(ct)  ; concrete paths only
    if is_covered_by(ct, caller_exclude, local_peer_id): continue
    if not is_covered_by(ct, grant_include, local_peer_id): return false
    if is_pattern(ct):
      for ge in grant_exclude:
        cge = canonicalize(ge, local_peer_id)
        if not patterns_overlap(ct, cge): continue
        if not is_covered_by(cge, caller_exclude, local_peer_id): return false
    else:
      for ge in grant_exclude:
        cge = canonicalize(ge, local_peer_id)
        if matches_pattern(ct, cge): return false
  return true

is_covered_by(path_or_pattern, pattern_set, local_peer_id):
  for p in pattern_set:
    if matches_pattern(path_or_pattern, canonicalize(p, local_peer_id)): return true
  return false

strip_wildcard(pattern):
  if pattern ends with "/*": return pattern[0 : len(pattern) - 2]
  if pattern == "*": return ""
  return pattern

patterns_overlap(a, b):
  prefix_a = strip_wildcard(a)
  prefix_b = strip_wildcard(b)
  return starts_with(prefix_a, prefix_b) or starts_with(prefix_b, prefix_a)

is_pattern(path): return path contains "*"

extract_peer(path, local_peer_id):
  ; path should be canonicalized (absolute): /{peer_id}/rest
  if path starts with "/":
    segments = split(path[1:], "/")
    if is_peer_id(segments[0]): return segments[0]
  return local_peer_id

check_path_permission(operation, path, capability, handler_pattern, local_peer_id):
  canonical_path = canonicalize(path, local_peer_id)
  for grant in capability.data.grants:
    if not matches_scope(handler_pattern, grant.handlers, local_peer_id): continue
    if not matches_scope(operation, grant.operations, local_peer_id): continue
    if not matches_scope(canonical_path, grant.resources, local_peer_id): continue
    return ALLOW
  return DENY
```

### 4.3 Delegation Chain (CONFORMANCE)

```
collect_authority_chain(cap, resolve_fn):
  chain = []
  current = cap
  depth = 0
  max_depth = 64
  while current is not null:
    if depth > max_depth: error(ChainTooDeep)
    chain.append(current)
    if current.data.parent is null: return chain
    current = resolve_fn(current.data.parent)
    if current is null: error(ChainUnreachable)
    depth += 1

verify_capability_chain(capability, included, local_peer_id):
  chain = collect_authority_chain(capability, fn(hash) { included[hash] })
  if is_error(chain): DENY
  root = chain[len(chain) - 1]
  root_granter = included[root.data.granter]
  if root_granter is null or root_granter.data.peer_id != local_peer_id: DENY
  for i in 0 .. len(chain) - 1:
    current = chain[i]
    sig = find_signature(current.content_hash, included)
    if sig is null: DENY
    granter = included[current.data.granter]
    if granter is null: DENY
    if not hash_equals(sig.data.signer, current.data.granter): DENY
    if not verify_signature(sig, granter): DENY
    t = now()
    if current.data.not_before is not null and t < current.data.not_before: DENY
    if current.data.expires_at is not null and current.data.expires_at < t: DENY
    if i < len(chain) - 1:
      parent = chain[i + 1]
      if not hash_equals(parent.data.grantee, current.data.granter): DENY
      if not is_attenuated(current, parent, local_peer_id): DENY
      if not check_delegation_caveats(parent, current, i): DENY
  root_sig = find_signature(root.content_hash, included)
  if root_sig is null: DENY
  if not verify_signature(root_sig, root_granter): DENY
  return ALLOW
```

### 4.4 Attenuation (CONFORMANCE)

```
is_attenuated(child, parent, local_peer_id):
  for child_grant in child.data.grants:
    if not grant_covered_by(child_grant, parent.data.grants, local_peer_id): return false
  if parent.data.expires_at is not null:
    if child.data.expires_at is null: return false
    if child.data.expires_at > parent.data.expires_at: return false
  return true

grant_covered_by(child_grant, parent_grants, local_peer_id):
  for parent_grant in parent_grants:
    if grant_subset(child_grant, parent_grant, local_peer_id): return true
  return false

grant_subset(child_grant, parent_grant, local_peer_id):
  if not scope_subset(child_grant.handlers, parent_grant.handlers, local_peer_id): return false
  if not scope_subset(child_grant.operations, parent_grant.operations, local_peer_id): return false
  if not scope_subset(child_grant.resources, parent_grant.resources, local_peer_id): return false
  child_peers = child_grant.peers or {include: [local_peer_id]}
  parent_peers = parent_grant.peers or {include: [local_peer_id]}
  if not scope_subset(child_peers, parent_peers, local_peer_id): return false
  return true

scope_subset(child_scope, parent_scope, local_peer_id):
  for child_pattern in child_scope.include:
    cc = canonicalize(child_pattern, local_peer_id)
    if not any(matches_pattern(cc, canonicalize(pp, local_peer_id)) for pp in parent_scope.include):
      return false
  if parent_scope.exclude is not null:
    for parent_ex in parent_scope.exclude:
      cp = canonicalize(parent_ex, local_peer_id)
      child_has = false
      if child_scope.exclude is not null:
        for child_ex in child_scope.exclude:
          if matches_pattern(cp, canonicalize(child_ex, local_peer_id)):
            child_has = true; break
      if not child_has: return false
  return true

check_delegation_caveats(parent, child, depth):
  caveats = parent.data.delegation_caveats
  if caveats is null: return true
  if caveats.no_delegation == true: return false
  if caveats.max_delegation_depth is not null:
    if depth >= caveats.max_delegation_depth: return false
  if caveats.max_delegation_ttl is not null:
    if child.data.expires_at is null: return false
    child_ttl = child.data.expires_at - child.data.created_at
    if child_ttl > caveats.max_delegation_ttl: return false
  return true
```

### 4.5 Handler Dispatch (CONFORMANCE)

```
resolve_handler(handler_path):
  segments = split(handler_path, "/")
  for i = len(segments) down to 1:
    prefix = join(segments[0..i], "/")
    entity = tree_get(prefix)
    if entity is not null and entity.type == "system/handler":
      suffix = handler_path[len(prefix):]
      return {handler: entity, pattern: prefix, suffix: suffix}
  return null
```

### 4.6 Type Resolution (REFERENCE)

```
resolve_type(type_path, type_store, namespace="default", visited={}):
  if type_path in visited: ERROR "circular inheritance"
  visited = visited + {type_path}
  type_def = type_store.lookup("system/type/" + namespace + "/" + type_path)
  if type_def is null: ERROR "unknown type"
  if type_def.data.extends is null: return type_def
  parent = resolve_type(type_def.data.extends, type_store, namespace, visited)
  merged_fields = parent.fields + type_def.fields
  if overlap(parent.fields, type_def.fields): ERROR "field redefinition"
  return {fields: merged_fields}

validate_entity(entity, resolved_type):
  errors = []
  for (name, field_spec) in resolved_type.fields:
    value = entity.data[name]
    if value is absent:
      if field_spec.default is not null: continue
      if not field_spec.optional: errors.add("missing required field: " + name)
    else: errors.addAll(validate_value(value, field_spec))
  return errors
```

Resolution depth limit: 64. Cycles MUST be detected.

---

## 5. Handler Registry

### 5.1 Bootstrap Handlers (pre-loaded)

| Pattern | Operations | Role |
|---------|-----------|------|
| `system/tree` | `get`, `put` | Tree access. Core extension adds: `snapshot`, `diff`, `merge`, `extract`, `create`, `destroy` |
| `system/handler` | `register`, `unregister` | Handler lifecycle |
| `system/type` | `validate` | Type validation |
| `system/protocol/connect` | `hello`, `authenticate` | Connection setup (pre-authorized, no auth required) |

### 5.2 Post-Bootstrap System Handlers

| Pattern | Operations | Extension |
|---------|-----------|-----------|
| `system/capability` | `request`, `delegate`, `revoke` | Core |
| `system/inbox/*` | `receive` | EXTENSION-INBOX, EXTENSION-SUBSCRIPTION |
| `system/continuation` | `advance`, `resume`, `abandon` | EXTENSION-CONTINUATION |
| `system/compute/*` | `eval` | EXTENSION-COMPUTE |
| `system/content/*` | `get` | EXTENSION-CONTENT (optional) |

### 5.3 Handler Index

```
system/handler/                              ; handler index
  {pattern}                                   ; system/handler/interface entity

system/capability/grants/                     ; handler capability grants
  {pattern}                                   ; handler's grant
```

Example: `system/handler/system/tree` = interface entity (pattern, name, operations), `system/capability/grants/system/tree` = tree handler's grant. Interface entities omit `max_scope` and `internal_scope` — operator security config is not exposed to connecting peers.

### 5.4 Registration Flow

`register`: stores manifest at pattern path, derives `system/handler/interface` entity and stores at `system/handler/{pattern}`, installs types at `system/type/*`, creates handler grant at `system/capability/grants/{pattern}`. Resource target: `{targets: ["system/handler/{pattern}"]}`.

`unregister`: reverses registration. Resource target: `{targets: ["system/handler/{pattern}"]}`.

Listing handlers: `EXECUTE system/tree operation: "get" resource: {targets: ["system/handler/"]}` (trailing slash = listing).

---

## 6. Connection Protocol

### 6.1 Flow (6 messages)

```
Initiator                           Responder
    |-- EXECUTE hello ------------------>|
    |<------------ EXECUTE_RESPONSE hello|   (responder's hello)
    |-- EXECUTE authenticate ----------->|
    |<---- EXECUTE_RESPONSE authenticate |   (initiator's capability)
    |<---------- EXECUTE authenticate ---|
    |-- EXECUTE_RESPONSE authenticate -->|   (responder's capability)
    |         Connection Established     |
```

### 6.2 Rules

- `system/protocol/connect` is sole pre-authorized path. No `author`/`capability` required.
- All other paths without auth → reject 403.
- `hello` before `authenticate` (enforced).
- Reconnect on same connection → 409.
- URIs during setup: peer-relative only (peer_id unknown).

### 6.3 Initial Capability (SHOULD grant)

```
grants: [
  { handlers:   {include: ["system/tree"]},
    resources:  {include: ["system/type/*", "system/handler/*"]},
    operations: {include: ["get"]} },
  { handlers:   {include: ["system/capability"]},
    resources:  {include: []},
    operations: {include: ["request"]} }
]
```

Namespace-scoped grants: implementations that segment handler visibility across peer groups target the initial grant to the appropriate namespace (`system/type/{namespace}/*`, `system/handler/{namespace}/*`). Handler grants at `system/capability/grants/*` are outside any non-admin initial scope.

### 6.4 Connection Error Codes

| Code | Status | Cause |
|------|--------|-------|
| `incompatible_protocol` | 400 | No common protocols |
| `incompatible_hash_format` | 400 | No common hash formats |
| `incompatible_key_type` | 400 | No common key types |
| `invalid_signature` | 401 | Bad nonce signature |
| `unsupported_key_type` | 400 | Unknown key type |
| `identity_mismatch` | 401 | Public key ≠ peer_id |
| `connection_already_established` | 409 | Duplicate connection |
| `connection_sequence_error` | 400 | Out-of-order operation |

---

## 7. Dispatch Chain

```
Frame received
  → Decode CBOR → envelope
  → Validate root entity hash + each included entity hash
  → EXECUTE?
    → URI = system/protocol/connect AND not connected?
      → Connection handler (no auth)
    → Otherwise:
      → verify_request(envelope, local_peer_id)
      → dispatch_path(uri, local_peer_id)          ; resolve to absolute path
      → reject 400 if peer_id != local_peer_id    ; inbound must target local
      → resolve_handler(path)                      ; tree walk, §4.5
      → check_permission(execute, capability, handler.pattern, local_peer_id)
      → Resolve handler grant from system/capability/grants/{pattern}
      → Build handler context (handler_grant + caller capability + execute + pattern + suffix + resource)
      → Handler processes operation
          (handler dispatches sub-requests with own grant; caller capability is context)
  → EXECUTE_RESPONSE?
    → Correlate by request_id, deliver
  → Other?
    → Close connection
```

Two-level authorization: dispatch scope (check_permission, all 4 dimensions from single grant) + handler path scope (check_path_permission, defense-in-depth).

---

## 8. Constants

### 8.1 Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 202 | Accepted (delivery acknowledged) |
| 400 | Bad request |
| 401 | Authentication failed |
| 403 | Forbidden |
| 404 | Not found |
| 409 | Conflict |
| 429 | Rate limited |
| 500 | Internal error |
| 501 | Not supported |
| 503 | Service unavailable |

### 8.2 Bounds Defaults

| Field | Default |
|-------|---------|
| `ttl` | 64 |
| `budget` | 100000 |
| `chain_id` | generated UUID |
| `visited` | [] |

### 8.3 Protocol Version

`"entity-core/1.0"`

### 8.4 Key Types

| Code | Algorithm | Key | Signature | Status |
|------|-----------|-----|-----------|--------|
| 0x01 | Ed25519 | 32 bytes | 64 bytes | REQUIRED |
| 0x02 | Ed448 | 57 bytes | 114 bytes | Reserved |

---

## 9. Conformance Matrix

### 9.1 Core Protocol — MUST

- Wire framing, ECF encoding, content hash validation (including format code → byte length), entity fidelity
- EXECUTE + EXECUTE_RESPONSE (2 wire types only)
- Envelope with per-entity hash validation
- Connection handler (hello, authenticate), pre-authorization rules
- Peer ID derivation, Ed25519 signatures, system/hash as bytes
- Request verification (§4.1), permission check (§4.2)
- Resource scope checking when resource present (§4.2)
- Pattern matching + scope checking via matches_scope
- Delegation chain verification + caveat checking
- Attenuation: scope_subset on all 4 dimensions
- Tree handler: get + put with resource target
- Two-level capability check (dispatch + handler path)
- System path reservation (user handlers MUST NOT register at system/*)
- Handler manifests at pattern paths as system/handler entities
- Handlers handler: register + unregister
- Bootstrap: system/tree, system/handler, system/type, system/protocol/connect
- Handler grants at system/capability/grants/{pattern}
- Handler index at system/handler/* with system/handler/interface entities
- `system/handler/interface` type recognition
- Type index at system/type/*
- Tree listing entry filtering against request capability — entries for paths the capability does not cover MUST be omitted
- Unknown field preservation

### 9.2 Core Protocol — SHOULD

- Root grant tracking + revocation
- Type system Level 1 (populate system/type/*)
- Bounds defaults when absent
- Tree listing pagination
- Tree extension (snapshot, diff, merge, extract, view trees)

### 9.3 Core Protocol — MAY

- Type system Level 2 (validation)
- Additional hash/key formats
- Resource limits enforcement
- Cycle detection (visited list)
- Frame compression/encryption via hello
- Extensions (inbox, subscription, content, compute)
- Standard library handlers

### 9.4 Extensions — Independent

Each extension has its own conformance section. A peer MAY implement any subset. Only dependency: Subscription requires Inbox.

---

## 10. Tree Paths (System Namespace)

```
system/tree                                   ; tree handler manifest
system/handler/                              ; handler index
system/handler/{pattern}                     ; system/handler/interface entity
system/type/                                 ; type definitions
system/type/primitive/*                      ; 8 primitives
system/type/system/*                         ; system types
system/capability                             ; capability handler manifest
system/capability/grants/{pattern}            ; handler capability grants
system/protocol/connect                       ; connection handler manifest
system/type                                  ; types handler manifest
system/tree/instances/{tree_id}               ; non-default tree configs
system/subscription/{id}                      ; active subscriptions
system/inbox/{path}/{id}                       ; inbox deliveries
system/continuation/suspended/{id}             ; suspended continuations
system/compute/*                              ; compute handler namespace
system/content/*                              ; content handler namespace
```
