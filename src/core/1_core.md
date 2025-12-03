# ART management

ART operations are similar in their nature. They are are: creation, key update, member addition and member removal. Those operations affect the leaf corresponding to some user. This modification propagates through its ancestors, all the way up to the root. In such a way the change of a leaf affects all the nodes on path from the leaf to the root of the tree. Each of ART update operations is backed up by zero-knowledge proof, which allows you to prove to a third party that ART update is performed correctly. SDK for creation and interaction with the ART tree can be found in the [core workspace](https://github.com/zero-art-rs/core). Here, elliptic curve cryptography is done with a help of [arkworks](https://arkworks.rs/) Rust crate. For proofs we use curve [Cortado](4_core.md). In general case, we denote a curve point as generic type `G`. 

`zrt-art` crate, defines several ART structures with their specifics. The ART tree itself is build with nodes of type `ArtNode`. As was said before, it is a binary tree, every node of which is a leaf node or internal node with exactly two children:
```rust,ignore
pub enum ArtNode<G> {
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

`PublicArt` is a wrapper over this tree:
```rust,ignore
pub struct PublicArt<G> {
    pub(crate) tree_root: ArtNode<G>,
    ...
}
```

`PrivateArt` is a further wrapper over `PublicArt`, with additional data. The purpose of this ART tree is to simplify the management of the state of the ART, while working with secrets of a specific user.
```rust,ignore
pub struct PrivateArt<G: AffineRepr> {  
    /// Public part of the art.
    pub(crate) public_art: PublicArt<G>,

    /// Set of secret keys on path from the user leaf to the root.
    pub(crate) secrets: ArtSecrets<G>,

    /// Index of a user leaf.
    pub(crate) node_index: ArtNodeIndex,
}
```

Note that `PublicArt` is safe to share, while `PrivateArt` should remain private because of `secrets`. There are no methods to perform operation over `PublicArt`, but it can be changed by applying changes to it. To perform operation over `PrivateArt`, one can use `ArtAdvancedOps<G, R>` trait. This trait is defined with two generic types. The first one defines curve group we are working with while the second one is a return type, so it is possible to have different output from the same method of the structure. Operation can return `BranchChange`, which contain all the necessary information for other users to update their ART, to maintain the same state of the ART tree. Methods of `ArtAdvancedOps` require mutable reference on the target struct, however `PublicArt` won't change until method `apply(...)` is called.
## ART creation
To create an ART tree one should gather a set of leaf secrets for all the members. Then `PrivateArt` can be created with `setup(...)` method:
```rust,ignore
use zrt_art::art::PrivateArt;

...

let private_art = PrivateArt::setup(&secrets).unwrap();
```

```admonish note
`PrivateArt` created in such way will use the leftmost secret key as its own. In our implementation the left most leaf will always remain the left most, so this might be useful to track the creator of the group, as his public key will always correspond to the left-most leaf in the ART tree.
```

When the `PrivateArt` is build, creator can serialize inner `PublicArt` and send data to other users. Having data of serialized `PublicArt` and any leaf secret key, user can deserialize `PublicArt` and correctly recompute root node secret. For example, serialization and deserialization can be done as next:
```rust,ignore
use postcard::{from_bytes, to_allocvec};

...

// Serialize.
let encoded_representation = to_allocvec(private_art.get_public_art()).unwrap();

// Deserialize.
let public_art: PublicArt<CortadoAffine> = from_bytes(&encoded_representation).unwrap();

// Recreate PrivateArt
let recovered_private_art = PrivateArt::new(public_art.clone(), secrets[0]).unwrap();
```

As shown above, with `PublicArt` and a leaf secret key, one can recreate `PrivateArt` from `PublicArt` with method `new(...)`. Note, that to save current state of a tree user should store all `secrets` from leaf to root. The reason is that leaf secret isn't always enough to restore correct state of the ART tree. to pass all the secrets to the PrivateArt call `restore(...)` method.
## ART management
### Key update
When the user have already received and deserialized his `PrivateArt` he might want to change his leaf secret key, as it is known to the creator of the group. One can use `update_key(...)` for this purpose.
```rust,ignore
use zrt_art::art::ArtAdvancedOps;

...

let (tk_1, change_1): (Fr, BranchChange<CortadoAffine>) = private_art.update_key(new_sk).unwrap();
```
### Applying changes
Now that the state of the art is different, we want to share it between other users. All the changes can be applied with a help of trait `ApplicableChange`. For example, with `BranchChange` this can be done as next:
```rust,ignore
use zrt_art::changes::ApplicableChange;

...

// Update PrivateArt.
change1.apply(&mut private_art).unwrap();
private_art.commit().unwrap();
```
When the change is applied to `PrivateArt` we also call `commit()`. This is used to mark that the update is finished. More about the ussage in [merge-section](#merge-changes).

If the user leaf is changed, (by key update or removal), then even if `PublicArt` can be updated correctly, it is impossible to update `PrivateArt` with `BranchChange` correctly, as the user leaf secret is different (and unknown to him). For this reason apply of usual `BranchChange` will give an error. To apply your own key update change you need to  use new secret key as change. For example:

```rust,ignore
new_sk.apply(&mut private_art).unwrap();
private_art.commit().unwrap();
```
### Add member
If we want to add  a new member to the group, we can transform some leaf of the ART into the internal node. For simplicity, all the time, left child of a new node will be set to old leaf and the right child will be the new leaf (for a new user). This behaviour can be performed with method `add_member(...)`:
```rust,ignore
use zrt_art::changes::ApplicableChange;

...

// Leaf addition.
let (tk_2, change_2): (Fr, BranchChange<CortadoAffine>) = private_art.add_member(new_sk).unwrap();
```
There are different algorithms to chose the node for extension. In our case, we search for the leaf, which will give the lowest weight imbalance to the tree. If there are several equal choices, we chose the left most one. Note that in our tree, it is possible to have blank (removed) nodes, which are still in a tree. If the is any, we replace it's public key with the new one instead of extending any existing leaf. 

### User Removal
At some point of time. we might want to remove the user or just leave from the group. In this case, there are two similar methods for those operations.
```rust,ignore
use zrt_art::changes::ApplicableChange;

...

// User removal.
let (tk_3, change_3): (Fr, BranchChange<CortadoAffine>) = private_art.remove_member(&target_path, new_sk).unwrap();

// Self removal.
let (tk_4, change_4): (Fr, BranchChange<CortadoAffine>) = private_art.leave_group(new_sk).unwrap();
```

To know which node is removed and which is not, leaves have a dedicated `status` field. Currently, there are three types of Leaf status:
```rust,ignore
pub enum LeafStatus {
    Active,
    PendingRemoval,
    Blank,
}
```
`Active` is a default state for active members of the group. If the leaf is marked as `PendingRemoval`, then the corresponding user is leaving the group, and someone in the group now can make the leaf `Blank`. The user is considered as removed, after his corresponding leaf is marked as `Blank`. Internal nodes doesn't have `LeafStatus` so retrieval of if from internal node will return `None`.

Direct removal of a node from the tree will lead to the inconsistent state, as it will be impossible to correctly update public keys. There are some exceptions, but we need a general approach. In our case, when someone removes a leaf (target leaf) from the tree, he changes the target leaf secret to the random one (unknown for target user), and update the ART as with key update operation. Now the removed user can't recompute ART root secret, as all the secrets on path from leaf to root are different and unknown to him. The downside of the approach is the user, which performed removal, now knows two secrets in the ART, which is bad for security reasons. To deal with this, other users can rewrite the node secret for the second time, but instead of replacing public keys like in usual ART update, append them together. Then all the corresponding secret keys will also be a sum of secret keys from both changes. This procedure, name confirm removal, requires weaker proofs. Also it can be performed indefinite number of times to be sure there is no user knowing secret key of the removed user leaf.

Self removal works similar to the `update_key(...)` method, as you update your leaf secret key. However, this method additionally updates the leaf status to `LeafStatus::PendingRemoval`. Now, anyone else in a group (not only owner and trusted admins) can remove the user with method `make_blank(...)`.

 Method `make_blank(target_leaf, ...)` works similar to `leave_group(...)` , but it can be used to remove other user. The target user is specified by the `target_path`, an index of the target leaf. This method changes status of the target leaf directly to `LeafStatus::Blank`.  Note that `make_blank` can behave differently, if the leaf is already marked as blank. In that case, changes will be applied by appending public keys of corresponding nodes together. This results in the target leaf secret key, and other nodes on the path, to be the sum of several secret keys created by different users. 

## Merge changes
Key update, new leaf addition and other ART changes lead to the epoch advance. Epoch, which starts from 0, can't decrease, and this might cause some issues. For example It is possible that two different users decide to update their secret keys at the same epoch, name, $e$. This will lead to error, as there are will be two different states of the ART tree for the next epoch $e + 1$. There are two ways to mitigate the problem. The easiest one is to allow only one change per epoch chosen wtih some deterministic algorithm, and discard other changes. The second way is to merge changes together (add public keys together). Unfortunately, this breaks the structure of the ART in a way it is impossible to use only leaf secret to recompute root key, so there is a need to store secrets of the whole branch.

Now about how merge is working. Assume current epoch is $e$, and there are several ART updates, which increase epoch to $e + 1$. If two or more users change some node in the ART tree, then the resulting public key of the node is a sum of all the public keys from changes. In this case, secret key is also a sum of corresponding secret keys from changes. If the node was updated exclusively by one user, than the public key can be replaced as usual. Note that the result of the merge will belong to the epoch $e+1$, not $e+2$. The reason is that there where no messages published to finalize the merge. It might be considered as finished on epoch advance.

Not all the changes can be merged. Adding a new member must be epoch exclusive, as otherwise, new user joining the group will not be able to recompute root secret key using his leaf secret.

In code merges are done with the usage of method `commit()`. All the changes applied between two commits are considered to be performed at the same epoch, so they will be merged together. For example:
```rust,ignore
...
public_art.commit().unwrap();

change1.apply(&mut public_art).unwrap();
change2.apply(&mut public_art).unwrap();
change3.apply(&mut public_art).unwrap();

public_art.commit().unwrap();

...
```

`PrivateZeroArt` operates with secrets, so merge implementation is more complex, but in code it looks the same. The only exception is applying changes created by the user himself. If user need to merge his own key update, then he can use his new leaf secret key as a change. 
## Aggregated changes
Users might want to apply several changes in a row. For example add 10 new users to the group. `AggregationContext` is a way to combine several changes into one change tree. The context, can be created as next:
```rust,ignore
use ark_std::rand::{thread_rng};
use zrt_art::art::{AggregationContext};

...

let mut agg = AggregationContext::from(private_art.clone());
```

This context have the next structure.
```rust,ignore
pub struct AggregationContext<T, G: AffineRepr> {
    pub(crate) prover_aggregation: BinaryTree<ProverAggregationData<G>>,
    pub(crate) operation_tree: T,
}
```

It contains `prover_aggregation` tree, where all the changes are stored in aggregated form, and `operation_tree`, which is a tree used to create changes.  `ArtAdvancedOps` is implemented for `AggregationContext`, so operations with aggregation have the same interface. Those operations will performed as operation over inner `operation_tree`. Changes are retreived, then applied back to `operation_tree`, to which we then call `commit()`.  Operations don't return anything, because changes are stored in the inner state of `prover_aggregation`. For example:
```rust,ignore
agg.update_key(&new_sk1).unwrap();
agg.add_member(&new_sk2).unwrap();
```

When all the desired operations are done, the aggregation can be transformed to send-safe form (without secret keys, co-path public keys, etc.) as next:
```rust,ignore
let aggregation = AggregatedChange::try_from(&agg).unwrap();
```

Now `aggregation` can be used to update ART tree:
```rust,ignore
// ART of a different user can be updated as next
aggregation.apply(&mut private_art).unwrap();
private_art.commit().unwrap();

// to apply your own key update you. need to create private change
let private_change = PrivateAggregatedChange::new(new_sk1, change.clone());
private_change.apply(&mut private_art).unwrap();
user0.commit().unwrap();
```

```admonish note
Aggregated changes are created to save resources while proving and verifying. For that reason they are incompatible with merges, and so, confirm remove is also not supported.
```

As with usual change, if aggregation contain key update of the user, than `AggregatedChange` will be applied incorrectly. To apply this aggregation one can use PrivateAggregatedChange which requires secret key used for key udpate.