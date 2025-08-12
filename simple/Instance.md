# NR Simple - Instance

An Instance defines an application that interfaces with Identity Servers and other applications across the NR Network.  
It allows for:

- Interoperability between different applications and services.
- Allowing itself to be listed as a server on the NR Network.
- Coordination of data and requests across the NR Network.
- Generic, typed data storage, distribution and aggregation of only supported types to resist censorship or tampering.

Instances should:

- Register with Identity Servers and declare supported schema providers and versions.
- Expose `/net/inst/capabilities` to advertise capabilities and supported schemas.
- For data store or retrieve and search:
  - Check `schema_provider` and `schema_version`.
  - Process only if supported.
  - Otherwise forward unchanged and keep signatures intact.
- Include schema metadata when returning data to clients.
