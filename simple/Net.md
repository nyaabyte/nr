# NR Net

- Every generic data item includes a schema provider and a schema version. Example: schema_provider = "StreamingServer/1" and schema_version = 1.
- Nodes must advertise which schema providers and versions they support via the capabilities endpoint.
- When a node receives generic data:
  - If it supports the schema provider and version, it processes the data normally.
  - If it does not, it forwards the data unchanged through the rest of the Net. It does not parse or validate content beyond basic NRTF integrity.
- Identity Servers and Instances must only store custom or generic data if they are explicitly compliant with the relevant schema providers and versions.
- When answering capability requests, include the supported schema providers and versions so peers can make proper decisions about storage and routing.

This rule helps to prevent the corruption or modification of data.

## Data Storage

- When storing data to serve across the Net, ensure the signature of the server and all parameters you received remain intact.
- When retrieving data, always verify the signature and parameters of the content with the original server before use.
- Supported data is aggregated from all sources providing the set schema, version and provider and the content and integrity is validated against the schema and original server by the server syncing/retrieving the data.
