# Art Update Proofs
Proofs can be managed with `zrt-zk` and `zrt-crypto` crates. There are two engines for proof creation and verification. `ZeroArtProverEngine` and `ZeroArtVerifierEngine`.
```rust,ignore
use zrt_zk::engine::{ZeroArtProverEngine, ZeroArtVerifierEngine};

...

let prover_engine = ZeroArtProverEngine::default();
let verifier_engine = ZeroArtVerifierEngine::default();
```

Proof of change with `ZeroArtProverEngine` can be done as next:
```rust,ignore
let (_, change, prover_branch) = private_art.update_key(new_sk).unwrap();
	
let proof = prover_engine
	.new_context(eligibility_artefact)
	.for_branch(&prover_branch)
	.with_associated_data(associated_data)
	.prove(&mut thread_rng())
	.unwrap();
```

Here, `eligibility_artefact` is an artefact, which describes the rights of the user to perform an operation. Prover branch is a collection of public keys and secrets returned by the operation for proof creation. Currently there are two methods: `for_branch(...)` and `for_aggregation(...)`. There are three types of `EligibilityArtefact`:
```rust,ignore
use cortado::{CortadoAffine, Fr};
use zrt_zk::cred::Credential;

...

pub enum EligibilityArtefact {
    /// Just normal member of the group, needed for UpdateKey & ConfirmRemove ops.
    Member((Fr, CortadoAffine)),

    /// Owner of the group, could perform AddMember, RemoveMember
    Owner((Fr, CortadoAffine)),  

    /// Holder of a credential, needed for anonymous AddMember & RemoveMember ops.
    CredentialHolder((Fr, Credential)), 
}
```

Now, proof verification can be done with verifier engine as next:
```rust,ignore
let verification_result = verifier_engine
	.new_context(eligibility_requirement)
	.with_associated_data(associated_data)
	.for_branch(&public_art.verification_branch(&change).unwrap())
	.verify(&proof);
```

For verification, we pass corresponding `eligibility_requirement`, the same `associated_data`, verification branch or tree and `proof`. `EligibilityRequirement` can be one of the next two types:
```rust,ignore
use cortado::CortadoAffine;
use curve25519_dalek::Scalar;

...

pub enum EligibilityRequirement {
    /// Requirement for proof of leaf possession or group membership, public key is provided
    Member(CortadoAffine),
    
    /// Requirement for the previleged rights
    Previleged((CortadoAffine, Vec<Scalar>)),
}
```

In this case, `Member` artefact corresponds to `Member` requirement at verification, when `Owner` and `CredentialHolder` artefacts correspond to `Previleged` requirement.

```admonish note
All the proofs are created specificly for elliptic curve [Cortado](./3_core.md). For that reasons all the proof creation and verification traits are implemented only for this curve.
```

## Example
ART update proof creation and verification can be done with `PrivateArt` structure. As was said before, when the user update `PrivateArt`, he receives `prover_branch` of type `Vec<ProverNodeData<G>>` along with new tree key and `BranchChange`. This proof can be serialized with crate `ark_serialize`. For example:

```rust,ignore
use ark_serialize::{CanonicalDeserialize, CanonicalSerialize};

let prover_engine = ZeroArtProverEngine::default();

...

let sk = Fr::rand(&mut rng);
let (_, change, prover_branch) = private_art.update_key(sk).unwrap();

let leaf_sk = private_art.secrets().preview().leaf();
let leaf_pk = CortadoAffine::generator().mul(leaf_sk).into_affine();
let proof = prover_engine
    .new_context(EligibilityArtefact::Member((leaf_sk, leaf_pk)))
    .for_branch(&prover_branch)
    .with_associated_data(associated_data)
    .prove(&mut thread_rng())
    .unwrap();
```

For verification we will use `EligibilityRequirement::Member` as it is the default one in this case (this is specified by the `zero-art` protocol).
```rust,ignore
let leaf_pk = private_art
	.preview()
	.node(&append_user_changes.node_index)
	.unwrap()
	.public_key();

let verification_branch = public_art
	.preview()
	.verification_branch(&change)?;

let verification_result = verifier_engine  
    .new_context(EligibilityRequirement::Member(leaf_pk))  
    .with_associated_data(&associated_data)  
    .for_branch(&verification_branch)  
    .verify(&proof);
```

Aggregation proof work similary.
```rust,ignore
// Create aggregation.
let mut agg = ProverChangeAggregation::from(private_art.clone());
agg.add_member(Fr::rand(&mut rng)).unwrap();
...

// Create proof.
let leaf_sk = private_art.secrets().preview().leaf();
let leaf_pk = CortadoAffine::generator().mul(leaf_sk).into_affine();
let eligibility_artefact = EligibilityArtefact::Owner((leaf_sk, leaf_pk));
let proof = prover_engine
	.new_context(eligibility_artefact)
	.for_aggregation(&ProverAggregationTree::try_from(&agg).unwrap())
	.with_associated_data(associated_data)
	.prove(&mut thread_rng())
	.unwrap();

// Convert aaggregation to send-safe form.
let change = AggregatedChange::try_from(&agg).unwrap();
```

Verification can be done for received `aggregation` by similar means.
```rust,ignore
let verification_tree = private_art
	.preview()
	.verification_tree(&change).unwrap();
	
let verification_result = verifier_engine
	.new_context(EligibilityRequirement::Previleged((leaf_pk, vec![])))
	.for_aggregation(&verification_tree)
	.with_associated_data(associated_data)
	.verify(&proof);
```
# Schnorr signature
Crate `zrt-crypto` has methods `sign(...)` and `verify(...)` for Schnorr signatures. They allow the user to sign some message with several keys. For example, to sign the messages `msg: &[u8]` with some vector of keys keys `sk: Vec<Fr>` and corresponding set of public keys `Q: Vec<CortadoAffine>`, one can do the next:
```rust,ignore
use zrt_crypto::schnorr::sign;
...
let signature: Vec<u8> = sign(&sk, &Q, msg).unwrap();
```

Then for verification one can pass the signature, the same set of public keys and the same message:
```rust,ignore
use zrt_crypto::schnorr::verify;
...
assert!(verify(&signature, &Q, msg).is_ok())
```

# Credentials
Credential issuer, which has his `issuer_secret_key: Fr` can issue an new credential for the holder. Assume the next prerequisites:
```rust,ignore
let holder_secret_key: Fr;  
let issuer_secret_key: Fr;  
let validity_period: u64;
let holder_public_key: CortadoAffine;  
let revocation_list: Vec<Scalar>;
```

the credential issuer can issue a credential as next:
```rust,ignore
use zrt_zk::cred::Credential;

// Issue a credential
let credential =  
    Credential::issue(issuer_secret_key, validity_period, holder_public_key).unwrap();  
  
// Verify the credential  
assert!(credential.verify().is_ok());
```

Then the holder can present the credential presentation proof with the next method:
```rust,ignore
// Create a credential presentation proof  
let proof = credential  
    .present(holder_secret_key, revocation_list.clone())  
    .unwrap();  
  
// Verify the credential presentation proof  
assert!(Credential::verify_presentation(&proof, revocation_list).is_ok());
```