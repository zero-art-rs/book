# Art Update Proofs
`zrt-art` provides two traits for proof creation and verification: `ProvableChange` and `VerifiableChange`. Their interface make proof creation and verification similar for all the changes.

`ProvableChange` provide change with method `prove`:
```rust,ignore
fn prove<'a, R>(  
    &self,  
    art: &mut PrivateZeroArt<'a, R>,  
    ad: &[u8],  
    eligibility: Option<EligibilityArtefact>,  
) -> Result<ArtProof, ArtError>;
```

Here `art` is an instance of PrivateZeroArt, (the only one used to create proofs), `ad` is an associated data and `eligibility` is an optional artefact, which describes the rights of the user to perform an operation. There are next types of `EligibilityArtefact`:
```rust,ignore
pub enum EligibilityArtefact {
    /// Just normal member of the group, needed for UpdateKey & ConfirmRemove ops.
    Member((Fr, CortadoAffine)),

    /// Owner of the group, could perform AddMember, RemoveMember
    Owner((Fr, CortadoAffine)),  

    /// Holder of a credential, needed for anonymous AddMember & RemoveMember ops.
    CredentialHolder((Fr, Credential)), 
}
```

On the other hand, trait `VerifiableChange` provide the change with method `verify`:
```rust,ignore
fn verify(
        &self,
        art: &T,
        ad: &[u8],
        eligibility_requirement: EligibilityRequirement,
        proof: &ArtProof,
    ) -> Result<(), ArtError>;
```

For verification, we pass current state of the `art`, the same associated data `ad`, corresponding `eligibility_requirement` and `proof`. `EligibilityRequirement` can also have different types:
```rust,ignore
pub enum EligibilityRequirement {
    /// Requirement only for proof of leaf possession or group membership, public key is provided
    Member(CortadoAffine),
    
    /// Requirement for the previleged rights
    Previleged((CortadoAffine, Vec<Scalar>)),
}
```

In this case, `Member` artefact corresponds to `Member` requirement at verification, when `Owner` and `CredentialHolder` artefacts correspond to `Previleged` requirement.

Implementations of `ProvableChange` and `VerifiableChange` for changes is done with dedicated methods of `zrt-zk` crate.

```admonish note
All the proofs are created specificly for elliptic curve [Cortado](./3_core.md). For that reasons all the proof creation and verification traits are implemented only for this curve.
```
## ART operation proof example
ART update proof creation and verification can be done with with `PrivateZeroArt` structure. When the user update ART, he receives `ArtOperationOutput` instead of `BranchChange`.

```rust,ignore
let output: ArtOperationOutput<CortadoAffine> = private_zero_art1
	.update_key(some_secret_key4)
	.unwrap();
```

To create a proof use dedicated prove method. `BranchChange` doesn't implement `ProvableChange`, because it lacks additional data, to create a proof.
```rust,ignore
let proof: ArtProof = output.prove(&mut private_zero_art1, ad, None).unwrap();
let change: BranchChange<CortadoAffine> = BranchChange::from(output);
```

For verification we will use `EligibilityRequirement::Member` as it is the default one in this case (this is specified by the `zero-art` protocol).
```rust,ignore
let eligibility_requirement: EligibilityRequirement = EligibilityRequirement::Member(
	private_zero_art2
		.get_node(&change.node_index)
		.unwrap()
		.get_public_key(),
);
change.verify(&private_zero_art2, associated_data, eligibility_requirement, &proof).unwrap();
```

Aggregation proof works similary.
```rust,ignore
// Create aggregation.
let mut agg = ProverChangeAggregation::default();
agg.add_member(Fr::rand(&mut rng), &mut private_zero_art1).unwrap();
...

// Create proof.
let proof = agg.prove(&mut private_zero_art1, associated_data, None).unwrap();

// Convert aaggregation to send-safe form.
let plain_agg = PlainChangeAggregation::try_from(&agg).unwrap();
```

Verification can be done for received `plain_agg` by similar means.
```rust,ignore
plain_agg.verify(&private_zero_art2, associated_data, eligibility_requirement, &proof)
        .unwrap();
```
# Schnorr signature
Crate `zrt-crypto` has methods `sign` and `verify` for Schnorr signatures. They allow the user to sign some message with several keys. For example, to sign the messages `msg: &[u8]` with some vector of keys keys `sk: Vec<Fr>` and corresponding set of public keys `Q: Vec<CortadoAffine>`, the user do the next:
```rust,ignore
use zrt_crypto::schnorr::sign;
...
let signature: Vec<u8> = sign(&sk, &Q, msg).unwrap();
```

Then for verification one can pass the signature, the same set op public keys and the message:
```rust,ignore
use zrt_crypto::schnorr::verify;
...
assert!(verify(&signature, &Q, msg).is_ok())
```

# Credentials
Credential issues, which has his `issuer_secret_key: Fr` can issue an new credential for the holder. Assume the next prerequisites:
```rust,ignore
let holder_secret_key: Fr;  
let issuer_secret_key:Fr;  
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