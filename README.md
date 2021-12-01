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


Polling
---

Event Format
---

Feed Types
---

### Event Feed

### Aggregate Feed




