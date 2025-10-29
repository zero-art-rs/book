# ART management

ART operations are similar in their nature. Main of them are: creation, key update, member addition and removal. Those operations affect the leaf corresponding to some user. This affects its parent node and so on, up to the root. In such a way the change of a leaf affects the whole branch of the tree. Each of ART update operations is backed up by zero-knowledge proof, which allows you to prove to a third party that the updated branch of the tree is updated correctly.

SDK for creation and interaction with the ART tree can be found in the [core workspace](https://github.com/zero-art-rs/core). Here, elliptic curve cryptography is done with a help of [arkworks](https://arkworks.rs/) Rust crate.

`zrt-art` crate, defines several ART structures with their specifics. They are build on top of the ART tree, which consistd of `ArtNode`s. As was shown before, it is a binary tree, every node of which is on leaf on node with two children:
```rust,ignore
pub enum ArtNode<G: AffineRepr> {
    Leaf {
        public_key: G,
        status: LeafStatus,
        metadata: Vec<u8>, 
    },
    Internal {
        public_key: G,
        l: Box<ArtNode<G>>,
        r: Box<ArtNode<G>>,
        weight: usize,
    },
}
```

`PublicART` is just a wrapper over this tree:
```rust,ignore
pub struct PublicART<G: AffineRepr> {  
    /// Referees to the root of ART tree structure.  
    pub root: Box<ARTNode<G>>,  
  
    /// Generator used to create the ART tree.  
    pub generator: G,  
}
```

`PrivateART` is a further wrapper over `PublicART`, with additional data. This simplifies the management of the state of the ART.
```rust,ignore
pub struct PrivateART<G: AffineRepr> {  
    /// Public part of the art
    pub(crate) public_art: PublicArt<G>,

    /// Set of secret keys on path from the user leaf to the root.
    #[serde(serialize_with = "ark_se", deserialize_with = "ark_de")]
    pub(crate) secrets: Vec<G::ScalarField>,

    /// Index of a user leaf.
    pub(crate) node_index: NodeIndex,
}
```

`PublicART` is safe to share, while `PrivateART` should remain private.

There are also `PublicZeroArt` and `PrivateZeroArt`. They are wrappers over corresponding trees, with additional verifier and/of proover engine for proof creation and verification. `PrivateZeroArt` also have a mutable refference to some random number generator, which is also used for proof generation.

```rust,ignore
pub struct PublicZeroArt {  
    pub(crate) public_art: PublicArt<CortadoAffine>,  
    pub(crate) verifier_engine: ZeroArtVerifierEngine,  
}  
```

```rust,ignore
pub struct PrivateZeroArt<'a, R>  
where  
    R: Rng + ?Sized,  
{  
    pub(crate) rng: &'a mut R,  
    pub(crate) private_art: PrivateArt<CortadoAffine>,  
    pub(crate) prover_engine: ZeroArtProverEngine,  
    pub(crate) verifier_engine: ZeroArtVerifierEngine,  
}
```

To work with those structures there are several traits. The main one is `ArtAdvancedOps`, which is an extension over `ArtBasicOps`. There are implementations of them for `PrivateArt` and `PrivateZeroArt`. Those traits require to perform operations with secrets. As a result they return some helper structures, which can be used update the state of other user tree:
- `BranchChange` - Usual change of some leaf along with the branch from the leaf to root.
- `ArtOperationOutput` - `BranchChange` with additional data for proof creation.
- `MergeBranchChange` - A change, which can combine several changes from different users.
- `PlainChangeAggregation`- Tree like structure, which aggregate several changes from the same user.
More about them here.
## ART creation
To create an ART one should gather a set of leaf secrets for all the members. Then `PrivateArt` can be created with `setup` method:
```rust,ignore
use zrt_art::art::art_types::PrivateArt;

...

let private_art = PrivateArt::setup(&secrets).unwrap();;
```

```admonish note
`PrivateART` created in such a way uses the leftmost secret key as its own. In our implementation the left most leaf will always remain the left-most, so this might be useful to track the creator of the group, as his public key will always correspond to the left-most leaf in the ART tree.
```

When the art is build, creator can serialize ART and send data to other users. `PublicART` and `PrivateART` have methods to serialize them as `PublicART`, which have no secret keys. Having data of serialized PublicART and any leaf secret key, other user can deserialize the `PrivateART` and correctly recompute root key secret. Serialization and deserialization can be done as next:
```rust,ignore
// Serialize.
let encoded_representation = to_allocvec(private_art.get_public_art()).unwrap();

// Deserialize.
let public_art: PublicArt<CortadoAffine> = from_bytes(&encoded_representation).unwrap();

// Recreate PrivateArt
let recovered_private_art = PrivateArt::new(public_art.clone(), secrets[0]).unwrap();
```

As shown above, with public art and a leaf secret key, one can recreate `PrivateArt` from `PublicArt` with method `new`.

`PublicZeroArt` also can be obtained from `PublicArt` with method `new`. It will create a default verifier engine for verification.

The last ART structure `PrivateZeroArt` can be created by passing to method `new` a `PrivateArt` and a mutable reference on a random number generator. It will also add a default prover and verifier engine.

## ART management
All the art operation are described in `ArtAdvancedOps` trait.  AS was said before it is implemented for `PrivateArt` and for `PrivateZeroArt`, but they work in the same way.
### Key update
When the user have already received and deserialized his private ART he might want to change his leaf secret key, as it is known to the creator of the group. Method `update_key`, changes the leaf secret key and public key.
```rust,ignore
use zrt_art::art::ArtAdvancedOps;

...

let change1: BranchChange<CortadoAffine> = private_art.update_key(new_sk).unwrap();
```
### Applying changes
Now that the state of the art is different, we want to share it between other users. Most the changes can be applied with a help of trait `ApplicableChange`. For example with `BranchChange` this can be done as next:

To apply the change one can call the next method for ART tree:
```rust,ignore
use zrt_art::changes::ApplicableChange;

...

// Update PrivateART.
change1.update(&mut private_art).unwrap();  
```

```admonish note
If the user leaf is changed, (by key update or removal), then even if `PublicART` can be updated correctly, it is impossible to update `PrivateART` correctly, as his leaf secret is invalid now and he will not be able to update his secrets.
```

### Add member
If we want to add  a new member to the group, we can transform some leaf of the ART into the internal node. Children of a new node will be old leaf and new leaf for a new user. For simplicity, all the time, the old leaf moves to the left, while new leaf will be appended to the right. This can be done with method `add_member`:
```rust,ignore
use zrt_art::changes::ApplicableChange;

...

// Leaf addition.
let change2 = private_art.add_member(new_sk).unwrap();
```
There are different algorithms to chose the node for extension. In out case, we search for the leaf, which will give the lowest weight imbalance to the tree. If there are several equal choices, we chose the left most one. Note that in our tree, it is possible to have blank (removed) nodes. If the is any, we replace it's public key with the new one instead of extending any existing leaf. 

### User Removal
In some point of time. we might want to remove the user or just leave from the group. In this case, there are two similar methods for those operations.
```rust,ignore
use zrt_art::changes::ApplicableChange;

...

// User removal.
let change3 = private_art.remove_member(&target_path, new_sk).unwrap();

// Self removal.
let change4 = private_art.leave_group(new_sk).unwrap();
```

To know which node is removed and which is not, leaves have a dedicated `status` field. Currently, there are three types of Leaf status:
```rust,ignore
pub enum LeafStatus {
    Active,
    PendingRemoval,
    Blank,
}
```
`Active` is a default state for active members of the group. If the leaf is marked as `PendingRemoval`, then the corresponding user is leaving the group, and someone in the group now can make the leaf `Blank`. The users is considered as removed, after his corresponding leaf is maked as `Blank`. Internal nodes doesn't have Leaf Status so retrieving leaf status from internal node will return `None`.

Direct removal of a node from the tree will lead to the inconsistent state, as it will be impossible to correctly update public keys. There are extreme cases when it is possible, but we need a better way with general approach. When someone removes a leaf (target leaf) from the tree, he changes the leaf secret to the random one (unknown for target user), and update the art as with key update operation. Now the removed user can't recompute ART root secret, as all the secrets on path from leaf to root are different and unknown to him. The downside of the approach is the user, which performed removal, knows two secrets in the art. This isn't beneficial for security. To deal with this, other users can rewrite the node secret for the second time, but instead of replacing public keys like in usual ART update, append them together. Then all the corresponding secret keys will also be a sum of secret keys from both changes. This procedure can be performed indefinite number of times to be sure there is no user knowing secret key of the removed user.

Self removal works similar to the `update_key` method, as you update your leaf secret key. However, this method additionally updates the leaf status to `LeafStatus::PendingRemoval`. Now, anyone else can commit the removal with method `make_blank`.

 Method `make_blank` also works similar to `update_key` , but it can be used to remove other user. The target user is specified by the `target_path`, an index of the target leaf. This method changes status of the target leaf directly to `LeafStatus::Blank`.  Note that `make_blank` can behave differently, if the leaf is already marked as blank. In that case, changes will be applied by appending public keys of corresponding nodes together. This results in the target leaf secret key, and other nodes on the path, to be the sum of several secret keys created by different users. 

## Merge changes
Key rotation, new leaf addition and other ART changes leads to the epoch advance. Epoch, which starts from 0, can't decrease. It is possible that two different users decides to updates their secret key at the same epoch, name, $e$. This will lead to error, as there are will be two different states of the ART tree for the next epoch $e + 1$. There are two ways to mitigate the problem. The easiest one is to allow only one change per epoch, and discard others using some deterministic algorithm. The second way is to merge changes together (public keys are appended together). Unfortunately, this breaks the structure of the ART in a way it is impossible to use only leaf secret to recompute root key, so there is a need to store secrets of the whole branch. Because of this, PrivateART has field `path_secrets` to store public keys of all the nodes on path from root to the corresponding leaf. 

So, how merge is working. Assume current epoch is $e$, and there are several ART updates, which increase epoch to $e + 1$. If two or more users change some node in the ART tree, then the resulting public key of the node is a sum of all the public keys from changes. In this case, secret key is also a sum of corresponding secret keys from changes. If the node was updated exclusively by one user, than the public key can be replaced as usual. Note that the result of the merge will belong to the epoch $e+1$, not $e+2$. The reason is that there where no changes published to commit the merge.

Merges can't be applied to all the changes. Adding a new member must be epoch exclusive, as otherwise, the new user, which joins the group, will not be able to recompute root secret key, using his leaf secret.

For `PublicART` and `PublicZeroArt`, which have no secret keys, merges are simple. You need to gather all the changes in one merge change, and call the update method:
```rust,ignore
let observer_merge_change = MergeBranchChange::new_for_observer(
	vec![change1, change2, change3]
);

observer_merge_change.update(&mut public_art).unwrap();
```

Private art operates with secrets, so merge becomes complex. There are two types of merge change: merge for observer and merge for participant. If the user didn't made any changes, which should be merged, he can simply use the first one:
```rust,ignore
let changes = vec![change1, change2, change3];
let observer_merge_change = MergeBranchChange::new_for_observer(changes);  

observer_merge_change.update(&mut private_art).unwrap();
```

In the other case, when the used did some change, which will be in merge, the user must create a different merge change:
```rust,ignore
let base_fork: PrivateART<Affine<Parameters>>;

...

let participant_merge_change = MergeBranchChange::new_for_participant(
	base_fork,
	applied_change,
	unapplied_changes,
);
participant_merge_change.update(&mut private_art).unwrap();
```

Here `applied_change` is a change created by the user, while `unapplied_changes` is a set of changed created by other users. This method requires to pass `base_fork`, which is the state of ART before the `applied_change`.
## Aggregated changes
The user might want to apply several changes at the same time. To do this, the user can aggregate changes into one change tree. To perform aggregation the user should create the default aggregation:
```rust,ignore
use zrt_art::types::{ProverChangeAggregation};

...

let mut agg = ProverChangeAggregation::default()
```

Then to add change to the aggregation the user can use a dedicated method of aggregation as with usual operations. Those methods affect `PrivateART` tree the same way, but they also update aggregation tree. For example:
```rust,ignore
agg.update_key(&new_sk, &mut private_art).unwrap();
```

When all the desired operations are done, the aggregation can be transformed to send-safe form (without secret keys and co-path) as next:
```rust,ignore
let plain_agg = PlainChangeAggregation::try_from(&agg).unwrap();
```

This aggregation `plain_agg` can be used to update ART tree:
```rust,ignore
// Update private art
plain_agg.update(&mut private_art).unwrap();
```

```admonish note
Aggregated changes are created to save resources while proving and verifying. For that reason they are incompatible with merges, and so, removing of the same user for the second time is not supported.
```

