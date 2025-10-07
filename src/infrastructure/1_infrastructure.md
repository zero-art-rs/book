# Node


Since only the centralized model is currently supported, Node is the core of the system. The main function of Node is to provide an API for creating and managing a group and obtaining historical data. Swagger documentation can be found here: [click](https://veil.distributedlab.com/swagger/).


## Group creation

There is no separate endpoint for creating a group, because all operations on the group, including creation and deletion, occur at the application layer protocol, which is described in protobuf files.

To create a group in a frame that will be sent to the backend, you must specify `GroupOperation::Init` and pass the formed public ART tree. As proof of the frame, you must provide the frame signature with your own identity key, the public key of which must be specified in the nonce field.

Now the group creation frame must be sent to the appropriate endpoint:
```http
POST /v1/group/{id}/frames
```

## Message sending

Frames are sent to the endpoint:
```http
POST /v1/group/{id}/frames
```

Frame proof depends on GroupOperation

## Message receiving

To obtain frames stored on the backend, use the same endpoint as for sending, but with the GET method. As proof that you are a member of this tree and have the right to receive frames, you must sign the group ID and random nonce with either the tree's private key or your own leaf's private key. The epoch required in the request is the epoch of the keys you used to sign the group ID and nonce.

```http
GET /v1/group/{id}/frames
```

## Obtaining an ART tree
Obtaining an ART tree may be necessary when joining a group after deriving the leaf secret from the invitation, and now you need to obtain the tree in order to be able to read and write to the group.

To do this, you must first obtain the challenge and sign it with the tree key or your own leaf secret:
```http
GET /v1/group/{id}/challenge
```

Then, with this signature, obtain the tree at the time of epoch n:

```http
GET /v1/group/{id}/{epoch}
```


```admonish note
If you obtain the tree of epoch n, then the tree key or leaf secret must also be from that epoch.
```

## Configuration

```toml
# config.toml

[centrifugo]
hmac_secret = "centrifugo_jwt_secret"
ttl = { secs = 86400, nanos = 0 }

[api]
address = "0.0.0.0:8000"
merge_changes = false

[storage]
database_url = "mongodb://mongo:27017/zkartgroup?replicaSet=rs0"
database_name = "zkartgroup"

[nats]
messages_namespace = "personal"
```

## MongoDB
MongoDB must be running in replica set mode so that MessageRelay can track frame insertion events.

```bash
# Start MongoDB in replica set mode
mongod --replSet rs0 --bind_ip_all --port 27017
```

```bash
# Initialize replica set
echo "try { rs.status() } catch (err) { rs.initiate({_id:'rs0',members:[{_id:0,host:'mongodb:27017'}]}) }" | mongosh --port 27017 --quiet
```


