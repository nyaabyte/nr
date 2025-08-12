# Skeletons

## NRTF

```rust
struct NRTFFrame {
    pub instance_id: Uuid7,
    pub timestamp: Timestamp,
    pub nonce: Nonce,
    pub data: Table,
}

struct NRTFError {
    pub code: u32,
    pub message: String,
}

struct NRTFResult {
    pub frame: NRTFFrame,
    pub error: Option<NRTFError>,
}
```

## Crypto, Types

Crypto and types are wrapped re-exports of existing UUID, timestamp, text nonces, PGP libraries and signatures.  
Though it increases bloat significantly, it provides a consistent and unified interface for working with components necessary for the Net to function.

## Instance/Identity Server Skeleton

```rust
use libnr::nrtf::{NRTFFrame, NRTFResult};
use libnr::crypto::{PublicKey, PrivateKey, Signature};
use libnr::types::{Nonce, Timestamp, Uuid7, Table};

pub struct AuthContext {
    pub public_key: PublicKey,
    pub private_key: PrivateKey,
}

pub enum Visibility {
    Public,
    Private,
    Restricted,
}

pub struct Delta {
    pub id: String,
    pub changed_at: Timestamp,
    pub data: Table,
    pub schema_provider: Option<String>,
    pub schema_version: Option<u32>,
}

pub struct SchemaProvider {
    pub name: String,
    pub version: u32,
    pub supported_operations: Vec<String>,
}

pub struct NodeCapabilities {
    pub basic_capabilities: Table,
    pub supported_schema_providers: Vec<SchemaProvider>,
    pub instance_type: Option<String>,
    pub blacklisted_schemas: Vec<(String, u32)>,
}

pub struct DataWithSchema {
    pub data_id: String,
    pub content_type: String,
    pub visibility: Visibility,
    pub schema_provider: String,
    pub schema_version: u32,
    pub data: Table,
}

impl NodeCapabilities {
    // Returns true if the provider and version are supported
    pub fn supports(&self, provider: &str, version: u32) -> bool;

    // Returns a list of supported operations for the given provider and version
    pub fn supported_operations(&self, provider: &str, version: u32) -> Vec<String>;

    // Returns true if the provider and version are blacklisted (primarily Identity Servers which must store all generic data)
    pub fn is_blacklisted(&self, provider: &str, version: u32) -> bool;
}

// Registration envelope (see spec)
pub struct RegistrationEnvelope {
    pub instance_id: Uuid7,
    pub instance_url: String,
    pub api_version: u32,
    pub nrtf_schema_version: u32,
    pub timestamp: Timestamp,
    pub nonce: Nonce,
    pub capabilities: Option<Table>,
    pub supported_schema_providers: Vec<String>,
    pub connected_identity_servers: Table,
    pub public_key: PublicKey,
    pub transport_sig_alg: String,
    pub proof_of_possession_signature: Signature,
}

// For migration
pub struct AliasMapEntry {
    pub old_id: String,
    pub new_id: String,
    pub expires_at: Timestamp,
}

// For NRTF errors
pub struct NRTFErrorFrame {
    pub scope: String,
    pub code: String,
    pub message: Option<String>,
}

pub trait Instance {
    // Registration and health
    fn register(&self, identity_server_url: &str, envelope: &RegistrationEnvelope, auth: &AuthContext) -> NRTFResult;
    fn send_heartbeat(&self, auth: &AuthContext) -> NRTFResult;

    // Capabilities
    fn get_capabilities(&self, capabilities_type: &str, auth: &AuthContext) -> NRTFResult<NodeCapabilities>;
    fn supports_schema_provider(&self, schema_provider: &str, schema_version: u32) -> bool;
    fn get_supported_operations(&self, schema_provider: &str, schema_version: u32) -> Vec<String>;

    // Data with schema compliance
    fn store_data(&self, data: DataWithSchema, auth: &AuthContext) -> NRTFResult;
    fn retrieve_data(&self, id: &str, supported_schemas: &[(String, u32)], auth: &AuthContext) -> NRTFResult<DataWithSchema>;
    fn delete_data(&self, id: &str, auth: &AuthContext) -> NRTFResult;

    // Schema-aware search and index
    fn search(&self, query: &str, schema_filter: Option<&[(String, u32)]>, auth: &AuthContext) -> NRTFResult<Vec<Table>>;
    fn update_search_index(&self, delta: &Delta, auth: &AuthContext) -> NRTFResult;

    // Cache
    fn request_cache_hint(&self, data_id: &str, hint: &str, auth: &AuthContext) -> NRTFResult;
    fn get_cache_status(&self, data_id: &str, auth: &AuthContext) -> NRTFResult<Table>;

    // Federation with schema filtering
    fn federation_sync(&self, since: Timestamp, schema_filter: Option<&[(String, u32)]>, auth: &AuthContext) -> NRTFResult<Vec<Delta>>;
    fn federation_announce(&self, deltas: &[Delta], auth: &AuthContext) -> NRTFResult;

    // Migration
    fn request_instance_migration(&self, new_id: &Uuid7, new_url: &str, auth: &AuthContext) -> NRTFResult;
    fn approve_instance_migration(&self, request_frame: &NRTFFrame, auth: &AuthContext) -> NRTFResult;
    fn get_alias_map(&self, auth: &AuthContext) -> NRTFResult<Vec<AliasMapEntry>>;

    // Evidence and consensus
    fn respond_evidence(&self, request_frame: &NRTFFrame, auth: &AuthContext) -> NRTFResult;
    fn vote(&self, poll_id: &Uuid7, decision: &str, signature: &Signature, auth: &AuthContext) -> NRTFResult;

    // For NRTF errors
    fn error(&self, error: NRTFErrorFrame) -> NRTFResult;
}

pub trait IdentityServer {
    // Registration and health
    fn handle_register_instance(&self, frame: &NRTFFrame) -> NRTFResult;
    fn broadcast_registration(&self, envelope: &RegistrationEnvelope, auth: &AuthContext) -> NRTFResult;
    fn challenge_reachability(&self, instance_id: &Uuid7, auth: &AuthContext) -> NRTFResult;

    // Capabilities
    fn get_capabilities(&self, capabilities_type: &str, auth: &AuthContext) -> NRTFResult<NodeCapabilities>;
    fn supports_schema_provider(&self, schema_provider: &str, schema_version: u32) -> bool;
    fn validate_schema_compliance(&self, frame: &NRTFFrame) -> bool; // MUST check provider and version
    fn is_schema_blacklisted(&self, schema_provider: &str, schema_version: u32) -> bool; // MUST drop if blacklisted, else store opaque frame at minimum

    // Data/search/federation with schema compliance
    fn handle_search_query(&self, frame: &NRTFFrame) -> NRTFResult; // MUST filter by supported schemas
    fn handle_federation_sync(&self, frame: &NRTFFrame) -> NRTFResult;
    fn handle_federation_announce(&self, deltas: &[Delta], auth: &AuthContext) -> NRTFResult;

    // Migration
    fn announce_server_migration(&self, new_key: &PublicKey, cutover_time: Timestamp, auth: &AuthContext) -> NRTFResult;
    fn commit_server_migration(&self, old_key: &PublicKey, new_key: &PublicKey, auth: &AuthContext) -> NRTFResult;
    fn get_alias_map(&self, auth: &AuthContext) -> NRTFResult<Vec<AliasMapEntry>>;

    // Evidence and consensus
    fn request_evidence(&self, instance_id: &Uuid7, message_uuids: &[Uuid7], auth: &AuthContext) -> NRTFResult;
    fn respond_evidence(&self, request_frame: &NRTFFrame, auth: &AuthContext) -> NRTFResult;
    fn poll(&self, target_type: &str, expires_at: Timestamp, auth: &AuthContext) -> NRTFResult;
    fn vote(&self, poll_id: &Uuid7, decision: &str, signature: &Signature, auth: &AuthContext) -> NRTFResult;

    // For NRTF errors
    fn error(&self, error: NRTFErrorFrame) -> NRTFResult;
}

// Example: Handling all NRTF routes
fn handle_nrtf_frame(frame: &NRTFFrame) {
    match frame.route() {
        "register/instance" => {/* Handle registration */},
        "capabilities/request" => {/* Handle capabilities request */},
        "data/store" => {/* Handle data store with schema compliance */},
        "data/retrieve" => {/* Handle data retrieve with schema compliance */},
        "data/delete" => {/* Handle data delete */},
        "search/query" => {/* Handle search with schema filtering */},
        "search/index" => {/* Handle index update */},
        "cache/request" => {/* Handle cache hint */},
        "cache/response" => {/* Handle cache status */},
        "federation/sync" => {/* Handle federation sync with schema filtering */},
        "federation/announce" => {/* Handle federation announce */},
        "migration/instance/request" => {/* Handle migration request */},
        "migration/instance/approve" => {/* Handle migration approve */},
        "migration/aliases/update" => {/* Handle alias map */},
        "evidence/request" => {/* Handle evidence request */},
        "evidence/response" => {/* Handle evidence response */},
        "consensus/poll" => {/* Handle poll */},
        "consensus/vote" => {/* Handle vote */},
        _ => {/* Unknown route: return error */},
    }
}

// Minimal aggregation helpers which may be used by implementations
fn dedupe_key(data_id: &str, frame_hash: &str) -> String { format!("{}:{}", data_id, frame_hash) }

fn resolve_conflict(ts_a: Timestamp, hash_a: &str, ts_b: Timestamp, hash_b: &str) -> bool {
    if ts_a > ts_b { return true; }
    if ts_b > ts_a { return false; }
    hash_a < hash_b
}
```
