# Transistor

[Transistor](https://github.com/zero-art-rs/transistor) is a very small service that has only two endpoints: one for uploading data to the server and the other for downloading. Data is deleted after downloading.

## Uploading:
```http
POST /transistor/v1/public/transfers
```
```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "type": "transfers",
    "attributes": {
      "data": "user_data"
    }
  }
}
```

## Downloading:
```http
GET /transistor/v1/public/transfers/{transfer_id}
```
```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "type": "transfers",
    "attributes": {
      "data": "user_data"
    }
  }
}
```

## Configuration

```yaml
# export KV_VIPER_FILE=config.yaml

log:
  level: debug
  disable_sentry: true

db:
  url: postgresql://test:test@127.0.0.1:7000/test?sslmode=disable

listener:
  addr: :8000
```

## Use Case

In order to share your profile with another user, for example via a QR code or link, not all metadata and SPKs can fit into the QR code, let alone the link. One option for sharing is to add the ID and AES key to the QR code or link, encrypt the data with the specified AES key, and upload it to Transistor with the specified ID.

![Transistor use case](../assets/2_infrastructure_0.png)
