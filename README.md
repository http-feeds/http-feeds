HTTP Feeds
===

Asynchronously decouple systems with plain JSON/HTTP APIs.

HTTP feeds is a minimal specification for polling events over HTTP:

- An HTTP feed provides a HTTP GET endpoint
- Returns a chronological sequence of events or aggregate updates
- Serialized in [CloudEvents](https://github.com/cloudevents/spec) event format `application/cloudevents-batch+json`
- As batched results 
- Respects the `lastEventId` query parameter to scroll through further items
- To support infinite polling for real-time feed subscriptions

HTTP feeds can be used for event streaming and asynchronous data replication.

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

An empty array signals the end of the feed.

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


## Model

The response contains an array of events that comply with the [CloudEvents Specification](https://github.com/cloudevents/spec).

Field    | Type   | Mandatory | Description
---      | ---    | ---       | ---
`specversion`     | String | Mandatory | The currently supported CloudEvents specification version.
`id`     | String | Mandatory | A unique value (such as a UUID) for this event. It can be used to implement deduplication/idempotency handling in downstream systems. It is used as the `lastEventId` in subsequent calls.
`type`   | String | Mandatory | The type of the event. May be used to specify and deserialize the payload. A feed may contain different event types. It SHOULD be prefixed with a reverse-DNS name.
`source` | String | Mandatory | The source system that created the event. Should be a URI of the system.
`time` | String | Mandatory | The event addition timestamp. ISO 8601 UTC date and time format.
`subject` | String | Optional | Key to identify the business object. It doesn't have to be unique within the feed. This should represent a business key such as an order number or sku. Used for `compaction` and `deletion`, if implemented.
`method` | String | Optional | The HTTP equivalent method type that the feed item performs on the `subject`. `PUT` indicates that the _subject_ was created or updated. `DELETE` indicates that the  _subject_ was deleted. Defaults to `PUT`.
`data`   | Object | Optional  | The payload of the item in JSON. May be missing, e.g. when the method was `DELETE`.

Further metadata may be added, e.g. for traceability.
