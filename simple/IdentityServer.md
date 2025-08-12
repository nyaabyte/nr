# NR Simple - Identity Server

An Identity Server defines a server which allows for the registration of and coordination of applications, data and services within the NR Network.

Identity Servers should:

- Expose `/net/id/capabilities` and include supported schema providers and versions.
- Only store or coordinate generic data for schemas they support, otherwise forward unchanged.
- Respect capabilities when coordinating search and federation catalog operations.
- Enforce schema provider and version checks before indexing or caching public data.
