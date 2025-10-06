# Reactivity

The following services are responsible for the reactivity of the system: [Message Relay](https://github.com/zero-art-rs/message-relay), [NATS JetStream](https://github.com/nats-io), and [Centrifugo](https://github.com/centrifugal/centrifugo).

## Message Relay

Message Relay subscribes to frame insertion events in MongoDB and sends new frames to the NATS JetStream message broker.

### Configuration

```toml
# config.toml

[logging]
level = "debug"

[storage]
database_url = "mongodb://mongodb:27017/zkartgroup?replicaSet=rs0"
database_name = "zkartgroup"
messages_outbox_collection_name = "messages_outbox"

[nats]
url = "nats://nats:4222"
subject = "events.>"
messages_namespace = "personal"
```

```admonish note
The database name must match the name in Node
```

## NATS JetStream

```bash
nats-server -js -sd /data -m 8222
```

```bash
echo 'Waiting for NATS to be ready...'
while ! nats server check connection --server=nats://nats:4222 2>/dev/null; do
    echo 'NATS not ready, waiting...'
    sleep 2
done
echo 'NATS is ready! Creating EVENTS stream...'
nats stream add EVENTS --subjects 'events.>' --storage file --retention limits --max-msgs=-1 --max-bytes=-1 --max-age='' --max-msg-size=-1 --dupe-window=2m --replicas=1 --server=nats://nats:4222 --defaults
echo 'EVENTS stream created successfully!'
```

## Centrifugo

### Configuration

```json
{
  "client": {
    "token": {
      "hmac_secret_key": "centrifugo_jwt_secret"
    },
    "allowed_origins": ["*"]
  },
  "admin": {
    "enabled": false,
    "password": "password",
    "secret": "secret"
  },
  "http_api": {
    "key": "secret"
  },
  "uni_sse": {
    "enabled": true
  },
  "channel": {
    "namespaces": [
      {
        "name": "personal",
        "history_size": 300,
        "history_ttl": "600s",
        "force_recovery": true,
        "presence": true
      }
    ]
  },
  "consumers": [
    {
      "enabled": true,
      "name": "nats_events",
      "type": "nats_jetstream",
      "nats_jetstream": {
        "url": "nats://nats:4222",
        "stream_name": "EVENTS",
        "subjects": ["events.>"],
        "durable_consumer_name": "centrifugo_consumer",
        "deliver_policy": "new",
        "max_ack_pending": 100
      }
    }
  ]
}
```

```admonish note
`centrifugo_jwt_secret` must be the same as in Node
```

### Running

```bash
centrifugo -c config.json
```
