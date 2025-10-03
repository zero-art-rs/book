# Infrastructure

# System design

The system comprises two core components:

- **Node:** A server-side component that **brokers** messages, **stores** encrypted data, and **manages** chat state.
- **Client:** A user-side component that **handles** message encryption, exchange, and key synchronization.

The high level of the system architecture:

![0_infrastructure_0.png](assets/0_infrastructure_0.png)

Tech stack:

- BE: Rust
- Centrifugo
- BE API: REST API
- NoSQL: MongoDB
- MessageQueue: Nats JetStream
- SDK: Rust
- Client: Rust CLI