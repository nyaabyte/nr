# NoReplacement Instance Implementation Standard

## Status

This document is NORMATIVE and defines the mandatory requirements for implementing NR Net compliant Instance nodes.

## Conformance

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are normative and must be followed as such.

## Instance Requirements

### Core Functions

Instance nodes MUST implement the following functions:

1. Identity Server Registration: MUST register with one or more Identity Servers
2. Network State Caching: MUST maintain local cache of Network Identity List
3. Message Forwarding: MUST forward and validate signed control messages
4. Consensus Participation: MUST participate in network consensus when required
5. Migration Support: MUST support Instance migration procedures
6. Data Transport Services: MAY provide generic data storage and transport
7. Federation Synchronization: MUST support federation sync for network connectivity

### Transport Requirements

- Protocol: HTTPS with TLS 1.3 (or [Betanet](../../simple/compliance/Betanet.md)'s equivalent as defined in [Net](../net/SPEC.md)) REQUIRED
- Certificate Validation: REQUIRED
- Application Layer Signatures: REQUIRED for all NRTF messages
- Base Path: All endpoints MUST use `/net/inst/` prefix

## Mandatory Endpoint Implementation

### Core Network Endpoints

Instances MUST implement the following endpoints:

#### 1. Network State Management

**GET /net/inst/view** (NRTF: view/request)

- Purpose: Return cached Network Identity List information
- Response: MUST include root_hash, instance count, server count
- Performance: MUST respond within 5 seconds
- Caching: Local cache MUST be updated within 300 seconds of changes

### GET /net/inst/config

- Purpose: Return current capability set and version information
- Response: MUST include capabilities array, api_version, schema_version
- Performance: MUST respond within 2 seconds

### GET /net/inst/type

- Purpose: Return instance type for compatibility checking
- Response: MUST include type and version fields
- Usage: Identity Servers MAY use this for registration filtering

#### 2. Registration Support

**POST /net/inst/register** (NRTF: register/instance)

- Purpose: Proxy registration requests to Identity Servers
- Processing: MAY forward to Identity Servers or return prepared NRTF frame
- Validation: MUST validate RegistrationEnvelope format if processing locally

#### 3. Consensus Participation

**POST /net/inst/evidence/response** (NRTF: evidence/response)

- Purpose: Respond to evidence requests from Identity Servers
- Input: Evidence request with message UUIDs
- Response: MUST include message existence flags and hashes
- Performance: MUST respond within 30 seconds

**POST /net/inst/vote** (NRTF: consensus/vote)

- Purpose: Submit consensus votes
- Input: Vote with poll_id, voter_type, voter_id, decision, signature
- Validation: MUST verify vote eligibility before submission

**GET /net/inst/poll/:poll_id** (NRTF: consensus/poll)

- Purpose: Return poll information from local cache
- Response: MUST include current poll status and vote counts
- Caching: Poll information MUST be kept current via broadcasts

#### 4. Migration Support

**GET /net/inst/migration/aliases** (NRTF: migration/aliases/update)

- Purpose: Return current alias mappings for migration windows
- Response: MUST include complete alias map with expiration times
- Caching: Alias information MUST be updated from broadcasts

#### 5. Schema Compliance

**GET /net/inst/capabilities** (NRTF: capabilities/request)

- Purpose: Advertise instance capabilities and supported schemas and versions
- Query Parameters: `type` (node or schema_providers or all) OPTIONAL
- Response Fields (ALL REQUIRED):
  - node_capabilities: Instance specific capabilities
  - supported_schema_providers: List of supported schema provider strings such as "StreamingServer/1"
  - schema_compliance_matrix: Mapping of schema providers to supported operations and any version notes
  - instance_type: Specific instance type identifier
- Caching: Response MAY be cached for up to 300 seconds

## Optional Data Transport Endpoints

If an Instance provides data transport services, it MUST implement:

### Data Storage Operations

**POST /net/inst/data/store** (NRTF: data/store)

- Purpose: Store data with schema compliance validation
- Processing Rules:
  - MUST check both `schema_provider` and `schema_version` fields
  - MUST validate schema compliance before processing
  - MUST forward unsupported schema providers or versions unchanged
  - MUST sign stored data with instance signature

**GET /net/inst/data/retrieve/:id** (NRTF: data/retrieve)

- Purpose: Retrieve stored data with visibility and schema enforcement
- Processing Rules:
  - MUST enforce visibility restrictions
  - MUST check schema provider and version compliance
  - MUST return schema metadata with data

**DELETE /net/inst/data/delete/:id** (NRTF: data/delete)

- Purpose: Delete stored data
- Authorization: MUST verify owner signature
- Processing: MUST remove from local storage and search indexes

### Search and Indexing

**POST /net/inst/search/query** (NRTF: search/query)

- Purpose: Perform search operations with schema filtering
- Processing Rules:
  - MUST apply schema filtering to results
  - MUST respect visibility restrictions
  - MUST only search supported schema data

**POST /net/inst/search/index** (NRTF: search/index)

- Purpose: Broadcast search index updates
- Processing: MUST update local indexes for supported schemas only

### Cache Coordination

**POST /net/inst/cache/request** (NRTF: cache/request)

- Purpose: Request cache hints from network
- Processing: Advisory only, no response guarantees

**GET /net/inst/cache/status/:id** (NRTF: cache/response)

- Purpose: Return cache status for data items
- Response: MUST include current cache status and locations

### Federation Operations

**POST /net/inst/federation/sync** (NRTF: federation/sync)

- Purpose: Synchronize data changes with federation partners
- Processing Rules:
  - MUST apply schema filtering to sync operations
  - MUST handle timestamp-based conflict resolution
  - MUST support incremental synchronization

## Schema Compliance Implementation

### Processing Rules

For all data operations, Instances MUST implement the following rules:

- Retrieve schema provider and version
- Check if it is internally supported
  - Validate the data according to your own locally-hosted schemas corresponding to the version and provider.
  - If valid, store or process the data locally and update search indexes as needed.
  - If not valid, reject the data and return an error response.
- Forward the canonical NRTF frame unchanged unless blacklisted, ("pass-through" behavior).

### Identity Server Storage Policy

Instances should assume Identity Servers will store generic data for any schema provider and version that is not explicitly blacklisted. Instances MAY query server capabilities to discover current blacklists and avoid routing unsupported items to those servers when possible.

### Aggregation and Validation

When receiving generic data:

1. Verify NRTF signature, timestamp, and nonce.
2. Check `schema_provider` and `schema_version`.
3. If supported: validate against schema rules, store or process locally, index public content, and forward as policy allows.
4. If not supported: forward unchanged without parsing content.

When publishing search results or catalogs:

1. Include only supported schemas in schema-aware results. Clearly indicate provider and version.
2. Preserve origin signatures and integrity hashes.
3. Respect requester capability hints for filtering.

When syncing with peers:

1. Deduplicate by `data_id` and `frame_hash`.
2. Resolve conflicts by newer timestamp then lexicographic hash.
3. Validate signatures. For supported schemas, apply schema validation. For unsupported schemas, accept only for pass-through buffering without parsing content.

### Schema Support Declaration

- Registration Phase: MUST declare `supported_schema_providers` during registration
- Capabilities Endpoint: MUST expose current schema providers and versions via capabilities
- Dynamic Updates: MAY update schema support (requires re-registration)

### Pass-through Behavior

For unsupported schema providers or versions:

- MUST NOT attempt to parse or validate data content
- MUST maintain NRTF frame integrity during forwarding
- MUST preserve original signatures
- MUST forward to federation partners unchanged

## Security Implementation Requirements

### Authentication and Message Validation

- NRTF Verification: REQUIRED for all control messages
- Timestamp Validation: MUST enforce ±300 second window
- Nonce Protection: MUST maintain 24-hour nonce cache per public key
- Rate Limiting: MUST implement adaptive rate limiting

### Key Management

- Key Generation: MUST use cryptographically secure random number generation
- Key Storage: MUST protect private keys from unauthorized access
- Key Rotation: MUST support key rotation procedures
- Signature Algorithm: Ed25519 REQUIRED

### Abuse Prevention

- Broadcast Deduplication: MUST maintain cache of recent broadcast hashes
- Suspicious Detection: MUST detect and report unusual patterns
- Network Isolation: MUST handle delisted node removal

## Performance Requirements

### Response Time Limits

- Health Endpoints: MUST respond within 2 seconds
- View Operations: MUST respond within 5 seconds
- Evidence Responses: MUST respond within 30 seconds
- Data Operations: SHOULD respond within 10 seconds

### Scalability Requirements

- Concurrent Connections: MUST support ≥50 concurrent connections
- Message Throughput: MUST process ≥5 messages per second
- Data Storage: If providing data services, MUST handle ≥1000 data items

### Caching Requirements

- Network State Cache**: MUST cache network state for ≥60 seconds
- Broadcast Cache: MUST cache recent broadcasts for deduplication
- Alias Map Cache: MUST cache migration aliases during transition periods

## Application-Specific Extensions

### Custom Endpoint Implementation

Instances MAY implement custom endpoints under `/*` path for application-specific functionality:

- Communication Protocol: RECOMMENDED to use NRTF for inter-instance communication
- Authentication: MUST use instance public key signatures
- Streaming: MAY use NRTF-over-WebSocket or similar protocols
- Validation: MUST handle validation according to instance type

### Instance Type Compliance

- Type Declaration: MUST declare specific instance type (e.g., "StreamingServer")
- Version Compatibility: MUST handle version compatibility within type
- Feature Compliance: MUST implement all features required by declared type

## Error Handling Requirements

### Standard Error Responses

- SCHEMA_UNSUPPORTED: When receiving unsupported schema data
- INSUFFICIENT_PERMISSIONS: When authorization fails
- RATE_LIMITED: When request rate exceeds limits
- VERSION_MISMATCH: When protocol versions are incompatible

### Error Response Format

- NRTF Messages: MUST use NRTF error format
- HTTP Endpoints: MUST use appropriate HTTP status codes
- Error Logging: MUST log errors for debugging purposes

## Testing and Compliance

### Required Test Coverage

- Protocol Compatibility: MUST pass protocol conformance tests
- Schema Compliance: MUST correctly handle supported and unsupported schemas
- Security Validation: MUST pass security test suites
- Performance Benchmarks: MUST meet performance requirements

### Network Integration Testing

- Multi-Instance Setup: MUST be tested with multiple instances
- Identity Integration: MUST successfully register and participate
- Consensus Participation: MUST correctly participate in consensus operations
- Federation Sync: MUST correctly synchronize with federation partners

## References

- [Network Implementation Standard](../net/SPEC.md)
- [Identity Server Implementation Standard](../identity/SPEC.md)
- [NRTF Format Specification](../nrtf/SPEC.md)

## NRTF Route Reference

| REST                              | NRTF route               | Direction         | Notes                |
| --------------------------------- | ------------------------ | ----------------- | -------------------- |
| GET /net/inst/view                | view/request             | Server→Instance   | Cached view          |
| POST /net/inst/register           | register/instance        | Instance→Server   | RegistrationEnvelope |
| POST /net/inst/evidence/response  | evidence/response        | Instance→Server   | Evidence submission  |
| GET /net/inst/capabilities        | capabilities/request     | Any→Instance      | Capabilities inquiry |
| GET /net/inst/migration/aliases   | migration/aliases/update | Server→Instance   | Alias map snapshot   |
| POST /net/inst/data/store         | data/store               | Client→Instance   | Data storage         |
| GET /net/inst/data/retrieve/:id   | data/retrieve            | Client→Instance   | Data retrieval       |
| DELETE /net/inst/data/delete/:id  | data/delete              | Client→Instance   | Data deletion        |
| POST /net/inst/search/query       | search/query             | Client→Instance   | Search operation     |
| POST /net/inst/search/index       | search/index             | Instance→Network  | Index update         |
| POST /net/inst/cache/request      | cache/request            | Instance→Network  | Cache coordination   |
| GET /net/inst/cache/status/:id    | cache/response           | Client→Instance   | Cache status         |
| POST /net/inst/federation/sync    | federation/sync          | Instance→Instance | Federation sync      |

## NRTF Examples

Registration broadcast received:

```txt
api 1
schema 1
id broadcast/register 7f6e5d4c-...
ts 2025-08-11T12:40:10Z
nonce 22223333444455556666777788889999
sigalg ed25519
pub base64:server...
sig base64:...
msg :{ instance_public_key_fingerprint "fpr1" server_id "srv1" server_timestamp "2025-08-11T12:40:09Z" }
```

Reachability challenge response (SIGNED_CHALLENGE):

```txt
api 1
schema 1
id challenge/reachability/response 018f...
ts 2025-08-11T12:41:00Z
nonce aaaa1111bbbb2222cccc3333dddd4444
sigalg ed25519
pub base64:inst...
sig base64:...
msg :{ instance_id "018f..." challenge_nonce "9af0..." timestamp "2025-08-11T12:41:00Z" signature "base64:..." }
```

Consensus vote submission:

```txt
api 1
schema 1
id consensus/vote 3a2b1c0d-...
ts 2025-08-11T12:42:11Z
nonce 9999aaaabbbbccccddddeeeeffff0000
sigalg ed25519
pub base64:inst...
sig base64:...
msg :{ poll_id "3a2b1c0d-..." voter_type "instance" voter_id "018f..." decision "GUILTY" }
```

Minimal data/store frame:

```txt
api 1
schema 1
id data/store doc-1
ts 2025-08-11T12:34:56Z
nonce aabbccddeeff00112233445566778899
sigalg ed25519
pub base64:inst...
sig base64:...
msg :{ data_id "doc-1" visibility "public" content_type "text/plain" schema_provider "StreamingServer/1" schema_version 1 data :{ title "Hi" body "World" } }
```

Schema compliance check example:

```txt
api 1
schema 1
id capabilities/request req-1
ts 2025-08-11T12:35:00Z
nonce bbccddeeaaffbb1122334455667788aa
sigalg ed25519
pub base64:client...
sig base64:...
msg :{ requestor_id "client123" capabilities_type "schema_providers" }
```

## Local Data Structures (For Implementations)

| Name          | Purpose                                | Key Fields                             |
| ------------- | -------------------------------------- | -------------------------------------- |
| InstanceCache | Track known instances and reachability | instance_id, state, last_seen_ts       |
| ServerCache   | Track identity servers                 | server_id, fingerprint, last_view_root |
| BroadcastLog  | Deduplicate observation_id             | observation_id, received_ts            |
| DataStore     | Local data storage                     | data_id, visibility, hash              |
| SearchIndex   | Keyword index                          | token → doc list                       |
| CacheManager  | Cache metadata                         | data_id, last_access                   |
| FederationLog | Last sync checkpoints                  | peer_id, ts, hash                      |

## Validation Steps (Receive)

1. Verify server_signature.
2. Verify embedded proof_of_possession.
3. Check version compatibility.
4. Update caches and schedule challenge if new.
5. Gossip unless duplicate or policy restriction.

## Data Operations

Store: validate schema compliance first, then validate and sign, index if public.  
Retrieve: enforce visibility and schema compliance, return data or NOT_FOUND.  
Delete: owner signature required.  
Search: local match and optional federation (only for supported schemas).  
Sync: send/receive list of (data_id, ts, hash) filtered by schema support.  
Schema compliance: Before processing any data operation, check if the `schema_provider` is in the node's supported list. If not supported, forward the request through federation without processing the content. This ensures network connectivity while maintaining type safety.

## Security Notes

- Maintain sliding window of nonces per server and per instance.
- Store last 256 broadcast hashes for dedup.
- Rotate keys via `/net/inst/key/rotate` (optional local endpoint) -> forwarded as `key/rotate` route.
- Migration windows: treat aliased ids as the same subject. Share nonce namespace. Refuse actions referencing old id after alias expiry unless verifying historical signatures.
