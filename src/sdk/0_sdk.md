# Client SDK

To interact with 0-ART groups, each user must have an identity key pair: `identity_secret_key` and `identity_public_key`. Hereinafter, we will consider either `identity_public_key` or `SHA3-256(identity_public_key)[..16]` to be the user identifier. Within each group, the user has a `leaf_secret`, which is used to derive the tree key (tk) and stage key (stk). `leaf_secret` is not permanent and can change so that the cryptosystem has forward secrecy. In addition to `leaf_secret`, the user must store the `stk` and `epoch` of the group.

Example of how to generate a key pair:
```rust,ignore
use cortado::{self, CortadoAffine, Fr as ScalarField};
use ark_std::rand::prelude::StdRng;
use ark_std::rand::thread_rng;
use ark_std::UniformRand;
use ark_ec::{AffineRepr, CurveGroup};
...
let mut rng = StdRng::from_rng(thread_rng()).unwrap();
let identity_secret_key = ScalarField::rand(&mut rng);
let identity_public_key = (CortadoAffine::generator() * identity_secret_key).into_affine();
```

The SDK provides a tool for creating, using, and managing groups based on 0-ART trees. Let's look at the high-level components of a group:
```rust,ignore
#[derive(Debug, Clone, Default)]
pub struct User {
    id: String,
    name: String,
    public_key: CortadoAffine,
    metadata: Vec<u8>,
    ...
}
```
The metadata and name refer to metadata. The `public_key` is the user's `identity_public_key`, which the user will use to sign the payload. In order to distinguish invited users from group members, the `public_key` field is set to `CortadoAffine::default()` (i.e., a point at infinity), which is related to the structure of the 0-ART tree and some other operations. The `id` is the first 16 bytes of the SHA3-256(public_key) and SHA3-256(leaf_key) hash in hex without 0x for group members and invited members, respectively.

```rust,ignore
#[derive(Debug, Default, Clone)]
pub struct GroupInfo {
    id: Uuid,
    name: String,
    created: DateTime<Utc>,
    metadata: Vec<u8>,
    members: GroupMembers,
}
```
For `GroupInfo`, as in `User`, the name, metadata, and created fields refer to metadata. The Group ID is set by the owner during creation and remains unchanged throughout the group's existence.

## Group creation

To create a new group, use the method `GroupContext::new(identity_secret_key: ScalarField, group_info: GroupInfo) -> Result<(Self, Frame)>`.

Note that `GroupMembers` must contain a `User` with your `identity_public_key`, for example:

```rust,ignore
let mut rng = StdRng::from_rng(thread_rng()).unwrap();

let identity_secret_key = ScalarField::rand(&mut rng);
let identity_public_key = (CortadoAffine::generator() * identity_secret_key).into_affine();

let owner = User::new(
    "owner".to_string(),
    identity_public_key,
    vec![],
    zero_art_proto::Role::Ownership,
);

let group_info = GroupInfo::new(
    Uuid::new_v4(),
    "group".to_string(),
    Utc::now(),
    vec![],
    vec![owner].into(),
);

let (mut group_context, initial_frame) =
    GroupContext::new(identity_secret_key, group_info).expect("Failed to create GroupContext");

```

At the moment, the group has only been created locally, and now it is necessary to send `initial_frame` to the Service Provider so that other invited participants can synchronize changes in the group through it.

## Send message

Currently, the SDK supports three types of payloads: GroupAction, CRDT, and Chat. GroupAction refers to any change in a group, such as adding or removing a member, updating group or member metadata, etc. CRDT and Chat payloads are defined in the protobuf file.

```rust,ignore
#[derive(Debug, Clone)]
pub enum Payload {
    Action(GroupActionPayload),
    Crdt(zero_art_proto::crdt_payload::Payload),
    Chat(zero_art_proto::chat_payload::Payload),
}

#[derive(Debug, Clone)]
pub enum GroupActionPayload {
    Init(GroupInfo),
    InviteMember(GroupInfo),
    RemoveMember(User),
    JoinGroup(User),
    ChangeUser(User),
    ChangeGroup(GroupInfo),
    LeaveGroup(User),
    FinalizeRemoval(User),
}
```

To send a message, you need to create a frame with the desired payloads.

Note that creating a frame can sometimes take longer because `leaf_secret` is implicitly updated, which requires generating a zero-knowledge proof.

```rust,ignore
impl GroupContext {
    pub fn create_frame(
        &mut self,
        payloads: Vec<models::payload::Payload>,
    ) -> Result<models::frame::Frame> {...}
}
```

## Recieve message
Frames accepted by the Service Provider will be sent to all active participants via the SSE channel, or participants can obtain these frames by requesting them from the Node. Frames contain encrypted data and information about group updates, so in order to update the group state and read the data, it is necessary to process this frame:

```rust,ignore
impl GroupContext {
    pub fn process_frame(
        &mut self,
        frame: models::frame::Frame,
    ) -> Result<Vec<models::payload::Payload>> {...}
}
```

## Add member

To add a user to a group, you need to generate a leaf_secret for them. There are two types of invited participants: Identified and Unidentified. 

An Identified participant is someone whose identity public key we know. SPK is an optional temporary public key of the person we are inviting, which can be used, for example, to track the invitation. If the person being invited has lost their private key from the SPK, they will not be able to accept the invitation. 

Unidentified, in turn, corresponds to a user whose identity public key we do not know, aka an anonymous invitation for which we generate the private key ourselves.

```rust,ignore
#[derive(Debug, Clone, Copy)]
pub enum Invitee {
    Identified {
        identity_public_key: CortadoAffine,
        spk_public_key: Option<CortadoAffine>,
    },
    Unidentified(ScalarField),
}

impl GroupContext {
    pub fn add_member(
        &mut self,
        invitee: Invitee,
        mut payloads: Vec<Payload>,
    ) -> Result<(Frame, Invite)> {...}
}
```

## Remove member
To delete a user, you need to know their ID or `identity_public_key`.

```rust,ignore
impl GroupContext {
    pub fn remove_member(
        &mut self,
        user_id: &str,
        mut payloads: Vec<Payload>,
    ) -> Result<(Frame, Option<User>)> {...}
}
```

Each of these actions: **CreateFrame**, **AddMember**, and **RemoveMember** results in the creation of a Frame, which must be sent to the Service Provider.

## State commiting
To prevent tree states from becoming out of sync, a commit mechanism is used. Therefore, if you sent a frame and the Service Provider returned a Status code OK, you need to commit the current state:

```rust,ignore
impl GroupContext {
    pub fn commit_state(&mut self) {...}
}
```

## Accept invite
To accept an invitation, you need to create `InviteContext`. `spk_secret_key` is a private key if spk was used when creating the invitation.

```rust,ignore
impl InviteContext {
    pub fn new(
        identity_secret_key: ScalarField,
        spk_secret_key: Option<ScalarField>,
        invite: Invite,
    ) -> Result<Self> {...}
}
```

Invite context is necessary in order to be able to obtain the ART tree from the Service Provider. To do this, you need to obtain a challenge and sign it with `sign_as_leaf`, and with the signed challenge you can obtain the tree. This is necessary so that no one except the group members can obtain the tree.

```rust,ignore
impl InviteContext {
    pub fn sign_as_leaf(&self, msg: &[u8]) -> Result<Vec<u8>> {...}
}
```

Now that you have the tree, you can upgrade `InviteContext` to `PendingGroupContext`. At this stage, you have not yet accepted the invitation, and `PendingGroupContext` is necessary to synchronize the tree state, after which you will be able to accept the invitation.

```rust,ignore
impl InviteContext {
    pub fn upgrade(self, art: PublicART<CortadoAffine>) -> Result<PendingGroupContext> {...}
}
```

To synchronize the status in `PendingGroupContext`, there is a method similar to `GroupContext`: `process_frame`.

Once you have synchronized, you can try to accept the invitation:

```rust,ignore
impl PendingGroupContext {
    pub fn join_group_as(&mut self, mut user: User) -> Result<Frame> {...}
}
```

If the Service Provider has accepted the Frame, you are now in the group and can upgrade PendingGroupContext to GroupContext to unlock the ability to send frames.

```rust,ignore
impl PendingGroupContext {
    pub fn upgrade(mut self) -> GroupContext {...}
}
```

# Example

```rust,ignore
fn generate_key_pair(rng: &mut StdRng) -> (CortadoAffine, ScalarField) {
    let secret_key = ScalarField::rand(rng);
    let public_key = (CortadoAffine::generator() * secret_key).into_affine();
    (public_key, secret_key)
}

fn generate_uuid(rng: &mut StdRng) -> Uuid {
    let mut bytes = [0u8; 16];
    rng.fill_bytes(&mut bytes);
    Uuid::from_bytes(bytes)
}

...

let mut rng = StdRng::seed_from_u64(0);

let (owner_public_key, owner_secret_key) = generate_key_pair(&mut rng);
let (member_identity_public_key, member_identity_secret_key) = generate_key_pair(&mut rng);
let (member_spk_public_key, member_spk_secret_key) = generate_key_pair(&mut rng);

let owner = User::new(
    "owner".to_string(),
    owner_public_key,
    vec![],
    zero_art_proto::Role::Ownership,
);

let group_info = GroupInfo::new(
    generate_uuid(&mut rng),
    "group".to_string(),
    Utc::now(),
    vec![],
    vec![owner].into(),
);

let (mut group_context, _) =
    GroupContext::new(owner_secret_key, group_info).expect("Failed to create GroupContext");

let (frame_0, invite) = group_context
    .add_member(
        Invitee::Identified {
            identity_public_key: member_identity_public_key,
            spk_public_key: Some(member_spk_public_key),
        },
        vec![],
    )
    .expect("Failed to add member in group context");

group_context.commit_state();

let new_leaf_secret = ScalarField::rand(&mut rng);
let frame_1 = group_context
    .key_update(new_leaf_secret, vec![])
    .expect("Failed to key update");

let public_art: PublicART<CortadoAffine> = group_context.state.art.clone().into();
group_context.commit_state();

let invite_context = InviteContext::new(
    member_identity_secret_key,
    Some(member_spk_secret_key),
    invite,
)
.expect("Failed to create InviritContext");
let mut pending_group_context = invite_context
    .upgrade(public_art)
    .expect("Failed to upgrade invite context to pending group context");
pending_group_context
    .process_frame(frame_0)
    .expect("Failed to process sync frame");
pending_group_context
    .process_frame(frame_1)
    .expect("Failed to sync 2");

let user = User::new(
    "user".to_string(),
    member_identity_public_key,
    vec![],
    zero_art_proto::Role::Write,
);

let frame_2 = pending_group_context
    .join_group_as(user)
    .expect("Failed to join group");
let mut member_group_context = pending_group_context.upgrade();

group_context
    .process_frame(frame_2)
    .expect("Failed to process accept invite flow");

let new_leaf_secret = ScalarField::rand(&mut rng);
let frame_3 = member_group_context
    .key_update(new_leaf_secret, vec![])
    .expect("Failed to update key");
member_group_context.commit_state();

group_context.process_frame(frame_3).expect("Failed to process frame");
let frame_4 = group_context.create_frame(vec![]).expect("Failed to create frame");
```
