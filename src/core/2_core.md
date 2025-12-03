# PublicArt preview
`PublicART` have a dedicated `merge_tree` struct to handle merges. Merge tree is also a binary tree as `ArtNode`, but it can have single child. Any node in this tree contains `PublicMergeData`. This structure represents change of the corresponding node in PublicArt after applying changes.
```rust,ignore
pub struct PublicMergeData<G: AffineRepr> {
    pub strong_key: Option<G>,
    pub weak_key: Option<G>,
    pub status: Option<LeafStatus>,
    pub weight_change: i32,
}
```

Here `status` is a new status of a leaf node and `weight_change` is change of the weight of the corresponding node in a tree. `strong_key` and `weak_key` defines how resulting public key of the node changes. The idea behind this is that there are several types of changes, differed by the way they are applied. We distinguish three categories of updates:
* Updates that overwrite the current key.
* Updates that may either overwrite or extend the current key, depending on conditions.
* Updates that unconditionally extend the current key.

This. can be generalized to `strong_key` and `weak_key`. When we call `commit()`, `strong_key`, if exists, will replace original one, and then `weak_key`, if exists, will be added to it. With this we can update our `merge_tree` by updating corresponding strong and weak keys.

To update the tree, user must have the latest state of it, but instead of requiring invocation of `commit()`, operations are performed over  `PublicArtPreview`. It is a wrapper over `PublicArt`, which can look in `PublicArt` root and its `merge_tree` at the same time. Internally, it behave like commit, but it doesn't change the tree. This structure has similar set of methods to PublicArt tree but cant be mutated.
## NodeIndex preview
NodeIndex can change so it also has its own preview. In our case it is an optional copy of the index. On commit, it will replace original one. `ArtNodeIndex` is a struct which handles this logic.

## ArtSecrets preview
`ArtSecrets` has more complex changes then `NodeIndex`.

```rust, ignore
pub struct ArtSecrets<G: AffineRepr> {
    secrets: Vec<G::ScalarField>,
    secrets_preview: Vec<ArtSecretPreview<G>>,
    secrets_preview_excess: Vec<ArtSecretPreviewExcess<G>>,
}
```

This structure contain `secrets`, which are current secrets of branch on path from root to leaf. `secrets_preview` and `secrets_preview_excess` store public keys, which are gathered from applied changes. The difference between them lies in `strong_key`, which can be  `None` for `secrets_preview`. The reason, is that `secrets_preview` have their corresponding original key from secrets to compute preview secret key with original secret key, while `ArtSecretPreviewExcess` is used when secrets are increasing in size. For example, when member addition applied and tree increases in size. 

To get secret root key of a tree, which isn't yet comited, one can all `preview()` method to get `ArtSecretsPreview` struct. This is a struct, which will look not only on `secrets` but also on `secrets_preview` and secrets_preview_excess, to retrieve secret keys. Internally it works similar to `PublicArtPreview`, with strong and weak secret keys.