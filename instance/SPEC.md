# NR Instance Specification

<!-- DESC -->
Uses [`NRTF`](../nrtf/SPEC.md) (NoReplacement Transmission Format)

The Instance is an Internet-facing node that:

1. Registers with one or more Identity Servers.
2. Serves (and optionally produces) content to other connected Instances peer-to-peer.
3. Forwards and validates signed control messages (broadcasts, challenges, consensus votes).
4. Maintains local caches of the Network Identity List.
5. Provide migration to other Instances in case of compromise or decommissioning.
6. Optional generic data store (public, private, restricted).
7. Simple search and index broadcast (public docs).
8. Optional cache hints (advisory only).
9. User data/profile information and transmission (scoped, migrates with user).
    - If you do not want to use users and/or are just storing or communicating real-time information, consider using one root user who will store (short) data via profile information or generic storage.
10. Federation sync via timestamp deltas.

Prefix all REST endpoints with `/net/inst/` for consistency. Control plane can also use NRTF routes directly over a persistent connection.

All signed control data is canonically represented as NRTF. JSON examples in this document are convenience translations and MUST NOT be re-signed independently.

<!-- ROUTES -->
## REST Endpoints (/net/inst/...)

1. GET `/net/inst/view` (nrtf_route: `view/request`)
    - Returns cached Network Identity List root and counts.
    - Response JSON (sample):

    ```json
    {"root_hash":"ab12...","instances":142,"servers":3}
    ```

2. POST `/net/inst/register` (proxy helper: normally Instance uses direct register to Identity Servers)
    - Accepts same payload as `/net/id/register` and forwards or returns prepared NRTF frame.

3. POST `/net/inst/evidence/response` (nrtf_route: `evidence/response`)
    - Supplies evidence to a requesting Identity Server.
    - Response JSON (sample):

    ```json
    {"instance_id":"uuid7","message_existence":{"uuid7":"true"},"message_hashes":{"uuid7":"ab12..."},"current_api_version":1,"signature":"<base64>"}
    ```

4. POST `/net/inst/vote` (nrtf_route: `consensus/vote`)
    - Submit consensus vote (Instance role).

5. GET `/net/inst/poll/:poll_id` (nrtf_route: `consensus/poll`)
    - Retrieve poll info from local cache (populated via broadcasts).

<!-- COMMENT -->
### Consensus Vote Submission

```json
{"poll_id":"uuid7","voter_type":"instance","voter_id":"uuid7","decision":"GUILTY","signature":"<base64>"}
```
<!-- END -->

6. GET `/net/inst/config`
    - Returns current capability set and version info.
    - Response JSON (sample):

    ```json
    {"capabilities":["cap1","cap2"],"api_version":1,"schema_version":1}
    ```

7. GET `/net/inst/type`
    - Returns the specified instance type to ensure implementations of NRTF instances do not conflict.
    - Identity Servers that restrict registration to a specific type/set of types may request this and deny registration to other Instances and remove any from lists provided to new Instances.
    - Response JSON (sample):

    ```json
    {"type":"StreamingServer","v":1}
    ```

8. GET `/net/inst/capabilities` (nrtf_route: `capabilities/request`)
    - Returns instance capabilities and supported schema providers.
    - Query params: `type` (optional filter: node|schema_providers|all)
    - Response JSON (sample):

    ```json
    {"node_capabilities":{"stream":true,"search":true},"supported_schema_providers":["StreamingServer/1","ChatServer/2"],"schema_compliance_matrix":{"StreamingServer/1":["store","retrieve","search"],"ChatServer/2":["store","retrieve"]}}
    ```

9. GET `/net/inst/migration/aliases` (nrtf_route: `migration/aliases/update`)
    - Fetch current alias mappings (old_id → new_id) for servers and instances.
    - Use to update local caches during migration windows.

<!-- COMMENT -->
### Generic Data Transport Endpoints
<!-- END -->

<!-- condensed endpoint descriptions below moved to list -->
10. POST `/net/inst/data/store` (data/store) – store public/private structured table with schema compliance.
11. GET `/net/inst/data/retrieve/:id` (data/retrieve) – fetch (enforce visibility and schema compliance).
12. DELETE `/net/inst/data/delete/:id` (data/delete) – owner/admin only.
13. POST `/net/inst/search/query` (search/query) – simple keyword/filter search.
14. POST `/net/inst/search/index` (search/index) – broadcast index delta.
15. POST `/net/inst/cache/request` (cache/request) – optional cache hint.
16. GET `/net/inst/cache/status/:id` (cache/response) – local/known cache info.
17. POST `/net/inst/user/data/store` (user/data/store) – scoped user key/value.
18. GET `/net/inst/user/data/:uid/:key` (user/data/retrieve) – retrieve if allowed.
19. GET `/net/inst/user/profile/:uid` (user/profile/get) – profile subset.
20. POST `/net/inst/federation/sync` (federation/sync) – pull/push deltas since ts.

21. - `/*`
    - Define any other routes that may be requested from other peer Instances to this Instance.
    - As NoReplacement is only a network, this is the Wild Wild West and validation/type checking will be handled by the Instance's defined type and the content of the request/response body.
    - It is recommended that you communicate over NRTF for all inter-instance communication and use proper signatures (your Instance's public key, etc).
    - If you must must stream content and need to use NRTF, see [NRTF-over-UDP](../nrtf/SPEC.md#nrtf-over-udp) which may be replicated by WebSocket or other streaming protocols.

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
| POST /net/inst/user/data/store    | user/data/store          | User→Instance     | User data storage    |
| GET /net/inst/user/data/:uid/:key | user/data/retrieve       | User→Instance     | User data retrieval  |
| GET /net/inst/user/profile/:uid   | user/profile/get         | Client→Instance   | User profile access  |
| POST /net/inst/federation/sync    | federation/sync          | Instance→Instance | Federation sync      |

<!-- EXPLANATIONS -->
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
| UserDataStore | User KV records                        | user_id, key, value_hash               |
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
User data: per (user_id, key) with visibility.  
Schema compliance: Before processing any data operation, check if the `schema_provider` is in the node's supported list. If not supported, forward the request through federation without processing the content. This ensures network connectivity while maintaining type safety.

## Security Notes

- Maintain sliding window of nonces per server and per instance.
- Store last 256 broadcast hashes for dedup.
- Rotate keys via `/net/inst/key/rotate` (optional local endpoint) -> forwarded as `key/rotate` route.
- Migration windows: treat aliased ids as the same subject. Share nonce namespace. Refuse actions referencing old id after alias expiry unless verifying historical signatures.
