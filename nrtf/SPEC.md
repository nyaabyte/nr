# NR Transmission Format Specification

<!-- DESC -->
NoReplacement Transmission Format (`NRTF`) is the single canonical, typed text format for transporting requests, responses, broadcasts, and control envelopes across the NR Network ("Net").  
Design goals:

- Human-readable and diff‑friendly
- Self‑canonical (no external language step) for signing and verification
- Minimal overhead for fast propagation
- Strong, predictable typing with deterministic ordering
- Forward compatible and easy for tooling to parse

<!-- EXPLANATIONS -->
## TLDR

```txt
api 1                                                        # API major version
schema 1                                                     # NRTF schema version
id watch/video 4f2d9d6e-9b11-7a33-b2aa-9d5c4c2e1f90          # route and resource id
ts 2025-08-11T12:34:56Z
nonce 7b5b0c9a1d8842b8b7f6c3d8a4e5f901                       # 128-bit hex
cap capabilities:[stream metrics]                            # optional (list form)
sigalg ed25519
pub base64:MCowBQYDK2VwAyEA1x2...                            # public key of sender
sig base64:o9KZ...                                           # signature over canonical NRTF (see Canonical Form and Signing)

msg :{                                                       # exactly one primary body element
    id "video_id@instan.ce/user"
    now :{ timestamp 0.0 user_watching "instan.ce/user" }
    related_ids :[ "abc" "def" "ghi" ]                       # list example
}
```

## Communication over Net

### API

The API statement defines the API major version used by the requester/responder. It MUST match `api_version` referenced in the Net specification. Patch versions (if any) are negotiated via capabilities (see `cap`).

```txt
api <int>
```

### Schema Version

The `schema <int>` statement declares the NRTF schema version (`nrtf_schema_version` in Net Flow). Identity Servers enforce compatibility rules:

- Exact match REQUIRED if responder does not advertise backward-compatible range in capabilities.
- On mismatch: reject (VERSION_MISMATCH) per Net Flow.

### ID

The ID statement identifies the logical route (RPC, stream, broadcast topic) and optional resource instance. Canonical forms:

```txt
id <route>
id <route> <resource-id>
id <verb>/<noun> <resource-id>
```

Where `<resource-id>` MAY be a UUIDv7, ULID, short slug, or composite token. A route MUST be registered with an expected request and response schema (see Type System). Spoofed or unregistered routes trigger rejection and possible suspicion scoring (see Net Flow repercussions: [Net Flow](../net/SPEC.md#flow)).

### Errors

Errors use the `error` statement and terminate the presence of a `msg` body unless a route explicitly allows a supplemental diagnostic table.  
Format:

```txt
error <scope>:<path> "<code>" ["<human-readable>"]
```

Examples:

```txt
error field:msg.id "not-equal"
error auth:token "expired" "Token expired 42s ago"
```

The error line alone is signed (indirectly) as part of the full frame. No JSON projection exists. The textual canonical form is authoritative.

### Messages

`msg` denotes the primary payload. Payloads are normally tables (`:{ ... }`). Other scalar primitives MAY be allowed for specialized routes (heartbeat pings) but SHOULD remain tables for extensibility.
Examples:

```txt
msg :{ id "abc" value 1 }
msg :{ items :{ first "a" second "b" } }
```

There MUST be at most one `msg` or one `error` per frame.

### Tables

Tables are defined using the `:` introducer followed by a brace block:

```txt
:{
    <key> <definition>
}
```

Nested tables reuse the same syntax. Keys MUST be unique within a table. Ordering of keys in the textual form MUST follow alphabetical ascending order for standard format (see below). Non-standard incoming ordering is permitted but MUST be normalized before signing.

### Allowed Key Characters

`[a-zA-Z0-9_:-]` (implementations MAY allow extended Unicode but MUST canonicalize via NFC). Keys beginning with `_` are reserved for future protocol metadata.

## Versioning and Compatibility

Major API version: `api` statement.  
Schema version: `schema` statement.  
Backward-compatible additions (new optional keys) do not bump major. Removing or changing type of existing key requires major increment.  
Capabilities advertisement: optional `cap` statement. Preferred modern forms:

```txt
cap capabilities:[stream metrics]
cap capabilities:{ stream true metrics true }        # table alternative
```

Negotiation rules:

1. Client sends desired (api, schema) and capabilities.
2. Server either accepts or responds with `error version:mismatch "expected=1 got=2"` (optionally providing supported range in human field).
3. Instances MUST NOT downgrade automatically without operator policy unless a downgrade is listed in its capabilities.

## Canonical Form and Signing

NRTF is inherently self‑canonical. Implementations MUST NOT project frames into other formats for signature computation.

Canonicalization algorithm (deterministic for signing and hashing):

1. Parse statements into structured form (ordered header list + single body element).
2. Validate there is at most one of each singleton header (`api`, `schema`, `id`, `ts`, `nonce`, `cap`, `sigalg`, `pub`, `sig`). Duplicate = rejection.
3. Remove any `sig` line(s) temporarily from the canonical set (signature is over everything else).
4. Canonicalize whitespace: when re‑rendering, each header line = `<key><SP><value>\n` with a single space separator. No trailing spaces.
5. Tables: keys MUST be unique. Order keys lexicographically ascending by raw UTF‑8 bytes of the key. Render as `:{<SP>` then a single space before each key except first if on new line (implementations MAY compact). Recommended canonical rendering (single line or minimal newlines) is:
    - Opening token `:{`
    - Either (a) one space separated sequence: `key value key value ...` then `}` OR (b) newline and 4‑space indent style. Both are acceptable for ingestion. For signing MUST normalize to style (a) single‑line minimal form.
6. Lists: order is significant and preserved. Render as `:[ item1 item2 ... ]` with single spaces between items, no trailing space before `]`.
7. Scalars:
    - int: base‑10, optional leading `-`, no leading zeros (except zero itself).
    - float: minimal decimal (strip trailing zeros. No `+`. At least one digit before and after decimal if decimal present). Scientific notation disallowed (use plain form) until a future extension.
    - bool: `true` / `false`.
    - none: literal `none`.
    - string: quoted with `"`. Escape `"` and `\\` and control chars (0x00-0x1F) using `\xHH`.
8. Binary data SHOULD be base64 prefixed: `base64:...` unless the route declares a different encoding.
9. Canonical body: exactly one of `msg` or `error` followed by a space and a value/table/list per grammar.
10. Concatenate canonical header lines (excluding `sig`) and body line(s) (already canonicalized) with `\n` line endings (LF). Append a final trailing LF.
11. Compute signature over these bytes using the algorithm declared in `sigalg`.
12. Insert the `sig` line after `pub` (or at end of headers if `pub` absent in a future extension) using base64 signature.

Pseudocode:

```txt
frame_bytes = canonical_render(headers_without_sig, body)
signature = sign(priv_key, frame_bytes)
emit headers_without_sig + ["sig base64:" + b64(signature)] + canonical_body
```

Signature Verification (ED25519):

```txt
ok = verify(pub, signature, canonical_render(headers_without_sig, body))
```

If verification fails: reject with `error auth:signature "bad-signature"`.

Replay Protection: combination of (`nonce`, `ts`, `pub`) must be unique inside a sliding window (`±300s` by default). Nonce length MUST be exactly 128 bits (32 hex characters).

Timestamp (`ts`) MUST be RFC3339 UTC. Clock skew > configured window => rejection with `error time:skew "too-far"`.

## Grammar (Example)

EBNF-like (whitespace = space / tab / newline):

```txt
message    = header* body ;
header     = api / schema / id / ts / nonce / cap / sigalg / pub / sig / ext_header ;
api        = "api" SP int ;
schema     = "schema" SP (int / string) ;
id         = "id" SP route (SP resource_id)? ;
ts         = "ts" SP rfc3339 ;
nonce      = "nonce" SP hex128 ;
cap        = "cap" SP capability_kv ;
sigalg     = "sigalg" SP token ;
pub        = "pub" SP base64 ;
sig        = "sig" SP base64 ;
ext_header = token SP token (SP token)* ;  # forward-compatible
body       = error / msg ;
error      = "error" SP scope_path SP quoted (SP quoted)? ;
msg        = "msg" SP value ;
value      = string / int / float / bool / none / table ;
table      = ":{" ( WS* pair )* WS* "}" ;
pair       = key SP value ;
```

`route` tokens use `/` to segment verbs/nouns. `scope_path` uses `<scope>:<path>`.

## Type System

Primitive types: string, int, float, bool, none.
Composite: table.

### Lists (Array Native Syntax)

Lists are first‑class via `:[ ... ]`.
Rules:

- Items separated by a single space.
- Items are any primitive or table or nested list.
- Empty list: `:[ ]`.
- Rendering for signing MUST collapse internal whitespace to single spaces.
- Order preserved. Lists are NOT re‑sorted.

Example:

```txt
msg :{ features :[ stream metrics tracing ] peers :[ :{ id "a" } :{ id "b" } ] }
```

Tables remain preferred for associative maps. Lists represent positional collections.

## RegistrationEnvelope (Unified NRTF Form Only)

`RegistrationEnvelope` (see Net spec) is ALWAYS transmitted as NRTF:

```txt
api <api_version>
schema <nrtf_schema_version>
id register/instance <instance_id>
ts <timestamp>
nonce <nonce>
cap capabilities:[stream metrics]
transport alg <transport_sig_alg>
pub base64:<public_key>
sigalg <proof_sig_alg>
sig base64:<proof_of_possession_signature>
msg :{ instance_url "<https-url>" connected_identity_servers :{ primary "<fingerprint>" } }
```

## Protections (NRTF Layer)

1. Canonical Form Attacks: Reject duplicate keys. Enforce normalized key order before signing.
2. Injection: Control characters (0x00-0x1F except tab/newline) MUST be escaped or rejected inside strings.
3. Oversized Messages: Implement max byte length (configurable, RECOMMENDED ≤64KB for control messages) and fail with `error size:msg "too-large"`.
4. Nonce Reuse: Track per (pub, nonce). Duplicates within window increment suspicion.
5. Downgrade: If peer proposes lower `api` or `schema` than previously confirmed for same identity without key rotation justification, flag suspicion.
6. Key Rotation: When `ROTATE_KEY` route is used, old `pub` must sign a rotation proof referencing new `pub`. Store chain for audits.
7. Consistent Parsing: Whitespace is insignificant except within quoted strings. Multiple spaces collapse. Trailing spaces ignored.
8. Time Skew: Large skew events optionally logged for operator time-sync alerts.
9. Partial Frames: In streaming transports, frames MUST terminate with a blank line (double newline) to delimit. Incomplete frames time out.
10. Encryption: NRTF itself does not provide encryption. Rely on TLS per Net spec.

## Generic Data Transport

Lightweight routes let instances store generic structured data, search, coordinate cache, sync, and manage user‑scoped records. Detailed flow lives in Net spec; this section only lists message schema stubs.

Core categories:

- Data store, retrieve, delete
- Search (query and index update)
- Cache (request, response, invalidate)
- User data and profile
- Federation sync (delta updates)

Single illustrative frame:

```txt
api 1
schema 1
id data/store doc-123
ts 2025-08-11T12:34:56Z
nonce aabbccddeeff00112233445566778899
sigalg ed25519
pub base64:...
sig base64:...
msg :{ data_id "doc-123" visibility "public" content_type "text/plain" data :{ title "Hello" body "World" } }
```

## Additional Concepts

### NRTF-over-UDP

NRTF frames MAY be sent over UDP. Each frame MUST be self-contained and terminated by a blank line (double LF separator). Lost frames are caller responsibility. Chunked payload indices allow selective retransmission.

### NRTF as a JSON replacement

NRTF may be used as a more efficient alternative to JSON for data interchange.
It ought to be compliant between different Instances otherwise your Instance may experience compatibility issues during transmission, and if you do not identify yourself and your schema correctly, your Instance may be marked as suspicious and deleted from the Net.

### From JSON to NRTF

#### Schema Types

The authoritative schema type is specified as "1".
Though implementations may be more flexible with their schema types, it is generally recommended to use this format for custom data schemas:

```txt
schema "StreamingServer.xyz/1"
```

#### Type Differences

The key similarities/differences between NRTF and JSON are:

- **Strings**: NRTF strings are always quoted and can include escape sequences. JSON strings are also quoted but have different escaping rules.
- **Numbers**: NRTF supports integers and floats as distinct types, while JSON has a single number type.
- **Booleans**: NRTF uses lowercase `true` and `false`, while JSON uses the same.
- **Tables**: NRTF tables are ordered key/value pairs, while JSON objects are unordered.
- **Lists**: NRTF lists are explicitly defined with `:[ ... ]`, while JSON arrays use `[ ... ]`.

The type system is inherently different: Validation and type checking is baked into the implementation, Instance, Identity Server, etc using NRTF type definitions. JSON lacks this strict typing, which can lead to ambiguity in data interpretation.

#### Steps

To convert JSON to NRTF, follow these steps:

- Parse the JSON object into its components (API version, schema version, ID, timestamp, nonce, etc.).
- Map the JSON fields to their NRTF equivalents.
- Serialize the NRTF frame.

#### Example of a JSON to NRTF Conversion

```json
{
  "api": 1,
  "schema": "StreamingServer.xyz/1",
  "id": "watch/video 4f2d9d6e-9b11-7a33-b2aa-9d5c4c2e1f90",
  "ts": "2025-08-11T12:34:56Z",
  "nonce": "d4e5c6b7a8f90123aa55ee771234abcd",
  "sigalg": "ed25519",
  "pub": "base64:MCowBQYDK2VwAyEA1x2...",
  "sig": "base64:Q9sm...",
  "msg": {
    "status": "ok",
    "bitrate": 3500.0,
    "viewer_count": 102
  }
}
```

```txt
api 1
schema "StreamingServer.xyz/1"
id watch/video 4f2d9d6e-9b11-7a33-b2aa-9d5c4c2e1f90
ts 2025-08-11T12:34:56Z
nonce d4e5c6b7a8f90123aa55ee771234abcd
sigalg ed25519
pub base64:MCowBQYDK2VwAyEA1x2...
sig base64:Q9sm...
msg :{ status "ok" bitrate 3500.0 viewer_count 102 }
```

If using generic JSON data, add signatures, timestamps, NRTF IDs, nonces and other data to the transmission and reorganize your existing messages into `msg` fields.

## Examples

Successful response:

```txt
api 1
schema 1
id watch/video 4f2d9d6e-9b11-7a33-b2aa-9d5c4c2e1f90
ts 2025-08-11T12:34:56Z
nonce d4e5c6b7a8f90123aa55ee771234abcd
sigalg ed25519
pub base64:MCowBQYDK2VwAyEA1x2...
sig base64:Q9sm...
msg :{ status "ok" bitrate 3500.0 viewer_count 102 }
```

Error response:

```txt
api 1
schema 1
id watch/video 4f2d9d6e-9b11-7a33-b2aa-9d5c4c2e1f90
ts 2025-08-11T12:35:01Z
nonce 09a1b2c3d4e5f6078899aabbccddeeff
sigalg ed25519
pub base64:MCowBQYDK2VwAyEA1x2...
sig base64:7Rlm...
error field:msg.viewer_count "not-found"
```

## Internal Type Verification

## Types

Types:

| Type   | Example        | Canonical Form          | Notes                                           |
| ------ | -------------- | ----------------------- | ----------------------------------------------- |
| string | "hello"        | quoted and escapes      | UTF-8. Escape `"` `\\` control chars via `\xHH` |
| int    | 42             | digits                  | 64-bit signed RECOMMENDED                       |
| float  | 3.14           | minimal decimal         | No NaN/Inf. No scientific notation              |
| bool   | true           | true / false            | lowercase                                       |
| none   | none           | none                    | Represents absence                              |
| table  | :{ key value } | ordered key/value pairs | Keys sorted ascending for signing               |
| list   | :[ a b c ]     | ordered items           | Order preserved, not sorted                     |
| binary | base64:AAAA    | base64:&lt;data&gt;     | For small blobs                                 |

Schema declarations MAY be expressed in a pseudo-table:

```txt
schema watch/video.request :{
    id string
    now :{ timestamp float user_watching string }
}
schema watch/video.response :{ status string bitrate float viewer_count int }
```

These lines are not transmitted during normal operation (unless it's an introspection route) but do assist tooling.

## Binary and Chunked Payloads

Small binary fields SHOULD be inlined base64. For large payload streaming (video, etc.) use chunked framing:

```txt
api 1
schema 1
id stream/segment <segment_id>
ts <ts>
nonce <nonce>
chunk meta :{ index 0 total 5 content_sha256 "ab..." bytes 1048576 }   # header table
sigalg ed25519
pub base64:...
sig base64:...
msg :{ data_chunk base64:<base64-blob> }
```

Rules:

- Each chunk has `chunk meta` with `index` (0-based), `total`, `content_sha256` (hash of concatenated raw bytes), and `bytes` (total byte length optional).
- Chunks MUST share identical `nonce` and `segment_id` pair.
- Receiver buffers until all indices [0..total-1] present. Verify hash. Then deliver assembled object.

## Conformance

An implementation is conformant if it:

1. Parses all required statements (`api`, `schema`, `id`, `msg` or `error`).
2. Enforces standard format prior to signing/verifying.
3. Enforces uniqueness of keys and replay protections (nonce+ts+pub).
4. Rejects messages violating declared type schemas (if locally registered).
5. Provides error codes per Error section.

## Error Codes (Catalog)

| Code               | Meaning (Summary)                         |
| ------------------ | ----------------------------------------- |
| VERSION_MISMATCH   | api/schema not supported                  |
| REPLAY_DETECTED    | Nonce+ts+pub reused or outside window     |
| BAD_SIGNATURE      | Signature verification failed             |
| FORMAT_VIOLATION   | Parsing / grammar / duplicate key error   |
| SIZE_EXCEEDED      | Frame exceeds configured byte limit       |
| TIME_SKEW          | ts outside allowable skew window          |
| ABUSE_THRESHOLD    | Rate / suspicion threshold crossed        |
| INCOMPATIBLE_CAP   | Missing required capability               |
| POLL_CLOSED        | Vote received after expiry                |
| NOT_FOUND          | Referenced resource absent                |
| UNAUTHORIZED       | Auth / key invalid                        |
| MIGRATION_CONFLICT | Overlapping migration attempt             |
| ALIAS_EXPIRED      | Action references expired alias           |
| STATE_CONFLICT     | Conflicting state view hash / merkle root |
| UNSUPPORTED_ROUTE  | Route not recognized                      |

Implementations MAY extend with vendor prefixed codes `VND_<TOKEN>`.

## Protocol Route Schemas (Implementations)

Mirrors the message schema list in Net spec. All fields are strings unless specified. `int`, `float`, `bool`, `none`, `table` map to NRTF primitives. These are not auto-sent. They document expected structures for tooling.

```txt
schema register/instance.request :{ instance_id string instance_url string api_version int nrtf_schema_version int timestamp string nonce string capabilities :{ } connected_identity_servers :{ } public_key string transport_sig_alg string proof_of_possession_signature string }
schema broadcast/register.message :{ registration :{ ... } instance_public_key_fingerprint string server_id string server_public_key string server_timestamp string server_observation_id string previous_ledger_hash string server_signature string }
schema challenge/reachability.request :{ instance_id string challenge_nonce string expected_signature_alg string }
schema challenge/reachability/response.message :{ instance_id string challenge_nonce string timestamp string signature string }
schema report/instance.message :{ instance_id string offending_message_uuids :{ } original_message_content_hash string original_instance_signature string server_signature string observed_api_version int observed_schema_version int server_timestamp string integrity_hash string }

# Offenses

schema evidence/request.message :{ instance_id string requestor_id string message_uuids :{ } }
schema evidence/response.message :{ instance_id string message_existence :{ } message_hashes :{ } current_api_version int signature string }
schema report/server.message :{ server_id string verified_message_signature string offending_server_message_uuid string victim_instance_message_uuid string cached_server_public_key string victim_instance_public_key string offending_message_content_hash string local_network_identity_list_root string server_view_root string divergence_proof :{ } instance_signature string }
schema view/request.message :{ server_id string nonce string }
schema view/challenge.message :{ server_id string observation_root string nonce string }
schema consensus/poll.message :{ target_type string target_id string evidence_hash string poll_id string expires_at string }
schema consensus/vote.message :{ poll_id string voter_type string voter_id string decision string signature string }
schema consensus/result/server.message :{ server_id string poll_id string decision string aggregate_signatures :{ } timestamp string }

# Migration
schema key/rotate.message :{ subject_type string subject_id string old_public_key string new_public_key string rotation_timestamp string old_key_signature string new_key_signature string }
schema heartbeat.message :{ subject_type string subject_id string last_seen_ts string state_hash string signature string }
schema recurse/request.message :{ traversal_id string depth int visited :{ } }
schema recurse/response.message :{ traversal_id string server_public_key string signed_instance_list :{ } signed_peer_identity_servers :{ } response_timestamp string signature string }
schema migration/server/announce.message :{ migration_id string old_server_id string new_server_public_key string new_server_endpoints :{ } planned_cutover string old_server_signature string quorum_attestations :{ } }
schema migration/server/commit.message :{ migration_id string old_server_id string new_server_id string cutover_timestamp string final_state_root string new_server_signature string old_server_signature string }
schema migration/instance/request.message :{ migration_id string old_instance_id string new_instance_id string new_instance_url string old_public_key string new_public_key string request_timestamp string old_instance_signature string }
schema migration/instance/approve.message :{ migration_id string old_instance_id string new_instance_id string approval_timestamp string server_id string server_signature string }
schema migration/instance/notice.message :{ migration_id string old_instance_id string new_instance_id string alias_expires_at string server_id string server_signature string }
schema migration/aliases/update.message :{ alias_set_hash string active_aliases :{ } server_id string timestamp string server_signature string }

# Generic Data Transport Routes

schema data/store.request :{ data_id string content_type string visibility string data table signature string }
schema data/store.response :{ data_id string stored_at string instance_id string signature string }
schema data/retrieve.request :{ data_id string requestor_id string access_token string }
schema data/retrieve.response :{ data_id string content_type string data table cached_at string signature string }
schema data/delete.request :{ data_id string owner_id string signature string }
schema data/delete.response :{ data_id string deleted_at string signature string }
schema search/query.request :{ query string filters :{ } limit int offset int requestor_id string }
schema search/query.response :{ results :[ ] total_count int query_id string cached_at string signature string }
schema search/index.message :{ data_id string keywords :[ ] metadata table instance_id string signature string }
schema cache/invalidate.message :{ data_id string reason string timestamp string instance_id string signature string }
schema cache/request.message :{ data_id string priority int requestor_instance_id string }
schema cache/response.message :{ data_id string cache_status string cached_instances :[ ] signature string }
schema user/data/store.request :{ user_id string data_key string data_value table visibility string signature string }
schema user/data/retrieve.request :{ user_id string data_key string requestor_id string access_token string }
schema user/data/retrieve.response :{ user_id string data_key string data_value table retrieved_at string signature string }
schema user/profile/get.request :{ user_id string fields :[ ] requestor_id string }
schema user/profile/get.response :{ user_id string profile table fields_provided :[ ] signature string }
schema federation/sync.request :{ last_sync_timestamp string filter_types :[ ] requestor_instance_id string }
schema federation/sync.response :{ updates :[ ] sync_timestamp string next_page_token string signature string }
```

Tooling MAY parse these schemas to auto-generate parsers/validators.
