# Introduction

Modern e2e messengers (Signal, Matrix, etc.) face scalability issues and some limitations that restrict their functionality, broader adoption and overall user experience. For example, Signal which is based on its own [Private Group System](https://signal.org/blog/signal-private-group-system/) limits maximum number of users in a group to 1000 because a Signal group of size $N$ resembles $\binom{N}{2}$ peer-to-peer chats leading to overall quadratic complexity of group communication size and linear in the number of members time of encryption which is quite expensive for end users and Service Provider as well. Therefore, it’s also challenging to adopt **private channels** that work on a publisher-subscriber model, but end-to-end encryption (this property, for instance, allows for not being responsible for content moderation). Our novel **0-ART** protocol aims to fix these problems and presents a new framework for building end-to-group(e2g) private messengers in various decentralized settings.

## Asynchronous Ratchet Tree

In the heart of our **0-ART** protocol lays *Asynchronous Ratchet Tree ([ART](https://eprint.iacr.org/2017/666.pdf)) —*  a cryptographic protocol designed to enhance the efficiency of group communications in centralized or decentralized settings. It introduces a mechanism for asynchronous updates of a ratchet tree structure, providing *a forward-secrecy* without the need for all participants to be online simultaneously, even if a number of group members is substantially large. *ART* is also adopted as a core part of the *Message Layer Security ([MLS](https://datatracker.ietf.org/doc/rfc9420/))* protocol, which is used in various messaging applications (*Wire*, etc). Unfortunately, the protocol is not designed to handle malicious group members trying to disrupt the group communication, leading to *Denial-of-Service (DoS)* ****or *Sybil* attacks in decentralized settings, which leads to limitations of broader adoption of the *ART* protocol in modern messengers.

## 0-ART

Our contribution is a novel **0-ART (zero-ART)** protocol built on top of the *ART* protocol that mitigates such attacks by using zero-knowledge proofs of every group operation (*InitGroup, AddMember, RemoveMember, KeyUpdate*). The protocol allows a trustless group communication in decentralized settings where every group member(possibly malicious) proves the correctness of tree updates, making the disruption of ART state practically unfeasible. This approach improves scalability and security of *ART* without a trusted party assistance or costly *Proof-of-Work*s. 

Also, **0-ART** provides a new zk-based anonymous credentials scheme enabling dynamic group membership changes while maintaining security and privacy.

## Security Considerations

- Identity management is decoupled from the Service Provider in our protocol, allowing users to manage their own identities or delegate them to a trusted Directory Service. Therefore, the Service provider couldn’t link user identities across different chats.
- All group metadata is hidden from out-of-group users and the Service Provider.
- All messages in a group are end-to-group encrypted, so the Service Provider or out-of-group users couldn’t decrypt them or determine the authority of each message.
- Service Provider or out-of-group users couldn’t actively interfere with the group ART state without being disclosed.
- **0-ART** supports several anonymity modes inside a group: full-anonymous, partially-public (restricted to specific chat), and full-public (traceable between different mutual chats).
- The use of anonymous credentials enables users to demonstrate eligibility for group operations without disclosing their identities or other sensitive information.
- The user doesn’t require the involvement of all other group members to update their key in the tree.