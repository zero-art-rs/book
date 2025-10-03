# Application Layer Protocol

We assume classic Pub/Sub model for our e2ee collaboration application where the **service provider** is merely untrusted delivery service which **could**:

- see encrypted application layer messages
- see current ART tree structure: e.g. public key of each node of a tree
- see current ART epoch (but importantly not set it)
- set sequence number of each message and ART update as it’s a part of a protocol
- potentially lead denial of service attacks by not delivering or selectively delivering messages
- see group creator identity public key.

SP **couldn’t:**

- see the content of any protected message inside group
- impersonate any user inside group providing fake proofs
- see metadata of users inside some group (meaning their identity public keys, names and other private information) except constantly ratcheting leaf public key
- see metadata of group itself except neutral `group_id`
