HTTP Feeds
===

Asynchronously decouple systems over plain HTTP without shared infrastructure.

HTTP feeds is a minimal specification for polling events over HTTP:

- An HTTP feed provides a HTTP GET endpoint
- Returns a chronological sequence of events or aggregate updates
- Serialized in [CloudEvents](https://github.com/cloudevents/spec) event format `application/cloudevents-batch+json`
- As batched results 
- Respects the `lastEventId` query parameter to scroll through further items
- To support infinite polling for real-time feed subscriptions


Example
---

```http
GET /inventory HTTP/1.1
Host: https://example.http-feeds.org
Accept: application/json
```

```http
200 OK
Content-Type: application/cloudevents-batch+json

[{
  "specversion" : "1.0",
  "type" : "org.http-feeds.example.inventory",
  "source" : "https://example.http-feeds.org/inventory",
  "id" : "1c6b8c6e-d8d0-4a91-b51c-1f56bd04c758",
  "time" : "2021-01-01T00:00:01Z",
  "subject" : "9521234567899",
  "data" : {
    "sku": "9521234567899",
    "updated": "2022-01-01T00:00:01Z",
    "quantity": 5
  }
},{
  "specversion" : "1.0",
  "type" : "org.http-feeds.example.inventory",
  "source" : "https://example.http-feeds.org/inventory",
  "id" : "292042fb-ab04-4653-af90-19a24032bffe",
  "time" : "2021-12-01T00:00:15Z",
  "subject" : "9521234512349",
  "data" : {
    "sku": "9521234512349",
    "updated": "2022-01-01T00:00:12Z",
    "quantity": 0
  }
}{
  "specversion" : "1.0",
  "type" : "org.http-feeds.example.inventory",
  "source" : "https://example.http-feeds.org/inventory",
  "id" : "fa3e2a22-398c-4d02-ad08-9415e43178e6",
  "time" : "2021-01-01T00:00:22Z",
  "subject" : "9521234567899",
  "data" : {
    "sku": "9521234567899",
    "updated": "2022-01-01T00:00:21Z",
    "quantity": 4
  }
}]
```

Client calls again with the last processed event id.

```http
GET /inventory?lastEventId=fa3e2a22-398c-4d02-ad08-9415e43178e6 HTTP/1.1
Host: https://example.http-feeds.org
Accept: application/json
```

```http
200 OK
Content-Type: application/cloudevents-batch+json

[]
```

If no event arrived since then, an empty array is returned.

## Polling

The client can continue polling in an infinite loop.

Pseudocode:

```python
endpoint = "https://example.http-feeds.org/inventory"
lastEventId = null

while true:
  try:
    response = GET endpoint + "?lastEventId=" + lastEventId 
    for event in response:
      process event
      lastEventId = event.id
    if response is empty:
      wait N seconds 
  except:
    wait N seconds  
```

The client _must_ persist the `id` of the last processed event as lastEventId for further fetches.

The client's event processing _must_ be idempotent (_at-least-once_ delivery semantic). 
The `id` _may_ be used for idempotency checks.
