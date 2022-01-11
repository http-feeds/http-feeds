HTTP Feeds
===

Asynchronously decouple systems with plain JSON/HTTP APIs.

HTTP feeds is a minimal specification for polling events over HTTP:

- An HTTP feed provides a HTTP GET endpoint
- Returns a chronological sequence (!) of events
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

The client can continue polling in an infinite loop to subscribe to the feed.

### Simple Polling

The client calls the endpoint with the last known `id` in an loop.
If the response is an empty array, the client reached the end of the stream and waits some time to make another call to get events that happened in the meantime.

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

The client _must_ persist the `id` of the last processed event as `lastEventId` for further fetches.

The client's event processing _must_ be idempotent (_at-least-once_ delivery semantic). 
The `id` _may_ be used for idempotency checks.

### Long Polling

The server may also support _long polling_ for lower latency.
The client adds a `timeout` query parameter to specify the max period of milliseconds to wait for an response.

Client Pseudocode:

```python
endpoint = "https://example.http-feeds.org/inventory"
lastEventId = null
timeout = 5000 // 5000 milliseconds is a good timeout period for long polling

while true:
  try:
    response = GET endpoint + "?lastEventId=" + lastEventId + "&timeout=" + timeout
    for event in response:
      process event
      lastEventId = event.id
    // no client wait step within the loop
  except:
    // protect the server from an in case of an server error
    wait N seconds
```

If there are no newer events available, the server keeps the connection open until new events arrive or the defined period timed out.
The server then sends the response (with the new events or an empty array) and the client can immediatelly perform another call.
The server can recognize new events more efficiently by implementing an event notification and/or performing a high-frequency polling to the database.

The cost of long polling is that the server needs to handle more connections concurrently.
This may become an issue with more than [10K connections](http://www.kegel.com/c10k.html).


## Aggregate Feeds

HTTP feeds can be used to provide an API for data collections of mutable objects (aka aggregates, master data) with other systems. 

Feed consumers can subscribe an _aggregate feed_ to perform near real-time data synchronization to build local read models and to trigger actions when new or updated data is received.

Every aggregate update leads to an appended feed entry with the full current state or an URL to fetch the current state.
An _aggregate feed_ must contain every aggregate (identified through its `subject`) at least once.
A feed consumer has an eventual consistent state when reaching the end of the feed.

The server should implement [Compaction](#Compaction) an may implement [Deletion](#Deletion) based on business requirements.

### Compaction

Each aggregate update leads to an additional entry in the feed.
It is good practice to keep the feed small to enable a quick synchronization of new clients.
When feed items include the full current state of the resource, older feed items for the same aggregate may be outdated.

Consumers that read the feed from the beginning will therefore have an inconsistent state for a short time.
To mitigate, entries may be deleted from the feed when another entry was appended to the feed with the same `subject`.

Example:
There is an update for subject `9521234567899`.

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

After a compaction run, the first entry is gone:

```http
200 OK
Content-Type: application/cloudevents-batch+json

[{
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


### Deletion

Some aggregates need to  be deleted, e. g. by regulatory requirements.

Aggregate feeds use a `method` field to signal the deletion of an `subject` to consumers that built a local read model before.

When aggregate was deleted, the server must append a `DELETE` entry with the `subject` to delete and no data content.

```
{
  "specversion" : "1.0",
  "type" : "org.http-feeds.example.inventory",
  "source" : "https://example.http-feeds.org/inventory",
  "id" : "06b13630-e4c3-4d85-a669-ce66fc4daa75",
  "time" : "2021-12-31T00:00:01Z",
  "subject" : "9521234567899",
  "method": "DELETE"
}
```

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
