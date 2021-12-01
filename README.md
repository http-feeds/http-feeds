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

```
GET https://example.http-feeds.org/feeds/customers
Accept: application/json

200 OK
Content-Type: application/cloudevents-batch+json
[{
  "specversion" : "1.0",
  "type" : "org.http-feeds.customer",
  "source" : "/feeds/customers",
  "id" : "1c6b8c6e-d8d0-4a91-b51c-1f56bd04c758",
  "time" : "2021-12-01T00:01:01Z",
  "subject" : "4f16ee9d-eee8-4f0d-9772-7ab7f0efbd5a",
  "data" : {
    "customerId": "4f16ee9d-eee8-4f0d-9772-7ab7f0efbd5a",
    "created": "2021-12-01T00:01:00Z",
    "updated": "2021-12-01T00:01:00Z",
    "name": "Alice",
    "email": "alice@example.org",
    "newsletter": false,
    "address": {}
  }
},{
    "specversion" : "1.0",
    "type" : "org.http-feeds.customer",
    "source" : "/feeds/customers",
    "id" : "6860e2a6-19a2-4903-badd-ca470a94ac82",
    "time" : "2021-12-01T00:02:02Z",
    "subject" : "97d825b1-b911-461d-9c95-4b20c8c51201",
    "data" : {
      "customerId": "97d825b1-b911-461d-9c95-4b20c8c51201",
      "created": "2021-12-01T00:02:00Z",
      "updated": "2021-12-01T00:02:00Z",
      "name": "Bob",
      "email": "bob@example.org",
      "newsletter": true,
      "address": {}
    }
  }
]
```

Polling
---

Event Format
---

Feed Types
---

### Event Feed

### Aggregate Feed




