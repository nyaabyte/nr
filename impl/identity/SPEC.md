# NoReplacement Identity Server Implementation Standard

## Status

This document is NORMATIVE and defines the mandatory requirements for implementing Net compliant Identity Servers.

## Conformance

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are normative and must be followed as such.

## Identity Server Requirements

### Core Functions

Identity Servers MUST implement the following functions:

1. Network Management: MUST maintain authoritative list of active Instances and peer Identity Servers
2. Registration Processing: MUST validate and process Instance registration requests
3. Consensus Coordination: MUST participate in network consensus for delisting decisions
4. Message Forwarding: MUST forward and validate signed control messages
5. Migration Support: MUST handle Identity Server and Instance migration procedures
6. Data Coordination: MUST coordinate generic data transport operations
7. Federation Synchronization: MUST provide federation sync and catalog services

### Transport Requirements

- Protocol: HTTPS with TLS 1.3 (or [Betanet](../../simple/compliance/Betanet.md)'s equivalent as defined in [Net](../net/SPEC.md)) REQUIRED
- Certificate Validation: REQUIRED
- Application Signatures: REQUIRED (TLS insufficient alone)
- Base Path: All endpoints MUST use `/net/id/` prefix

## Mandatory Endpoint Implementation

### Core Control Endpoints

Identity Servers MUST implement the following endpoints with exact specifications:

#### 1. Instance Management

**GET /net/id/instances** (NRTF: view/request and recurse/response)

- Purpose: Return current known active/pending Instances
- Query Parameters: `state` (active|pending|all) OPTIONAL
- Response: MUST include root_hash and instances array
- Performance: MUST respond within 10 seconds

**POST /net/id/register** (NRTF: register/instance)

- Purpose: Process Instance registration
- Input: RegistrationEnvelope in NRTF format
- Validation Steps (ALL REQUIRED):
  - Version compatibility check
  - Timestamp freshness (±300s window)
  - Nonce replay protection (24h window)
  - Signature verification
  - URL reachability probe
  - Rate limiting compliance
- Response Codes:
  - 202: Registration accepted
  - 400: VERSION_MISMATCH
  - 409: REPLAY_DETECTED
  - 429: SOFT_FAIL_THROTTLE
  - 403: HARD_FAIL_ABUSE

#### 2. Consensus Operations

**POST /net/id/consensus/vote** (NRTF: consensus/vote)

- Purpose: Accept consensus votes
- Input: Vote with poll_id, voter_type, voter_id, decision, signature
- Validation: MUST verify voter eligibility and signature
- Quorum Processing: MUST implement quorum counting

**GET /net/id/consensus/poll/:poll_id** (NRTF: consensus/poll)

- Purpose: Return poll status and current vote tally
- Response: MUST include current vote counts and remaining time

#### 3. Evidence and Reporting

**POST /net/id/evidence/request** (NRTF: evidence/request)

- Purpose: Request evidence from accused Instance
- Input: instance_id and message_uuids list
- Forwarding: MUST forward to target Instance

**POST /net/id/report/instance** (NRTF: report/instance)

- Purpose: Process Instance misbehavior reports
- Input: SIN_REPORT with evidence bundle
- Processing: MUST validate evidence and initiate consensus poll

**POST /net/id/report/server** (NRTF: report/server)

- Purpose: Process Identity Server misbehavior reports
- Input: SERVER_SIN_REPORT with divergence proof
- Processing: MUST validate divergence proof and coordinate response

#### 4. Migration Support

**POST /net/id/migration/server/announce** (NRTF: migration/server/announce)

- Purpose: Announce upcoming server migration
- Input: Migration announcement with old key signature
- Processing: MUST validate and broadcast to network

**POST /net/id/migration/server/commit** (NRTF: migration/server/commit)

- Purpose: Finalize server migration
- Input: Dual signatures (old and new keys)
- Processing: MUST update alias mappings

**POST /net/id/migration/instance/request** (NRTF: migration/instance/request)

- Purpose: Process Instance migration request
- Processing: MUST perform reachability challenge on new URL

**POST /net/id/migration/instance/approve** (NRTF: migration/instance/approve)

- Purpose: Approve Instance migration after validation
- Processing: MUST broadcast migration notice

**GET /net/id/migration/aliases** (NRTF: migration/aliases/update)

- Purpose: Return current alias mappings
- Response: MUST include complete alias map with expiration times

#### 5. Network View Operations

**GET /net/id/view/root** (NRTF: view/request)

- Purpose: Return current Network Identity List root hash
- Response: MUST include signed root hash and timestamp

**GET /net/id/recurse** (NRTF: recurse/request)

- Purpose: Perform recursive peer discovery
- Parameters: `depth`, `visited` (CSV of fingerprints)
- Processing: MUST implement cycle detection

#### 6. Data Transport Coordination

**POST /net/id/search/global** (NRTF: search/query)

- Purpose: Coordinate network-wide search operations
- Processing: MUST apply schema filtering

**GET /net/id/data/catalog** (NRTF: federation/sync)

- Purpose: Provide federation synchronization catalog
- Response: MUST include timestamped data changes since specified time

**POST /net/id/cache/coordinate** (NRTF: cache/request)

- Purpose: Coordinate cache hints across network
- Processing: Advisory only, no guarantees required

**POST /net/id/federation/announce** (NRTF: federation/sync)

- Purpose: Announce federation deltas to peers
- Processing: MUST validate and distribute changes

## Schema Compliance Implementation

### Capabilities Advertisement

**GET /net/id/capabilities** (NRTF: capabilities/request)

- Purpose: Advertise server capabilities and supported schemas and versions
- Query Parameters: `type` (node or schema_providers or all) OPTIONAL
- Response Fields (ALL REQUIRED):
        - node_capabilities: Basic server capabilities
        - supported_schema_providers: List of supported schema provider strings such as "StreamingServer/1"
        - schema_compliance_matrix: Mapping of schema providers to supported operations and any version notes
- Caching: Response MAY be cached for up to 300 seconds

### Schema Processing Rules

For all data operations involving generic data:

1. Schema Validation: MUST check both `schema_provider` and `schema_version` fields for all generic data operations
2. Supported Processing:
    - MUST process data normally if the schema provider and version are supported
    - MUST validate data according to the declared schema rules
    - MUST perform indexing for public data when applicable
3. Unsupported Handling:
    - MUST forward data unchanged through federation
    - MUST NOT attempt to parse or validate data content
    - MUST maintain NRTF frame integrity during forwarding
4. Capabilities Awareness:
    - MUST publish supported schema providers and versions in capabilities
    - SHOULD consult peer capabilities when coordinating storage or search
    - MUST avoid storing custom or generic data locally unless the server is explicitly marked as compliant with the relevant schema provider and version

### Schema Storage Policy (Identity Servers)

- Identity Servers MUST explicitly blacklist schema providers and versions they refuse to store.
- If a schema provider or version is not blacklisted, the Identity Server MUST store generic data for it at least as an opaque, canonical NRTF frame with minimal metadata. Minimal metadata SHOULD include: `data_id`, `schema_provider`, `schema_version`, `visibility`, `received_at` (timestamp), `frame_hash`, and `origin` (sender identity where available).
- Processing and indexing of content remain limited to supported schema providers and versions. For non-supported but non-blacklisted schemas, the server stores the canonical frame and forwards it unchanged without attempting to parse or index content fields.
- Identity Servers MUST expose blacklisted schemas and versions via capabilities so peers can route accordingly.

### Aggregation and Validation (Receive, Publish, Sync)

When receiving generic data directly or via federation:

1. Verify NRTF signature, timestamp, and nonce replay protections.
2. Ensure `schema_provider` and `schema_version` are present. If missing, reject with SCHEMA_UNSUPPORTED and do not store.
3. If the pair is blacklisted: drop the content and optionally record a rejection event with reason, do not forward.
4. If supported: validate against schema rules, store structured record, index public content, and forward as policy allows.
5. If not supported and not blacklisted: store the canonical NRTF frame and minimal metadata, do not parse content, and forward unchanged.

When publishing or returning catalogs and indexes:

1. Include public items. For supported schemas include schema-aware index data. For stored but unsupported schemas, include only minimal metadata such as identifiers, timestamps, and hashes, never content-derived fields.
2. Ensure signatures and integrity hashes are preserved end-to-end, never re-sign on behalf of the origin for content objects.
3. Respect capability hints from requestors (for example, preferred schema providers or versions) when filtering results.

When syncing with peers (pull or push):

1. Deduplicate by `data_id` and `frame_hash`.
2. Resolve conflicts by newer timestamp and then by lexicographic hash tie-break if timestamps are equal.
3. Validate signatures and discard frames with invalid signatures.
4. For supported schemas: perform schema-level validation before accepting updates. For non-supported but non-blacklisted schemas: accept and store the canonical frames without parsing content.
5. Do not propagate items on the blacklist.

## Security Implementation Requirements

### Cryptographic Standards

- Digital Signatures: Ed25519 REQUIRED
- Hash Functions: SHA-256 REQUIRED for fingerprints
- Random Generation: Cryptographically secure REQUIRED
- Key Storage: Secure key material protection REQUIRED

### Authentication and Authorization

- Message Authentication: ALL control messages MUST have valid signatures
- Rate Limiting: REQUIRED on per-IP and per-key basis
- Replay Protection: REQUIRED using nonce and timestamp (±300s window)
- Version Validation: REQUIRED for all incoming messages

### Abuse Prevention

- Suspicious Detection: REQUIRED
- Automatic Limiting: REQUIRED for detected abuse
- Evidence Preservation: REQUIRED for audit purposes
- Quarantine Procedures: REQUIRED for problematic nodes

## Consensus Protocol Implementation

### Quorum Requirements

- Minimum Quorum: MUST be ≥2 nodes
- Maximum Quorum: MUST be <50% of healthy nodes
- Dynamic Adjustment: RECOMMENDED based on network size
- Vote Collection: MUST collect votes for minimum 300 seconds

### Delisting Process

1. Evidence Collection: MUST gather and validate evidence
2. Network Distribution: MUST distribute evidence to all healthy nodes
3. Vote Collection: MUST wait for minimum voting period
4. Quorum Verification: MUST verify quorum achievement
5. Final Notice: MUST distribute final delisting notice
6. State Update: MUST update local state and caches

## Error Handling Requirements

### Standard Error Responses

ALL endpoints MUST support these error conditions:

- VERSION_MISMATCH: API version incompatibility
- REPLAY_DETECTED: Duplicate nonce/timestamp combination
- BAD_SIGNATURE: Cryptographic verification failure
- RATE_LIMITED: Exceeded rate limiting thresholds
- SCHEMA_UNSUPPORTED: Unknown schema provider in data

### Error Response Format

- Format: MUST use NRTF error format for control messages
- HTTP Status: MUST use appropriate HTTP status codes for REST endpoints
- Logging: MUST log all errors for debugging purposes

## Performance Requirements

### Response Time Limits

- Health Endpoint: MUST respond within 5 seconds
- Registration: MUST process within 30 seconds
- View Operations: MUST respond within 10 seconds
- Consensus Operations: MUST process votes within 5 seconds

### Scalability Requirements

- Concurrent Connections: MUST support ≥100 concurrent connections
- Message Throughput: MUST process ≥10 messages per second
- Storage Growth: MUST handle network growth to 10,000 instances

## Testing and Compliance

### Required Test Coverage

- Protocol Compatibility: MUST pass all protocol conformance tests
- Security Validation: MUST pass all security test suites
- Performance Benchmarks: MUST meet all performance requirements
- Error Handling: MUST handle all specified error conditions

### Network Integration Testing

- Multi-Server Setup: MUST be tested with ≥3 Identity Servers
- Consensus Participation: MUST participate correctly in consensus
- Migration Procedures: MUST handle all migration scenarios
- Schema Compliance: MUST correctly handle supported and unsupported schemas

## References

- [Network Implementation Standard](../net/SPEC.md)
- [Instance Implementation Standard](../instance/SPEC.md)
- [NRTF Format Specification](../nrtf/SPEC.md)

All JSON fields names mirror the protocol schemas in `net/SPEC.md` and `nrtf/SPEC.md`.

## NRTF Route Reference

| REST                                    | NRTF route                        | Direction                | Notes                 |
| --------------------------------------- | --------------------------------- | ------------------------ | --------------------- |
| GET /net/id/instances                   | view/request and recurse/response | Server→Instance          | Aggregated set        |
| POST /net/id/register                   | register/instance                 | Instance→Server          | RegistrationEnvelope  |
| POST /net/id/challenge                  | challenge/reachability            | Server→Instance          | Manual challenge      |
| POST /net/id/evidence/request           | evidence/request                  | Server/Instance→Instance | Evidence fetch        |
| POST /net/id/report/instance            | report/instance                   | Server→All               | SIN_REPORT broadcast  |
| POST /net/id/report/server              | report/server                     | Instance→Servers         | SERVER_SIN_REPORT     |
| POST /net/id/consensus/vote             | consensus/vote                    | Instance/Server→Server   | Vote submission       |
| GET /net/id/consensus/poll/:id          | consensus/poll                    | Server→Instance          | Poll state            |
| POST /net/id/key/rotate                 | key/rotate                        | Server→Peers             | Key rotation          |
| GET /net/id/view/root                   | view/request                      | Instance→Server          | Root hash             |
| GET /net/id/recurse                     | recurse/request                   | Instance→Server          | Peer expansion        |
| POST /net/id/migration/server/announce  | migration/server/announce         | Server→Peers/Instances   | Start server move     |
| POST /net/id/migration/server/commit    | migration/server/commit           | Server→Peers             | Finish server move    |
| POST /net/id/migration/instance/request | migration/instance/request        | Instance→Server          | Start instance move   |
| POST /net/id/migration/instance/approve | migration/instance/approve        | Server→Instance          | Approve instance move |
| GET /net/id/migration/aliases           | migration/aliases/update          | Server→Instance          | Alias map snapshot    |
| GET /net/id/capabilities                | capabilities/request              | Any→Server               | Capabilities inquiry  |
| POST /net/id/search/global              | search/query                      | Any→Server               | Search (public)       |
| POST /net/id/search/index/sync          | search/index                      | Instance→Server          | Index delta           |
| GET /net/id/data/catalog                | federation/sync                   | Any→Server               | Public delta list     |
| POST /net/id/cache/coordinate           | cache/request                     | Instance→Server          | Cache hint            |
| POST /net/id/federation/announce        | federation/sync                   | Server→Server            | Push deltas           |

## Minimal JSON Schema Snippets (Example)

RegistrationEnvelope:

```json
{
  "instance_id":"uuid7",
  "instance_url":"https://...",
  "api_version":1,
  "nrtf_schema_version":1,
  "timestamp":"2025-08-11T12:35:01Z",
  "nonce":"<32 hex chars>",
  "capabilities":{"stream":true},
  "connected_identity_servers":{"primary":"<fingerprint>"},
  "public_key":"<base64>",
  "transport_sig_alg":"ed25519",
  "proof_of_possession_signature":"<base64>"
}
```txt

Consensus Vote:

```json
{"poll_id":"uuid7","voter_type":"instance","voter_id":"uuid7","decision":"GUILTY","signature":"<base64>"}
```

## Example SIN_REPORT (NRTF)

```txt
api 1
schema 1
id report/instance 5f6e7d8c-... 
ts 2025-08-11T12:40:00Z
nonce 11112222333344445555666677778888
sigalg ed25519
pub base64:...
sig base64:...
msg :{
     instance_id "5f6e7d8c-..."
     offending_message_uuids :{ m1 "018f..." }
     original_message_content_hash "ab12..."
     original_instance_signature "base64:..."
     server_signature "base64:..."
     observed_api_version 1
     observed_schema_version 1
     server_timestamp "2025-08-11T12:39:58Z"
     integrity_hash "cd34..."
}
```

## Operational Notes

- All control POSTs require authentication (signed NRTF frames).
- Rate limiting: per IP and per public key fingerprint.
- Replay window: 5 minutes. Duplicate (nonce, pub) denied.
- Delisting: Follows quorum rules in `net/SPEC.md`.
- Migration: During alias window treat old and new ids as the same principal for rate limits and replay detection. Reject signatures made with old key if timestamp > alias_expires_at.
- Generic data: optional coordination (search index merge, simple catalog diffs).
- Federation: pull (GET catalog) or push (announce) deltas and newest timestamp wins.

## Data Transport Coordination

Search: merge keyword docs. Simple dedupe by data_id.  
Cache: accept hints. No guarantees.  
Federation sync: diff = list of (data_id, ts, hash). Conflict: choose higher ts then hash tie‑break.
Schema compliance: Only process data for supported schema providers. Check capabilities before accepting data operations. Forward unsupported data unchanged to maintain network connectivity.

## Force-Deletion (Server Misbehavior)

If an Identity Server is found sinful (see Net spec) a SERVER_DELIST_NOTICE is distributed. Upon receipt, Instances remove it and keep its fingerprint in a revocation cache for the configured retention.
