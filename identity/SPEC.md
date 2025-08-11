# NR Identity Server Specification

<!-- DESC -->
Uses [`NRTF`](../nrtf/SPEC.md) (NoReplacement Transmission Format)

The Identity Server is an Internet-facing node that:

1. Inevitably propagates itself through the Net and connects to other Identity Servers.
2. Maintains the authoritative (quorum-confirmed) list of Instances and peer Identity Servers.
3. Provides the list to new Instances.
4. Handles registration of Instances.
5. Handles the transmission of control messages (broadcasts, challenges, consensus votes).
6. Provides migration data to other Servers in the Net in case of compromise or decommissioning.
7. Forwards and validates signed control messages.
8. Forwards and validates Instance migration requests (From Instance→Server→Instance)
9. Coordinates generic data transport (search, cache, user data).
10. Handles basic federation sync and catalog queries.
11. Optional lightweight data discovery.

The Identity Server keeps the authoritative (quorum-confirmed) list of Instances and peer Identity Servers. It validates registrations, gossips broadcasts, challenges reachability, coordinates consensus (delisting, server misbehavior, etc.), and **facilitates the generic data transport layer** that enables searching, caching, and storing of arbitrary data across the network.

Transport: HTTPS (TLS, see [Net Flow](../net/SPEC.md#flow)). Control routes use NRTF frames as the sole signed canonical format (REST JSON forms are translation conveniences only). Prefix all REST endpoints with `/net/id/`.

<!-- ROUTES -->
## REST Endpoints (/net/id/...)

Each REST endpoint has an equivalent NRTF route (shown in `nrtf_route`). Clients MAY use either.

1. GET `/net/id/instances` (nrtf_route: `view/request` then aggregated by `recurse/response`)
    - Current known active/pending Instances.
    - Query params: `state` (optional filter: active|pending|all)
    - JSON Response:

      ```json
      {"root_hash":"ab12...","instances":[{"instance_id":"uuid7:...","state":"Active","fingerprint":"..."}]}
      ```txt

2. POST `/net/id/register` (nrtf_route: `register/instance`)

    - Executes instance registration.
    - JSON Body mirrors RegistrationEnvelope.
    - Success: 202 Accepted and broadcast_observation_id.
    - Errors: 400 VERSION_MISMATCH, 409 REPLAY_DETECTED, 429 SOFT_FAIL_THROTTLE, 403 HARD_FAIL_ABUSE.
    - NRTF Example:

      ```txt
      api 1
      schema 1
      id register/instance 018f24c2-8d30-7ab2-a4ff-a1b2c3d4e5f6
      ts 2025-08-11T12:35:01Z
      nonce 9af0d1c2e3b4a5968877665544332211
      cap capabilities:json:"[\"stream\",\"metrics\"]"
      sigalg ed25519
      pub base64:...
      sig base64:...
      msg :{ instance_url "https://inst.example" connected_identity_servers :{ main "fpr1" } }
      ```txt

3. POST `/net/id/challenge` (server-initiated typically) (nrtf_route: `challenge/reachability`)

    - Purpose: Issue reachability challenge to Instance (debug/manual trigger).
    - JSON Body: `{ "instance_id":"uuid7:..." }`
    - Response: `200 {"challenge_nonce":"..."}`

4. POST `/net/id/evidence/request` (nrtf_route: `evidence/request`)
    - Purpose: Request accused Instance evidence set.
    - Body: `{ "instance_id":"uuid7:...","message_uuids":["uuid7:...",...] }`

5. POST `/net/id/report/instance` (nrtf_route: `report/instance` → SIN_REPORT)
    - Purpose: Publish alleged Instance misbehavior.
    - Body: SIN_REPORT schema.

6. POST `/net/id/report/server` (nrtf_route: `report/server` → SERVER_SIN_REPORT)
    - Purpose: Report server misbehavior.

7. POST `/net/id/consensus/vote` (nrtf_route: `consensus/vote`)
    - Purpose: Submit vote for active poll.

8. GET `/net/id/consensus/poll/:poll_id` (nrtf_route: `consensus/poll`)
    - Purpose: Fetch poll status.

9. POST `/net/id/key/rotate` (nrtf_route: `key/rotate`)
     - Purpose: Identity Server key rotation (rare).

10. GET `/net/id/view/root` (nrtf_route: `view/request`)
     - Purpose: Get current Network Identity List root and timestamp.

11. GET `/net/id/recurse` (nrtf_route: `recurse/request`)
     - Params: `depth`, `visited` (optional, CSV of fingerprints)
     - Returns aggregated peer list.

12. POST `/net/id/migration/server/announce` (nrtf_route: `migration/server/announce`)
    - Announces upcoming server migration (old key signed). Includes new public key and planned cutover time.
    - Response: 202 if accepted. Broadcast may follow.

13. POST `/net/id/migration/server/commit` (nrtf_route: `migration/server/commit`)
    - Finalizes server migration. Both old and new signatures required.
    - Response: 200 + alias window details.

14. POST `/net/id/migration/instance/request` (nrtf_route: `migration/instance/request`)
    - Instance asks to move to new id/key/url. Old key signs request.
    - Response: 202 pending reachability challenge.

15. POST `/net/id/migration/instance/approve` (nrtf_route: `migration/instance/approve`)
    - Server confirms migration after new instance passes challenge.

16. GET `/net/id/migration/aliases` (nrtf_route: `migration/aliases/update`)
    - Returns current alias map (old_id → new_id) and hash.

<!-- COMMENT -->
### Generic Data Transport Endpoints
<!-- END -->

17. POST `/net/id/search/global` (search/query) – network search (public scope).
18. POST `/net/id/search/index/sync` (search/index) – index delta ingest.
19. GET `/net/id/data/catalog` (federation/sync) – list changed public ids since timestamp.
20. POST `/net/id/cache/coordinate` (cache/request) – optional cache hint intake.
21. GET `/net/id/user/locate/:user_id` (user/profile/get) – profile snapshot / location.
22. POST `/net/id/federation/announce` (federation/sync) – push deltas (alt to pull).

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
| POST /net/id/search/global              | search/query                      | Any→Server               | Search (public)       |
| POST /net/id/search/index/sync          | search/index                      | Instance→Server          | Index delta           |
| GET /net/id/data/catalog                | federation/sync                   | Any→Server               | Public delta list     |
| POST /net/id/cache/coordinate           | cache/request                     | Instance→Server          | Cache hint            |
| GET /net/id/user/locate/:user_id        | user/profile/get                  | Any→Server               | User profile/location |
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

- All control POSTs require authentication (mTLS or signed NRTF frames).
- Rate limiting: per IP and per public key fingerprint.
- Replay window: 5 minutes. Duplicate (nonce, pub) denied.
- Delisting: Follows quorum rules in `net/SPEC.md`.
- Migration: During alias window treat old and new ids as the same principal for rate limits and replay detection. Reject signatures made with old key if timestamp > alias_expires_at.
- Generic data: optional coordination (search index merge, simple catalog diffs).
- Federation: pull (GET catalog) or push (announce) deltas; newest timestamp wins.

## Data Transport Coordination

Search: merge keyword docs; simple dedupe by data_id.  
Cache: accept hints; no guarantees.  
User profile: lightweight table served via user/profile/get.  
Federation sync: diff = list of (data_id, ts, hash). Conflict: choose higher ts then hash tie‑break.

## Force-Deletion (Server Misbehavior)

If an Identity Server is found sinful (see Net spec) a SERVER_DELIST_NOTICE is distributed. Upon receipt, Instances remove it and keep its fingerprint in a revocation cache for the configured retention.
