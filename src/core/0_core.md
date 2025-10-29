# Core

0-ART is a tree-based Continuous Group Key Agreement (CGKA) protocol, which is a modification of the existing ART, TreeKEM and MLS protocols. This protocol manages to combine their strengths.

Core is a main library of `zero-art` asynchronous verifiable group management protocol. It encapsulates the following crates:
- [cortado](https://github.com/zero-art-rs/core/blob/main/cortado) - carefully crafted elliptic curve defined over [bulletproofs](https://crates.io/crates/bulletproofs)' ristretto scalar field (e.g. 'native elliptic curve' for [bulletproofs](https://crates.io/crates/bulletproofs)).
- [zrt-zk](https://github.com/zero-art-rs/core/blob/main/zk) - zero-knowledge proofs for ART trees based on [bulletproofs](https://crates.io/crates/bulletproofs).
- [zrt-art](https://github.com/zero-art-rs/core/blob/main/art) - different ART trees routines.
- [zrt-crypto](https://github.com/zero-art-rs/core/blob/main/crypto) - various cryptographic primitives (X3DH etc.).

ART tree is a binary tree, which consists of leaves and internal nodes. both of them store some data.
```rust,ignore
pub enum ArtNode<G: AffineRepr> {
    Leaf {
        public_key: G,
        ...
    },
    Internal {
        public_key: G,
        ...
    },
}
```

The main part is `public_key`. For a leaf it is the public key of some user in a group. On the other hand, internal nodes contain `public_key`, which corresponds to the secret key of the Diffie-Hellman key exchange between its two children nodes. Computed secret is a point, but we can derive it. For example, we can take an x coordinate of a resulting point. In this way, having any leaf secret and existing tree, it is possible to recompute the secret in a root node of the tree. The knowledge of the root secret allows to derive encryption key from it to encrypt some message. With derived in such way secret anyone in a group will be able to decrypt it.

ART tree can be modified in several ways. There are methods for key update and new user addition or removal. Their main purpose is to change some leaf secret of change tree structure in some way. All the art operations are specified int the [zrt-art](https://github.com/zero-art-rs/core) crate. More details on ART operations can be found [here](./1_core.md). 

Each of the art update is baked up by the zero-knowledge proof based on [bulletproofs](https://crates.io/crates/bulletproofs). Note, that if secret key and corresponding public key of some leaf changes, then their parent node also changes. For that reason, leaf update will cause the update of the whole branch from the leaf to the root node. With our proofs, one can prove that the leaf and all the leading nodes are updated correctly. Also you prove that common secret computed with Diffie-Hellman key exchange was computed correctly. All of this is done for every level of a tree. More details about it are on the [proofs](./2_core.md) page.