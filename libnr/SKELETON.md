
# Instance/Identity Server Skeleton

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

    // Data
    fn store_data(&self, data: Table, visibility: Visibility, auth: &AuthContext) -> NRTFResult;
    fn retrieve_data(&self, id: &str, auth: &AuthContext) -> NRTFResult<Table>;
    fn delete_data(&self, id: &str, auth: &AuthContext) -> NRTFResult;

    // Search and index
    fn search(&self, query: &str, auth: &AuthContext) -> NRTFResult<Vec<Table>>;
    fn update_search_index(&self, delta: &Delta, auth: &AuthContext) -> NRTFResult;

    // Cache
    fn request_cache_hint(&self, data_id: &str, hint: &str, auth: &AuthContext) -> NRTFResult;
    fn get_cache_status(&self, data_id: &str, auth: &AuthContext) -> NRTFResult<Table>;

    // User data/profile
    fn user_data_store(&self, user_id: &str, key: &str, value: &str, auth: &AuthContext) -> NRTFResult;
    fn user_data_retrieve(&self, user_id: &str, key: &str, auth: &AuthContext) -> NRTFResult<Option<String>>;
    fn get_user_profile(&self, user_id: &str, auth: &AuthContext) -> NRTFResult<Table>;

    // Federation
    fn federation_sync(&self, since: Timestamp, auth: &AuthContext) -> NRTFResult<Vec<Delta>>;
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

    // Data/search/federation
    fn handle_search_query(&self, frame: &NRTFFrame) -> NRTFResult;
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
        "data/store" => {/* Handle data store */},
        "data/retrieve" => {/* Handle data retrieve */},
        "data/delete" => {/* Handle data delete */},
        "search/query" => {/* Handle search */},
        "search/index" => {/* Handle index update */},
        "cache/request" => {/* Handle cache hint */},
        "cache/response" => {/* Handle cache status */},
        "user/data/store" => {/* Handle user data store */},
        "user/data/retrieve" => {/* Handle user data retrieve */},
        "user/profile/get" => {/* Handle user profile */},
        "federation/sync" => {/* Handle federation sync */},
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
```
