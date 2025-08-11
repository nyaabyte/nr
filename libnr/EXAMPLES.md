
# Example Usage

## NRTF Encoding/Decoding

```rust
use libnr::nrtf::{NRTFEncoder, NRTFDecoder, schema::{Schema, SchemaID, NRTF}};

#[derive(Schema, PartialEq, Debug)]
struct StreamStatus {
    status: String,
    bitrate: f32,
    viewer_count: i32,
}

impl StreamStatus {
    const SCHEMA_ID: SchemaID = SchemaID {
        name: "StreamStatus",
        field: "status",
        version: 1,
    };
    const ROUTE: &'static str = "watch/video";
}

fn nrtf_encode_decode_example() {
    let encoder = NRTFEncoder::new();
    let decoder = NRTFDecoder::new();
    let data = StreamStatus {
        status: "ok".into(),
        bitrate: 3500.0,
        viewer_count: 102,
    };
    let encoded = encoder.encode(
        &data,
        StreamStatus::SCHEMA_ID,
        StreamStatus::ROUTE,
        "--private_key--",
        "--public_key--"
    );
    // Encoded NRTF frame (see nrtf/SPEC.md for canonical form)
    /* api 1
       schema 1
       id watch/video <resource_id>
       ts <timestamp>
       nonce <nonce>
       sigalg ed25519
       pub base64:...
       sig base64:...
       msg :{ status "ok" bitrate 3500.0 viewer_count 102 } */

    let decoded: StreamStatus = decoder.decode(&encoded).unwrap();
    assert_eq!(data, decoded);
}
```

## Actix Web Example Servers

### Identity Server (Actix)

```rust
use actix_web::{web, App, HttpServer, HttpResponse, Responder};
use libnr::nrtf::{NRTFFrame, NRTFEncoder, NRTFDecoder};
use libnr::types::{Table, Uuid7, Timestamp};
use libnr::crypto::{PublicKey, PrivateKey};

// Example: Shared authentication context
struct AuthContext {
    public_key: PublicKey,
    private_key: PrivateKey,
}

async fn register_instance(body: web::Bytes, data: web::Data<AuthContext>) -> impl Responder {
    let decoder = NRTFDecoder::new();
    let frame: NRTFFrame = decoder.decode(&body).unwrap();
    // Validate, store registration, respond
    HttpResponse::Ok().body("registered")
}

async fn search_query(body: web::Bytes) -> impl Responder {
    // Parse NRTF search query, run search, return results
    HttpResponse::Ok().body("search results")
}

async fn federation_sync() -> impl Responder {
    // Return catalog delta as NRTF
    HttpResponse::Ok().body("catalog delta")
}

#[actix_web::main]
async fn main_identity() -> std::io::Result<()> {
    let auth = AuthContext {
        public_key: PublicKey::from_bytes(&[0; 32]),
        private_key: PrivateKey::from_bytes(&[0; 32]),
    };
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(auth.clone()))
            .route("/net/id/register", web::post().to(register_instance))
            .route("/net/id/search/global", web::post().to(search_query))
            .route("/net/id/data/catalog", web::get().to(federation_sync))
    })
    .bind(("127.0.0.1", 8081))?
    .run()
    .await
}
```

### Instance Server (Actix)

```rust
use actix_web::{web, App, HttpServer, HttpResponse, Responder};
use libnr::nrtf::{NRTFFrame, NRTFEncoder, NRTFDecoder};
use libnr::types::{Table, Uuid7, Timestamp};
use libnr::crypto::{PublicKey, PrivateKey};

struct AuthContext {
    public_key: PublicKey,
    private_key: PrivateKey,
}

async fn data_store(body: web::Bytes, data: web::Data<AuthContext>) -> impl Responder {
    let decoder = NRTFDecoder::new();
    let frame: NRTFFrame = decoder.decode(&body).unwrap();
    // Store data, respond
    HttpResponse::Ok().body("stored")
}

async fn data_retrieve(path: web::Path<String>) -> impl Responder {
    let id = path.into_inner();
    // Lookup data by id, return as NRTF
    HttpResponse::Ok().body(format!("retrieved: {}", id))
}

async fn search_query(body: web::Bytes) -> impl Responder {
    // Parse NRTF search query, run search, return results
    HttpResponse::Ok().body("search results")
}

async fn federation_sync(body: web::Bytes) -> impl Responder {
    // Parse NRTF sync request, return deltas
    HttpResponse::Ok().body("federation delta")
}

#[actix_web::main]
async fn main_instance() -> std::io::Result<()> {
    let auth = AuthContext {
        public_key: PublicKey::from_bytes(&[0; 32]),
        private_key: PrivateKey::from_bytes(&[0; 32]),
    };
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(auth.clone()))
            .route("/net/inst/data/store", web::post().to(data_store))
            .route("/net/inst/data/retrieve/{id}", web::get().to(data_retrieve))
            .route("/net/inst/search/query", web::post().to(search_query))
            .route("/net/inst/federation/sync", web::post().to(federation_sync))
    })
    .bind(("127.0.0.1", 8082))?
    .run()
    .await
}
```

All protocol features and routes are defined as trait methods and structs in [SKELETON.md](SKELETON.md).  
Use it as the canonical reference for implementing any NR-compliant server or client.
