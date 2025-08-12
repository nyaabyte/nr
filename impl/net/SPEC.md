# NR Network Specification

<!-- DESC -->
The NoReplacement Network ("Net") is made of Identity Servers and Instances. Some could remotely compare it to existing federated networks such as ATProto, which also uses relays.
The goal of the Net is to keep a correct, consistent view of who is on the network and reject potential tampering or abuse.
This is with strong typing (NRTF), digital signatures, timestamps and nonces, quorum voting, and audit logs.

<!-- ROUTES -->
## Shared REST Endpoints (/net/nr/...)

These MUST be implemented by both Server and Instance.

1. GET `/net/nr/health` (nrtf_route: `heartbeat`)
    - Returns instance status, version, uptime.
    - Response JSON (sample):

      ```json
      {"status":"ok","api_version":1,"schema":1,"time":"2025-08-11T12:34:56Z","uptime_s":1234 }
      ```

2. GET `/net/nr/capabilities` (nrtf_route: `capabilities/request`)
    - Returns node capabilities and supported schema providers.
    - Query params: `type` (optional filter: node|schema_providers|all)
    - Response JSON (sample):

      ```json
      {"node_capabilities":{"stream":true,"search":true},"supported_schema_providers":["StreamingServer/1","ChatServer/2"],"schema_compliance_matrix":{"StreamingServer/1":["store","retrieve","search"],"ChatServer/2":["store","retrieve"]},"instance_type":"StreamingServer"}
      ```

<!-- EXPLANATIONS -->
## Protocol Message Schemas (For Implementation)

Types reference NRTF primitives: string, int, float, bool, none, table. `uuid7` denotes a UUIDv7 string. `fingerprint` denotes hash (e.g. SHA-256 hex). `time` = RFC3339 UTC string. `bytes_b64` = base64.

Note: All control messages are transmitted ONLY as NRTF frames. There is no JSON/CBOR conversion for signing.

## Notes on Protocols

The wider Internet (HTTP/S) will be used for communication between Instances and Identity Servers initially. We're looking into a potential migration to Raven Dev Team's [Betanet](https://ravendevteam.org/betanet) upon it reaching maturity for significantly more decentralized transport.

## Flow

**See [NRTF "Communication over Net"](../nrtf/SPEC.md#communication-over-net)**

### Control Plane Flows

- Instance-to-Identity Server
  - When an Instance registers (happy path):
        1. Instance prepares RegistrationEnvelope (pure NRTF):
        - Fields: instance_id (UUIDv7), instance_url (HTTPS, externally reachable), api_version, nrtf_schema_version, timestamp (RFC3339), nonce (128-bit random), capabilities (optional list/table), connected_identity_servers (initially declared), public_key (Ed25519 recommended), proof_of_possession_signature (signature over canonical NRTF headers+body), transport_sig_alg.
    2. Instance sends RegistrationEnvelope to a specifically-defined Identity Server over mutually-authenticated TLS (mTLS), HTTPS with DANE or TOFU, or [Betanet](../../simple/compliance/Betanet.md)'s Outer TLS + Access-Ticket + Handshakes and a Certificate Pin set. Direct TLS is not a substitute for the application signatures.
        3. Identity Server performs verification steps:
            - Version check: api_version and nrtf_schema_version must be compatible. Else reject VERSION_MISMATCH (do not cache).
            - Freshness and replay: timestamp inside ±300s (default) and nonce not reused in last 24h for this public key. Else REPLAY_DETECTED.
            - URL: must be HTTPS and syntactically valid. Reachability probe (HEAD) allowed.
            - Signature: verify proof_of_possession_signature over canonical NRTF (excluding sig line). If fail → BAD_SIGNATURE.
            - Abuse controls: apply rate limit and Sybil heuristics (key reuse, reputation). Exceeded → SOFT_FAIL_THROTTLE or HARD_FAIL_ABUSE.
            - Persistence: write provisional record (state=Pending) and append hash of envelope to audit log.
        4. Identity Server constructs RegistrationBroadcast containing: original canonical registration data, instance_public_key_fingerprint, server_id, server_public_key, server_timestamp, server_observation_id (UUIDv7), previous_ledger_hash (hash-chaining audit), and server_signature over all fields.
        5. Quorum (recommended): wait for ≥Q peers to reply VALIDATE_OK within T (min 5s). On success mark Active. If not enough replies: leave Pending, retry, or enter Isolated mode.
        6. Broadcast: send RegistrationBroadcast to all current Instances and peer Identity Servers (fan-out with exponential backoff). Deduplicate by observation_id.
  - Behavior of receiving Instances (upon RegistrationBroadcast):
        1. Verify server_signature using cached server_public_key (or fetch and bootstrap trust if first encounter via trust-on-first-use and optional out-of-band hash/pin list).
        2. Verify embedded registration proof_of_possession_signature.
        3. Version/schema: if incompatible mark Incompatible. Do not gossip further. Optionally send Advisory.
        4. Cache key: store instance_public_key by instance_id with state=UnverifiedReachability.
        5. Gossip: forward broadcast unchanged unless duplicate observation_id, too old, or policy forbids. Preserve ordering. If backlog > N prioritize newer observation_ids.
        6. Reachability: send CHALLENGE(instance_id, challenge_nonce, expected_signature_alg) to instance_url. Expect SIGNED_CHALLENGE echoing nonce, timestamp and signature over (challenge_nonce || instance_id || timestamp). If valid → state=Reachable.
  - Identity Server recursion and aggregation:
        1. To supply complete Net view, Identity Server initiates RECURSE_IDENTITIES query to each peer (depth-first or breadth-first with cycle detection using visited server_public_key fingerprints).
        2. Each Identity Server responds with: server_public_key, signed_instance_list (list of {instance_id, public_key_fingerprint, state_hash, connected_identity_servers_fingerprints, last_seen_ts}), signed_peer_identity_servers (similar structure), response_timestamp, and signature.
        3. Verify each signature, reject skewed timestamps, merge using "latest last_seen_ts wins". If conflicting fingerprint for same instance_id → raise CONFLICT and quarantine that instance.
        4. Build Network Identity List. Hash it (Merkle root). Sign. Return path back hop-by-hop. Each hop may append attestation.
        5. Instance stores set and subscribes to delta events (REGISTER, DELIST, STATE_CHANGE) via long-poll, WebSocket, or signed batch.
  - Version/Type mismatch handling:
    - On mismatch reject fast. If previously Active set state=StaleVersion (not fully delisted) and ignore until VERSION_UPGRADE.
  - Marking an Instance as sinful (Instance Misbehavior Path):
        1. Identity Server criteria for Sin: malformed messages beyond retry threshold, repeated replay attempts, signature forgery attempt, schema deviation, or propagation of tampered broadcasts.
        2. Server issues SIN_REPORT(instance_id, evidence_bundle) where evidence_bundle includes: offending_message_uuid(s), original_message_content (canonical), original_instance_signature, server_signature, observed_api_version, observed_schema_version, server_timestamp, and integrity_hash.
        3. Broadcast to all healthy Instances (health = consistent version and successful audit log verification, not under probation) and peer Identity Servers.
        4. Receiving healthy Instance actions:
            - Verify server_signature and original_instance_signature.
            - Validate schema/version claims against local NLTF types.
            - Contact accused Instance directly with EVIDENCE_REQUEST(message_uuid list) to solicit its SignedEvidenceResponse (includes message_uuid existence flags, original content if retained, current api_version, and its signature).
            - Perform layered evaluation:
                a. If message_uuid absent per accused but present in server evidence → possible server Sin. Mark server Suspicious and escalate.
                b. If api_version mismatch (accused claims newer/older) → note but continue.
                c. If content and signatures all align with server claim and violation stands → local vote = GUILTY.
            - Post-flight tamper detection:
                - Signatures mismatch while public keys match → suspect server content alteration (server Sin).
                - Public keys mismatch but signatures validate over legitimate content → suspect key substitution attempt (server Sin).
        5. Quorum-based delisting: Only after ≥Q healthy Instances and ≥Q Identity Servers submit CONSENSUS_VOTE(GUILTY) within T window is the accused Instance marked Delisted globally. Otherwise mark Disputed.
        6. Server removes Delisted Instance after consensus, keeps proof for audit trail.
  - Identity Server to Instance (Server Misbehavior Path):
    - Trigger: An Instance detects a sinful Server (tampering, forged evidence, key substitution, replay facilitation, inconsistent signed view hashes, or majority-confirmed false SIN_REPORT).
    - Instance compiles SERVER_SIN_REPORT including: verified_message_signature, offending_server_message_uuid, victim_instance_message_uuid (if applicable), cached_server_public_key, victim_instance_public_key (if permission granted), offending_message_content (canonical), local_deduplicated_identity_set_root, server_view_root (claimed), divergence_proof (Merkle path differences), timestamps, and instance_signature.
    - Propagation and Verification Steps:
            1. Receiving Identity Server validates instance_signature and ensures api_version/schema alignment.
            2. Checks divergence_proof by recomputing Merkle paths. If mismatch confirmed, increments server reputation penalty.
            3. Contacts victim Instance (if different) with VIEW_CHALLENGE to corroborate message_uuid and content hash.
            4. Requests accused Server's self-signed current view via VIEW_REQUEST(server_id, nonce). Accused must respond with signed Network Identity List root and nonce within SLA.
            5. Compares accused response root with reported divergence_proof. If discrepancy stands and ≥Q corroborations from other Identity Servers OR cryptographic inconsistency (invalid signature) present, mark server DelistingPending.
            6. Broadcast CONSENSUS_POLL(server_id, evidence_hash) to gather votes from healthy Instances and Identity Servers.
            7. If quorum GUILTY reached, issue SERVER_DELIST_NOTICE with multi-signature aggregate (≥Q identity server signatures) for finality. Distribute to all Instances.
            8. Instances upon receipt remove server from active list, retain entry in RevocationCache (with expiry) to prevent stale reintroduction without re-audit.
            9. Accused Server may later submit REINSTATEMENT_REQUEST including patch notes and version proofs and attestations from ≥Q healthy Instances. Process mirrors registration but requires differential audit of prior infractions.
    - If api_version of the reporting Instance is itself incorrect, receiving Server returns REJECT_UNSUPPORTED_VERSION and may rate-limit further accusations from that Instance.

### Data Plane

Generic data layer adds a set of routes: store, retrieve, and delete data, search and index update, cache request, response, and invalidate, and federation sync deltas. Core rules:

- All frames signed (same NRTF rules).
- Visibility: public (replicable and searchable), private (origin only), or restricted (implementation policy).
- Public items MAY be indexed and gossiped. Private items never leave origin on store.
- Federation sync: timestamp and optional type filters, conflicts resolved by (newer ts, then lex hash).
- Cache coordination is advisory. Correctness does not depend on cache hits.
- Schema compliance: Generic data MUST be marked with both `schema_provider` (for example, "StreamingServer/1") and `schema_version`. Instances and Identity Servers MUST only store and process data for schema providers and versions they explicitly support in their capabilities. If the schema provider or version is unsupported, the node MUST NOT attempt to process the content and MUST forward the frame unchanged to maintain network connectivity.
- Capabilities advertisement: Nodes MUST advertise supported schema providers and versions via capabilities endpoints. Data operations MUST check the local node’s declared support and the sender’s capabilities before processing, otherwise the data MUST be forwarded unchanged through the rest of the Net.
- Pass-through behavior: When a node receives data with an unsupported `schema_provider` or version, it MUST forward the data unchanged through federation without attempting to parse or validate the content beyond NRTF frame integrity.

#### Identity Server storage policy

- Identity Servers MUST explicitly blacklist schema providers and versions they refuse to store.
- If a schema provider or version is not blacklisted, Identity Servers MUST store generic data at least as the canonical NRTF frame with minimal metadata even if they do not support parsing that schema. Processing and indexing are only for supported schemas.
  - Security Measures:
    - Multi-layer Signatures: Every propagated object (registration, broadcast, evidence, view) is individually signed. Optional multi-signature aggregates for consensus-critical events.
    - Replay Protection: Nonce, timestamp and sliding cache per key.
    - Merkle Integrity: Network Identity List hashed. Divergence proofs enable efficient inconsistency challenges.
    - Quorum Requirements: Delisting (Instance or Server) requires ≥Q matching votes from both roles (Instances and Identity Servers). Q is configurable but must be ≥2 and < half of total healthy participants to avoid deadlock. Recommended dynamic Q = ceil(log2(N+1)).
    - Rate Limiting and Abuse Detection: Adaptive throttling based on moving averages. Misbehavior raises suspicion score. Exceeding threshold triggers probation rather than immediate delist (except for cryptographic forgery attempts).
    - Key Rotation: Instances and Servers may rotate keys via ROTATE_KEY request containing new_public_key, old_public_key_fingerprint, rotation_timestamp, and cross-signature from old key. Peers validate chain and update caches. Rotations within &lt;M minutes frequency cause suspicion.
    - Audit Logging: Append-only hash chain per Identity Server. Periodic checkpoint signed roots gossip to peers for external verification. Mismatch triggers VIEW_CHALLENGE.
    - Revocation Caches: Maintain recently delisted keys to prevent replay re-registration using stale broadcasts.
    - Schema and Version Governance: Strict semantic versioning. Backward compatibility window defined. Minor versions may be accepted if declared by capability negotiation. Major mismatch → rejection.
    - Transport Security: mTLS or equivalent STRONGLY RECOMMENDED. At minimum enforce TLS (or equivalent), strong cipher suites, OCSP stapling. Certificate pinning optional.
    - Privacy: Only fingerprints (hashes) of large evidence objects may be broadcast. Full content fetch on-demand to limit data leakage.
    - Consistent Conflict Resolution: For conflicting public_key_fingerprint per instance_id, place instance into Quarantine. No traffic forwarded until resolved by quorum.
    - Liveness Safeguards: Heartbeat (SIGNED_HEARTBEAT) every H seconds. Absence beyond K intervals transitions state to Unreachable. After further threshold → Expired (does not auto-delist, but excluded from quorum counts).
    - Resource Caps: Upper bounds on broadcast fan-out per interval. Overflow enters backpressure queue with signed ordering metadata.
    - Edge Case Handling: Duplicate observation_id → discard. Stale timestamp → discard. Unverifiable signature → reject and increment suspicion. Unreachable instance_url after R retries → mark Unreachable. Inconsistent Merkle root across ≥2 peers → initiate cross-check protocol.

Aggregation and validation rules (receive, publish, sync):

1. Receive: verify signatures, timestamp, and nonce. Ensure `schema_provider` and `schema_version` exist, apply blacklist, process if supported, otherwise store opaque frame and forward unchanged.
2. Publish: include schema-aware index data only for supported schemas, include minimal metadata for stored but unsupported schemas, preserve signatures and hashes.
3. Sync: deduplicate by `data_id` and `frame_hash`, resolve by newer timestamp then lexicographic hash, validate signatures, apply schema-level validation only for supported schemas, do not propagate blacklisted items.

### Migration (Servers and Instances)

Migration lets an Identity Server or Instance move to new infrastructure (new key and/or URL) without losing reputation and while limiting takeover risk.

Goals:

- Preserve continuity (registrations, votes, view roots).
- Provide cryptographic linkage between old and new identities.
- Time‑boxed alias period to avoid indefinite duplication.
- Allow simple rollback if commit not finalized.

Phases (Server):

1. Announce: SERVER_MIGRATION_ANNOUNCE broadcast (signed by old key. MAY include peer attestations).
2. Cutover: After planned_cutover, new server begins responding in parallel. Peers challenge both.
3. Commit: SERVER_MIGRATION_COMMIT (signed by both old and new keys) finalizes mapping old_server_id → new_server_id. Old key retires (only used for revocation proofs after grace window).
4. Alias Expiry: After configurable alias window W (recommended 24h) references to old_server_id MUST be treated as references to new_server_id. After expiry, old id invalid.

Phases (Instance):

1. Request: INSTANCE_MIGRATION_REQUEST from old instance (signed by old key) naming new key, new URL.
2. Approval: Identity Server validates reachability of new URL and key possession (challenge) then issues INSTANCE_MIGRATION_APPROVED.
3. Notice: INSTANCE_MIGRATION_NOTICE broadcast so caches add alias old_instance_id → new_instance_id until alias_expires_at.
4. Expiry: After alias_expires_at, old_instance_id considered Retired. Evidence or votes referencing old id during window map to new id.

Security Rules:

- Both keys must sign the commit/notice where possible.
- Alias mapping stored in hash chain (ALIAS_MAP_UPDATE) to prevent tampering.
- Migration requests rate limited. Multiple overlapping migrations for same subject rejected.
- Misuse (attempt to assume unrelated high‑reputation id) detected if old key signature missing or quorum attestations absent → reject and mark as suspicious.

Client Handling:

- On receipt of announce/notice, update local alias map { old_id → new_id, expires_at }.
- During alias period treat both ids as same principal for replay and rate limits (shared nonce namespace).
- When verifying historical signatures containing old_id after expiry, accept only if signature timestamp < alias_expires_at.

Types reference NRTF primitives: string, int, float, bool, none, table. `uuid7` denotes a UUIDv7 string. `fingerprint` denotes hash (such as SHA-256 hex). `time` = RFC3339 UTC string. `bytes_b64` = base64.

1. RegistrationEnvelope (NRTF route `register/instance`)
   - instance_id uuid7
   - instance_url string (https URL)
   - api_version int
   - nrtf_schema_version int
   - timestamp time
   - nonce string (32 hex chars = 128 bits)
   - capabilities list (`cap capabilities:[feature1 feature2]`) OR table (`cap capabilities:{ feature1 true }`)
   - supported_schema_providers list (list of supported schema_provider strings, e.g., ["StreamingServer/1", "ChatServer/2"])
   - connected_identity_servers table (id → fingerprint)
   - public_key bytes_b64
   - transport_sig_alg string
   - proof_of_possession_signature bytes_b64

2. RegistrationBroadcast (NRTF: broadcast/register)
   - registration RegistrationEnvelope (canonical form)
   - instance_public_key_fingerprint fingerprint
   - server_id fingerprint
   - server_public_key bytes_b64
   - server_timestamp time
   - server_observation_id uuid7
   - previous_ledger_hash fingerprint|null
   - server_signature bytes_b64

3. CHALLENGE (NRTF: challenge/reachability)
   - instance_id uuid7
   - challenge_nonce string (32 hex chars = 128 bits)
   - expected_signature_alg string

4. SIGNED_CHALLENGE (NRTF: challenge/reachability/response)
   - instance_id uuid7
   - challenge_nonce string
   - timestamp time
   - signature bytes_b64

5. SIN_REPORT (NRTF: report/instance)
   - instance_id uuid7
   - evidence :{ messages :{ &lt;uuid7&gt; fingerprint } original_contents_hash fingerprint } (or on-demand retrieval)
   - offending_message_uuids table (k → uuid7)
   - original_message_content_hash fingerprint
   - original_instance_signature bytes_b64
   - server_signature bytes_b64
   - observed_api_version int
   - observed_schema_version int
   - server_timestamp time
   - integrity_hash fingerprint (hash of concatenated ordered fields)

6. EVIDENCE_REQUEST (NRTF: evidence/request)
   - instance_id uuid7 (accused)
   - requestor_id uuid7
   - message_uuids table (k → uuid7)

7. SignedEvidenceResponse (NRTF: evidence/response)
   - instance_id uuid7 (accused)
   - message_existence table (uuid7 → bool)
   - message_hashes table (uuid7 → fingerprint|null)
   - current_api_version int
   - signature bytes_b64

8. SERVER_SIN_REPORT (NRTF: report/server)
   - server_id fingerprint|uuid7
   - verified_message_signature bytes_b64
   - offending_server_message_uuid uuid7
   - victim_instance_message_uuid uuid7|null
   - cached_server_public_key bytes_b64
   - victim_instance_public_key bytes_b64|null
   - offending_message_content_hash fingerprint
   - local_network_identity_list_root fingerprint
   - server_view_root fingerprint
   - divergence_proof table (paths → fingerprint)
   - timestamps :{ local time, observed time }
   - instance_signature bytes_b64

9. VIEW_REQUEST (NRTF: view/request)
   - server_id fingerprint
   - nonce string

10. VIEW_CHALLENGE (NRTF: view/challenge)
    - server_id fingerprint
    - observation_root fingerprint
    - nonce string

11. CONSENSUS_POLL (NRTF: consensus/poll)
    - target_type string (instance|server)
    - target_id uuid7|fingerprint
    - evidence_hash fingerprint
    - poll_id uuid7
    - expires_at time

12. CONSENSUS_VOTE (NRTF: consensus/vote)
    - poll_id uuid7
    - voter_type string (instance|server)
    - voter_id uuid7|fingerprint
    - decision string (GUILTY|NOT_GUILTY|ABSTAIN)
    - signature bytes_b64

13. SERVER_DELIST_NOTICE (NRTF: consensus/result/server)
    - server_id fingerprint
    - poll_id uuid7
    - decision string (GUILTY)
    - aggregate_signatures table (signer_id → bytes_b64)
    - timestamp time

14. ROTATE_KEY (NRTF: key/rotate)
    - subject_type string (instance|server)
    - subject_id uuid7|fingerprint
    - old_public_key bytes_b64
    - new_public_key bytes_b64
    - rotation_timestamp time
    - old_key_signature bytes_b64 (signature by old key over new key and timestamp)
    - new_key_signature bytes_b64 (optional cross-sign)

15. SIGNED_HEARTBEAT (NRTF: heartbeat)
    - subject_type string
    - subject_id uuid7|fingerprint
    - last_seen_ts time
    - state_hash fingerprint
    - signature bytes_b64

16. RECURSE_IDENTITIES (NRTF: recurse/request)
    - traversal_id uuid7
    - depth int (remaining depth)
    - visited table (fingerprint → bool)

17. RECURSE_IDENTITIES_RESPONSE (NRTF: recurse/response)
    - traversal_id uuid7
    - server_public_key bytes_b64
    - signed_instance_list table (instance_id → fingerprint)
    - signed_peer_identity_servers table (server_id → fingerprint)
    - response_timestamp time
    - signature bytes_b64

18. SERVER_MIGRATION_ANNOUNCE (NRTF: migration/server/announce)
    - migration_id uuid7
    - old_server_id fingerprint
    - new_server_public_key bytes_b64
    - new_server_endpoints table (label → url)
    - planned_cutover time
    - old_server_signature bytes_b64 (old key over all above)
    - quorum_attestations table (server_id → bytes_b64) (optional)

19. SERVER_MIGRATION_COMMIT (NRTF: migration/server/commit)
    - migration_id uuid7
    - old_server_id fingerprint
    - new_server_id fingerprint
    - cutover_timestamp time
    - final_state_root fingerprint
    - new_server_signature bytes_b64 (new key)
    - old_server_signature bytes_b64 (old key)

20. INSTANCE_MIGRATION_REQUEST (NRTF: migration/instance/request)
    - migration_id uuid7
    - old_instance_id uuid7
    - new_instance_id uuid7
    - new_instance_url string
    - old_public_key bytes_b64
    - new_public_key bytes_b64
    - request_timestamp time
    - old_instance_signature bytes_b64

21. INSTANCE_MIGRATION_APPROVED (NRTF: migration/instance/approve)
    - migration_id uuid7
    - old_instance_id uuid7
    - new_instance_id uuid7
    - approval_timestamp time
    - server_id fingerprint
    - server_signature bytes_b64

22. INSTANCE_MIGRATION_NOTICE (NRTF: migration/instance/notice)
    - migration_id uuid7
    - old_instance_id uuid7
    - new_instance_id uuid7
    - alias_expires_at time
    - server_id fingerprint
    - server_signature bytes_b64

23. ALIAS_MAP_UPDATE (NRTF: migration/aliases/update)
    - alias_set_hash fingerprint
    - active_aliases table (old_id → new_id)
    - server_id fingerprint
    - timestamp time
    - server_signature bytes_b64

<!-- COMMENT -->
#### Generic Data Transport Messages
<!-- END -->

23. DATA_STORE (NRTF: data/store)
    - data_id string (unique identifier for the data)
    - content_type string (MIME type or application identifier)
    - visibility string (public|private|restricted)
    - schema_provider string (provider/version, e.g. "StreamingServer/1")
    - schema_version int (semantic version for the data schema)
    - data table (the actual data content as NRTF table)
    - signature bytes_b64 (data integrity signature)

24. DATA_STORE_RESPONSE (NRTF: data/store.response)
    - data_id string
    - stored_at time
    - instance_id uuid7 (storing instance)
    - signature bytes_b64

25. DATA_RETRIEVE (NRTF: data/retrieve)
    - data_id string
    - requestor_id string (instance identifier)
    - supported_schemas list (list of supported schema_provider strings)
    - access_token bytes_b64 (authorization token)

26. DATA_RETRIEVE_RESPONSE (NRTF: data/retrieve.response)
    - data_id string
    - content_type string
    - schema_provider string (provider/version that created the data)
    - schema_version int (semantic version for the data schema)
    - data table (retrieved data content)
    - cached_at time
    - signature bytes_b64

27. DATA_DELETE (NRTF: data/delete)
    - data_id string
    - owner_id string
    - signature bytes_b64

28. DATA_DELETE_RESPONSE (NRTF: data/delete.response)
    - data_id string
    - deleted_at time
    - signature bytes_b64

29. SEARCH_QUERY (NRTF: search/query)
    - query string (search terms)
    - filters table (content_type, visibility, etc.)
    - limit int (max results)
    - offset int (pagination offset)
    - requestor_id string

30. SEARCH_QUERY_RESPONSE (NRTF: search/query.response)
    - results list (array of matching data entries)
    - total_count int
    - query_id uuid7
    - cached_at time
    - signature bytes_b64

31. SEARCH_INDEX_UPDATE (NRTF: search/index)
    - data_id string
    - keywords list (extracted keywords)
    - metadata table (additional searchable metadata)
    - instance_id uuid7
    - signature bytes_b64

32. CACHE_INVALIDATE (NRTF: cache/invalidate)
    - data_id string
    - reason string (updated|deleted|expired)
    - timestamp time
    - instance_id uuid7
    - signature bytes_b64

33. CACHE_REQUEST (NRTF: cache/request)
    - data_id string
    - priority int (cache priority 1-10)
    - requestor_instance_id uuid7

34. CACHE_RESPONSE (NRTF: cache/response)
    - data_id string
    - cache_status string (cached|not_cached|caching)
    - cached_instances list (instances with cached copies)
    - signature bytes_b64

35. FEDERATION_SYNC_REQUEST (NRTF: federation/sync)
    - last_sync_timestamp time
    - filter_types list (data types to sync)
    - requestor_instance_id uuid7

36. FEDERATION_SYNC_RESPONSE (NRTF: federation/sync.response)
    - updates list (incremental update records)
    - sync_timestamp time
    - next_page_token string (pagination token)
    - signature bytes_b64

37. CAPABILITIES_REQUEST (NRTF: capabilities/request)
    - requestor_id string (requesting entity identifier)
    - capabilities_type string (node|schema_providers|all)

38. CAPABILITIES_RESPONSE (NRTF: capabilities/response)
    - node_capabilities table (basic node capabilities)
    - supported_schema_providers list (list of supported schema_provider strings)
    - schema_compliance_matrix table (schema_provider → supported_operations)
    - instance_type string (optional, specific instance type like "StreamingServer")
    - signature bytes_b64

Field naming in NRTF tables should use snake_case to match these schema names.

## Schema Compliance and Data Routing

The NoReplacement Network enforces schema compliance to ensure type safety while maintaining network connectivity:

### Schema Provider Format

Schema providers use the format `<Provider>/<Version>` (e.g., "StreamingServer/1", "ChatServer/2").

### Compliance Rules

1. Registration: Instances MUST declare `supported_schema_providers` during registration.
2. Data Operations: Nodes MUST only process data for schemas they explicitly support.
3. Pass-through: Unsupported data MUST be forwarded unchanged through federation.
4. Capabilities: All nodes MUST expose `/capabilities` endpoints listing supported schemas.

### Processing Flow

1. Receive data: check `schema_provider` field in incoming data.
2. Compliance check: Verify if the schema is in the node's supported list.
3. Process/Forward:
   - If supported: Process normally (validate, store, index, etc.)
   - If unsupported: Forward through federation without content processing
4. Ensure connectivity: Network remains functional even with schema mismatches.

This approach allows the network to evolve and support diverse use cases while maintaining reliability and preventing data corruption from misunderstood formats.

- Identity Server to Instance
  - If the Server has committed a Sin:
    - (See Server Misbehavior Path above for detailed steps.)

<!-- SEE -->
## See

- [Identity Server](../identity/SPEC.md)
- [Instance](../net/SPEC.md)
- [NRTF](../nrtf/SPEC.md)
