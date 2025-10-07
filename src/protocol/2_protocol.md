# Protocol description

Main unit of transmission between client and SP is `Frame` that encodes group operation `Frame.group_operation` and optional message `Frame.protected_payload` to group. Upon receiving a `Frame` SP validates a `Frame.proof`, assigns to `Frame` a `seq_num` and stores it in DB. Each user then might acquire a list of `SPFrame` from SP for given group.

Here we describe a protocol for e2e collaborative work on document, so semantically we assume the following duality:

- Group ↔ Document
- Message ↔ CRDT incremental update

## Initialization

In order to create a group (document) with identifier `group_id` the owner delivers to SP via `POST /v1/group/{group_id}/frames`:

```protobuf
Frame { // initial group frame
	frame: FrameTBS {
		epoch: 0,
		group_operation: GroupOperation {
			init: <serialized signed initial ART tree>
		},
		protected_payload: Encrypt(ProtectedPayload {
			payload: ProtectedPayloadTBS {
				seq_num: 0,
				payload: [
					GroupActionPayload {
						init: GroupInfo { <description of a group, except users' public keys> }
					},
					CRDTPayload: {
						full_document: <full serialized CRDT document>
					}
				]
			},
			signature: Sign(identity_public_key, SHA3(payload.serialize()))
		}.serialize())
	},
	proof: Sign(identity_public_key, SHA3(frame.serialize()))
}
```

And sends to each invited member their `Invite` using some private channel. 

## Joining group

Invitee then acquires challenge from SP via `GET /v1/group/challenge` and delivers signed `GET /v1/group/:id/:epoch` request on which SP responds with ART tree a user uses to join the group by deriving root key for particular epoch(0 if member is added in initial ART), obtaining historic frames, checking the first initial frame’s initial ART corresponds to ART from SP if user is added from initial phase and sending to the group the following frame:

```protobuf
Frame { // joining group frame
	frame: FrameTBS {
		epoch: <current_epoch + 1>,
		group_operation: GroupOperation {
			key_update: <branch update>
		},
		protected_payload: Encrypt(ProtectedPayload {
			payload: ProtectedPayloadTBS {
				seq_num: <current_seq_num + 1>,
				payload: [
					GroupActionPayload {
						join_group: User { <user full info including identity_public_key> }
					},
				]
			},
			signature: Sign(identity_public_key, SHA3(payload.serialize()))
		}.serialize())
	},
	proof: ARTProve(frame.group_operation, associated_data = SHA3(frame.serialize()))
}
```

## Sending a message to group

Let a group member wants to send a message containing some document change to the group. They could do it in different ways depending on a sender of previous message in the group:

- if previous frame sender is current user then frame takes the following format:
    
    ```protobuf
    Frame { // sending a consequtive frame (previous frame was from the same user))
    	frame: FrameTBS {
    		epoch: <current_epoch>, // stay the epoch untouched
    		// group_operation intentionally skipped
    		protected_payload: Encrypt(ProtectedPayload {
    			payload: ProtectedPayloadTBS {
    				seq_num: <current_seq_num + 1>,
    				payload: [
    					CRDTPayload: {
    						incremental_change: <incremental change of CRDT document>
    					}
    					// user could also include ChatPayload
    				]
    			},
    			signature: Sign(identity_public_key, SHA3(payload.serialize()))
    		}.serialize())
    	},
    	proof: Sign(current_art_root_key, SHA3(frame.serialize()))
    }
    ```
    
- if previous frame sender is another user current user must propagate the epoch:
    
    ```protobuf
    Frame { // sending frame with epoch propagation (previous frame was from another user)
    	frame: FrameTBS {
    		epoch: <current_epoch + 1>, // increment the epoch
    		group_operation: GroupOperation {
    			key_update: <branch update> // perform ART branch update 
    		},
    		protected_payload: Encrypt(ProtectedPayload {
    			payload: ProtectedPayloadTBS {
    				seq_num: <current_seq_num + 1>,
    				payload: [
    					CRDTPayload: {
    						incremental_change: <incremental change of CRDT document>
    					}
    					// user could also include ChatPayload
    				]
    			},
    			signature: Sign(identity_public_key, SHA3(payload.serialize()))
    		}.serialize())
    	},
    	proof: ARTProve(frame.group_operation, associated_data = SHA3(frame.serialize()))
    }
    ```
    

## Inviting new member

Each user with appropriate role has a right to perform group management operation, especially inviting new members to group or removing existing members. By convention group owner owes a leftmost leaf in ART tree so **SP** could easily check an eligibility of that group operation by checking *PoK* of leftmost secret leaf (already included in `AddMember` and `RemoveMember` proofs). For more complicated case of an administrator (not owner) changing membership we propose to include credential presentation proof as a proof of eligibility, but it’s a subject for separate topic.

In order to invite a new member to a group invitor sends `Invite` to invitee by private channel and posts the following `Frame` to group:

```protobuf
Frame { // invitational group frame
	frame: FrameTBS {
		epoch: <current_epoch + 1>,
		group_operation: GroupOperation {
			add_member: <branch update for new member>
		},
		protected_payload: Encrypt(ProtectedPayload {
			payload: ProtectedPayloadTBS {
				seq_num: <current_seq_num + 1>,
				payload: [
					GroupActionPayload {
						invite_member: User { <invitor introduces new user to group setting Role and optionally PublicKey> }
					},
					CRDTPayload: {
						full_document: <full serialized CRDT document so that invitee could see the full doc without breaking Forward Secrecy>
					}
				]
			},
			signature: Sign(identity_public_key, SHA3(payload.serialize()))
		}.serialize())
	},
	proof: ARTProve(frame.group_operation, associated_data = SHA3(frame.serialize()))
}
```

## Removing existing member

In a group each user might leave the group intentionally or be removed by other group member with appropriate rights.

If user decides to remove themselves from a group they post the following `Frame`:

```protobuf
Frame { // group leaving frame
	frame: FrameTBS {
		epoch: <current_epoch>,
		group_operation: GroupOperation {
			leave_group: <user index>,
		},
		protected_payload: Encrypt(ProtectedPayload {
			payload: ProtectedPayloadTBS {
				seq_num: <current_seq_num + 1>,
				payload: [
					GroupActionPayload {
						leave_group: User { <user info> }
					}
				]
			},
			signature: Sign(identity_public_key, SHA3(payload.serialize()))
		}.serialize())
	},
	proof: Sign(current_art_leaf_key, SHA3(frame.serialize()))
}
```

In either case (if user leaves group or is removed by other member) full removal of users could be performed in two phases according to our [cryptoprotocol](https://www.notion.so/Group-operations-22c953b6504d8194a14ed3eee5cca36c?pvs=21). Firstly, a member with granted access to performing membership changes updates a branch for removing user rewriting public keys on removing user’s direct path (importantly not deleting ART tree node itself) by sending the `Frame`:

```protobuf
Frame { // removing member frame
	frame: FrameTBS {
		epoch: <current_epoch + 1>,
		group_operation: GroupOperation {
			remove_member: <branch update for removing member>
		},
		protected_payload: Encrypt(ProtectedPayload {
			payload: ProtectedPayloadTBS {
				seq_num: <current_seq_num + 1>,
				payload: [
					GroupActionPayload {
						remove_member: User { <user info> }
					},
				]
			},
			signature: Sign(identity_public_key, SHA3(payload.serialize()))
		}.serialize())
	},
	proof: ARTProve(frame.group_operation, associated_data = SHA3(frame.serialize()))
}
```

than **SP** and members of the group treat the updated leaf as *removing.* To complete removal of the leaf and assure that noone possesses its secret any member could finalize it by simply updating it once more:

```protobuf
Frame { // removing member frame
	frame: FrameTBS {
	c
		group_operation: GroupOperation {
			remove_member: <branch update for removing member once again>
		},
		protected_payload: Encrypt(ProtectedPayload {
			payload: ProtectedPayloadTBS {
				seq_num: <current_seq_num + 1>,
				payload: [
					GroupActionPayload {
						finalize_removal: User { <user info> }
					},
				]
			},
			signature: Sign(identity_public_key, SHA3(payload.serialize()))
		}.serialize())
	},
	proof: ARTProve(frame.group_operation, associated_data = SHA3(frame.serialize()))
}
```

**SP** and any member treat this update using *merge technique* introduced in [CausalART](https://www.notion.so/Causal-ART-256953b6504d808a8339cab88e3dca63?pvs=21) protocol.

## Concurrent Updates

Our protocol handles concurrent updates of the group state (`epoch` counter incremented to the same value by different members in parallel) using [CausalART](https://www.notion.so/Causal-ART-256953b6504d808a8339cab88e3dca63?pvs=21) if and only if operations commute with each other:

- `add_member + key_update`
- `key_update + key_update`
- `remove_member + key_update`
- `add_member + remove_member`
- any frame with at the same epoch without `group_operation` commutes with any `group_operation` performed by another user (although `epoch` is incremented).

In other cases such as `add_member + add_member` **SP** will fail the operation explicitly.

To handle concurrent updates of the document itself we use common *CRDT* protocol implemented in [automerge](https://automerge.org/) library.

## Metadata changes

Each user could change their profile at any moment by including the following `Payload` in `Frame`(either along with some group operation or out of it):

```protobuf
GroupActionPayload {
	change_user: User { <user could update any field except public_key> }
},
```

Metadata of the group could be changed only by the owner by including in `Frame` the following `Payload`:

```protobuf
GroupActionPayload {
	change_group: User { <owner could only update `name` and `picture`> }
},
```