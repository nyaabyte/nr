# NoReplacement <br/> <img alt="NoReplacement Protocol Logo" src="assets/nr.png" width="100">

### Repo privated again, I need to make the specs make sense and add more detail. Since most of this was glued together with GPT5 I will have to do a multi week audit and rework but the general concept should still remain the same.
I aim to also add verbosely defined documentation regarding “Sybil heuristics” and what metadata should be stored about nodes plus define the census voting threshold and requirements and less complex language. Since this is a big overhaul... Expect it to take a long time. In the meantime I hope another NB member hops on the gravy train to start piecing together LibNR with me so I can inevitably make a YouTube clone and get [pop-you-ler](https://youtu.be/rpFXOD_olvY).

**Federal law is NoReplacement for parenting.**

NoReplacement (`NR`) is a general-purpose decentralized network that aims to be extremely resistant to tampering, snooping and censorship through comprehensive generic data transport capabilities.  

The network provides a complete platform for storing, searching, caching, and distributing arbitrary data while maintaining strong security, access control, and consistency guarantees. Registration, Instance/Identity Server deletion, and data transport coordination are handled between Instances (servers that serve data to users) and Identity Servers (servers that serve instance identities, authentication, and data coordination to other Instances and Servers).

Instances communicate with each other peer-to-peer for both control plane operations and data plane services including content distribution, search federation, and cache coordination.

See [Net](impl/net/SPEC.md) to begin your implementation.

- If you are developing a NoReplacement Instance or general-purpose implementation, you may want to view:
  - [Instance](impl/instance/SPEC.md)
  - [NRTF](impl/nrtf/SPEC.md)

- If you are developing a NoReplacement NRTF parser or attempting to convert JSON to NRTF, you may want to view:
  - [NRTF](impl/nrtf/SPEC.md)
  - [NRTF as a JSON Replacement](impl/nrtf/SPEC.md#nrtf-as-a-json-replacement)

- If you are developing a NoReplacement Identity Server or general-purpose implementation, you may want to view:
  - [Identity Server](impl/identity/SPEC.md)
  - [NRTF](impl/nrtf/SPEC.md)

## Compared to

- Matrix protocol:
  - Matrix is a decentralized communication protocol designed for real-time communication, supporting features like end-to-end encryption and interoperability.
  - Compared to Matrix, NR does focus on real-time communication, but not-so-much on encryption or interopability between separate messaging systems.

- ATProto (Bluesky)
  - The AT Protocol (ATProto) focuses on modularity and user data portability. It is oriented more towards social media usage. It uses a federated architecture with Personal Data Servers and relays for indexing.
  - NR is most similar to ATProto as it uses relays, individual data servers and emphasizes migration of servers and data. However, social media is not the main focus. NR's main focus is censorship resistance and transport of generic data.

- ActivityPub (Mastodon)
  - ActivityPub is a W3C standard for decentralized data transport, widely used among federated networks.
  - Compared to ActivityPub, NR does align more with ActivityPub's possible uses for generic data transport even if it is primarily used for social networks, but NR emphasizes real-time data and migration of servers as well.

- Betanet (Raven Dev Team)
  - Though not a real social-media supporting "protocol" as is Matrix/ATProto/ActivityPub, Betanet provides a decentralized "Internet 2" that meets the same anti-censorship standards we expect. NoReplacement's Net should be compatible and compliant with Betanet and should not work against it.
  - In LibNR (official implementation), HTTP transport is built-in to simplify communication between Instances and Identity Servers and will need to be adapted when Betanet reaches maturity. A grace period for the NR protocol will likely be put in place to allow a smooth transition to Betanet for intra-Net communication.
  - Since Instance-Server communication is handled internally by the library and is unified across implementations, this transition should be relatively seamless for developers. Your website will still most likely be using HTTP to serve data to clients.
  - However, you will likely have to adapt to a future Betanet-compatible web server library to define routes if you do not use a separate server to handle the exposing of HTX/HTTP endpoints. It should, however, be straightforward as LibNR and other libraries do most of the communication for you.

## Projects

### Implementations

- [LibNR](libnr/) is the official NoReplacement library implementation for building easy-ish Identity Servers and Instances with custom schemas and types.

### Projects using NoReplacement

- [AIR (Allie's Internet Radio)](https://github.com/microcrit/air.git) will use NoReplacement for its data storage and peer-to-peer audio/video streaming upon its maturity.

<!-- NOTES -->
<!--
For developers:
The Rust gitignore is reserved for a future "main" implementation.
  -->
<!-- END -->
