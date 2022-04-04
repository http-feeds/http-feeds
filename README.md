HTTP Feeds
===

Asynchronous [event streaming](#event-feeds) and [data replication](#aggregate-feeds) with plain HTTP APIs.

HTTP feeds is a minimal specification for polling events over HTTP:

- An HTTP feed provides a HTTP GET endpoint
- that returns a chronological sequence (!) of events
- serialized in [CloudEvents](https://github.com/cloudevents/spec) event format
- in batched responses using the media type `application/cloudevents-batch+json`
- and respects the `lastEventId` query parameter to scroll through further items
- to support [infinite polling](#polling) for real-time feed subscriptions.

HTTP feeds can be used to [decouple systems](https://scs-architecture.org) asynchronously [without message brokers](https://www.innoq.com/en/articles/2021/12/http-feeds-schnittstellen-ohne-kafka-oder-rabbitmq), such as Kafka or RabbitMQ.

Example
---

```http
GET /inventory HTTP/1.1
Host: https://example.http-feeds.org
```

```http
HTTP/1.1 200 OK
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
},{
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
```

```http
HTTP/1.1 200 OK
Content-Type: application/cloudevents-batch+json

[]
```

An empty array signals the end of the feed.

## Polling

A client can continue polling in an infinite loop to subscribe to a feed.

### Simple Polling

A client calls the endpoint with the last known event id as `lastEventId` query parameter in an endless loop.
If the response is an empty array, the client reached the end of the stream and waits some time to make another call to get events that occured in the meantime.

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

Client _must_ persist the `id` of the last processed event as `lastEventId` for further fetches.

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
    // Delay the next request only in case of a server error
    wait N seconds
```

If there are no newer events available, the server keeps the connection open until new events arrive or the defined period timed out.
The server then sends the response (with the new events or an empty array) and the client can immediately perform another call.

The latency can be improved, as the server can react to new events efficiently by implementing an internal event notification, change data capture triggers, or performing a high-frequency polling to the database.

The cost of long polling is that the server needs to handle more open connections concurrently.
This may become an issue with more than [10K connections](http://www.kegel.com/c10k.html).

## Event ID

The `event.id` is used as `lastEventId` to scroll through further events.
This means that events need to be strongly ordered to retrieve subsequent events.

Events may also get deleted from the feed, e.g. through [Compaction](#compaction) and [Deletion](#deletion).
The server must still respect the original position and send only newer events, even when an event with the `lastEventId` was deleted.

One way to achieve this is to use an time-ordered [UUIDv6](https://uuid.ramsey.dev/en/stable/nonstandard/version6.html) as event ID (see [IETF draft](https://datatracker.ietf.org/doc/html/draft-peabody-dispatch-new-uuid-format#section-4.3) and [Java-Library](https://github.com/f4b6a3/uuid-creator/wiki/1.6.-Time-ordered)). 
This a viable option, especially if only one server appends new events to the feed, but might be a problem with multiple servers when the clocks are not in-sync.

An alternative is to use a database [sequence](https://www.postgresql.org/docs/current/functions-sequence.html) that is used as part of the `event.id` and interpreted when querying the database for the next batch. Example: The event.id `0000001000001::5f8de8ff-30d8-4fab-8f5a-c32f326d6f26` contains a database sequence `0000001000001` and a random UUID.


## Event Feeds

HTTP feeds can be used to provide an API for publishing immutable domain events to other systems.

It is quite common that one http feed endpoint includes different event `type`s that belong to the same bounded context.

## Aggregate Feeds

HTTP feeds can be used to provide an API for data collections of mutable objects (aka aggregates, master data) to other systems.

An aggregate is identified through its `subject`. 
An aggregate feed _must_ contain every aggregate at least once.
Every created aggregate and each update leads to an appended feed entry with the full current state.

Feed consumers can subscribe an _aggregate feed_ to perform near real-time data synchronization to build local read models and to trigger actions when new or updated data is received.
A feed consumer has a consistent state when reaching the end of the feed.

The server should implement [Compaction](#compaction) and may implement [Deletion](#deletion) based on business requirements.

### Compaction

Each aggregate update leads to an additional entry in the feed.
It is good practice to keep the feed small to enable a quick synchronization of new clients.
When feed items include the full current state of the resource, older feed items for the same aggregate may be outdated.

Clients that read the feed from the beginning will therefore occur an inconsistent state for a short time.
To mitigate, entries _may_ be deleted from the feed when another entry was appended to the feed with the same `subject`.

Example:
There is an update for subject `9521234567899`.

```http
HTTP/1.1 200 OK
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
},{
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
HTTP/1.1 200 OK
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
},{
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

Aggregate feeds use a `method` field with value `DELETE` to signal the deletion of an `subject` to consumers that built a local read model before.

When aggregate was deleted, the server must append a `DELETE` entry with the `subject` to delete and no data content.

```json
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

Clients _must_ delete this aggregate or otherwise handle the removal.

The server _should_ start a [compaction run](#compaction) afterwards to delete previous entries for the same aggregate.

## Data Model

An HTTP feed endpoint must support these query parameters:

Query Parameter | Type        | Mandatory | Description
---             | ---         | ---       | ---        
`lastEventId`   | String      | Optional  | The last event `id` that was processed. Used to scroll through further events. May be missing or `null` to start from the beginning of the feed.
`timeout`       | Number      | Optional  | A timeout is set, when long-polling should be used and is supported by the server. Max waiting time in milliseconds for long-polling, after which the server must send a response. A typical value is `5000`.


The response body contains an array of events that comply with the [CloudEvents Specification](https://github.com/cloudevents/spec) in the [`application/cloudevents-batch+json` format](https://github.com/cloudevents/spec/blob/v1.0.1/json-format.md#4-json-batch-format).

Field    | Type            | Event Feed | Aggregate Feed | Description
---      | ---             | ---        | ---        | ---
`specversion`     | String | Mandatory  | Mandatory | The currently supported CloudEvents specification version.
`id`     | String | Mandatory | Mandatory | A unique value (such as a UUID) for this event. It can be used to implement deduplication/idempotency handling in downstream systems. It is used as the `lastEventId` in subsequent calls.
`type`   | String | Mandatory | Mandatory | The type of the event. May be used to specify and deserialize the payload. A feed may contain different event types. It SHOULD be prefixed with a reverse-DNS name.
`source` | String | Mandatory | Mandatory | The source system that created the event. Should be a URI of the system.
`time` | String | Mandatory | Mandatory | The event addition timestamp. ISO 8601 UTC date and time format.
`subject` | String | n/a    | Mandatory | Key to identify the business object. It doesn't have to be unique within the feed. This should represent a business key such as an order number or sku. Used for [compaction](#compaction) and [deletion](#deletion), if implemented.
`method` | String | n/a | Optional | The HTTP equivalent method type that the feed item performs on the `subject`. `PUT` indicates that the _subject_ was created or updated. `DELETE` indicates that the  _subject_ was deleted. Defaults to `PUT`.
`datacontenttype` | String | Optional | Optional | Defaults to `application/json`.
`data`   | Object | Mandatory | Optional |  The payload of the item. Defaults to JSON. May be missing, e.g. when the method was `DELETE`.

Further metadata may be added, e.g. for traceability.


## Authentication

HTTP feeds _may_ be protected with [HTTP authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication). 

The most common authentication schemes are [Basic](https://tools.ietf.org/html/rfc7617) and [Bearer](https://tools.ietf.org/html/rfc6750).

The server _may_ filter feed items based on the principal.
When filtering is applied, [caching](#caching) may be unfeasible.

## Caching

Servers _may_ set [appropriate](https://devcenter.heroku.com/articles/increasing-application-performance-with-http-cache-headers) response headers, such as `Cache-Control: public, max-age=31536000`, when a batch is full and will not be modified anymore.

## Libraries and Examples

* [http-feeds-server-spring-boot-starter](https://github.com/http-feeds/http-feeds-server-spring-boot-starter)
* [http-feeds-server-spring-boot-example](https://github.com/http-feeds/http-feeds-server-spring-boot-example)
* [http-feeds-server-aws-serverless-example](https://github.com/http-feeds/http-feeds-server-aws-serverless-example)

## About

This site is maintained by [Jochen Christ](https://twitter.com/jochen_christ).  
HTTP feeds has evolved from [rest-feeds](https://github.com/rest-feeds/rest-feeds).  
Contributions are highly appreciated, e.g. libraries or examples in different languages or frameworks help a lot.

Found an error or something is missing? 
Please [raise an issue](https://github.com/http-feeds/http-feeds/issues/new) or [create a pull request](https://github.com/http-feeds/http-feeds/pulls).

This specification is published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).


<!-- 100% privacy friendly analytics -->
<script async defer src="https://scripts.simpleanalyticscdn.com/latest.js"></script>
<noscript><img src="https://queue.simpleanalyticscdn.com/noscript.gif" alt="" referrerpolicy="no-referrer-when-downgrade" /></noscript>


<a href="https://github.com/http-feeds/http-feeds" class="github-corner" aria-label="View source on GitHub"><svg width="80" height="80" viewBox="0 0 250 250" style="fill:#151513; color:#fff; position: absolute; top: 0; border: 0; right: 0;" aria-hidden="true"><path d="M0,0 L115,115 L130,115 L142,142 L250,250 L250,0 Z"></path><path d="M128.3,109.0 C113.8,99.7 119.0,89.6 119.0,89.6 C122.0,82.7 120.5,78.6 120.5,78.6 C119.2,72.0 123.4,76.3 123.4,76.3 C127.3,80.9 125.5,87.3 125.5,87.3 C122.9,97.6 130.6,101.9 134.4,103.2" fill="currentColor" style="transform-origin: 130px 106px;" class="octo-arm"></path><path d="M115.0,115.0 C114.9,115.1 118.7,116.5 119.8,115.4 L133.7,101.6 C136.9,99.2 139.9,98.4 142.2,98.6 C133.8,88.0 127.5,74.4 143.8,58.0 C148.5,53.4 154.0,51.2 159.7,51.0 C160.3,49.4 163.2,43.6 171.4,40.1 C171.4,40.1 176.1,42.5 178.8,56.2 C183.1,58.6 187.2,61.8 190.9,65.4 C194.5,69.0 197.7,73.2 200.1,77.6 C213.8,80.2 216.3,84.9 216.3,84.9 C212.7,93.1 206.9,96.0 205.4,96.6 C205.1,102.4 203.0,107.8 198.3,112.5 C181.9,128.9 168.3,122.5 157.7,114.1 C157.9,116.9 156.7,120.9 152.7,124.9 L141.0,136.5 C139.8,137.7 141.6,141.9 141.8,141.8 Z" fill="currentColor" class="octo-body"></path></svg></a><style>.github-corner:hover .octo-arm{animation:octocat-wave 560ms ease-in-out}@keyframes octocat-wave{0%,100%{transform:rotate(0)}20%,60%{transform:rotate(-25deg)}40%,80%{transform:rotate(10deg)}}@media (max-width:500px){.github-corner:hover .octo-arm{animation:none}.github-corner .octo-arm{animation:octocat-wave 560ms ease-in-out}}</style>
