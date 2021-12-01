HTTP Feeds
===

Asynchronously decouple systems over plain HTTP without shared infrastructure.

HTTP Feeds is a specification for polling events over HTTP:

- A plain HTTP GET endpoint
- Returns a chronological sequence of events
- Serialized in [CloudEvents](https://github.com/cloudevents/spec) event format with default to `application/cloudevents-batch+json`
- As batched results with `lastEventId` query parameter to scroll through further items
- To support polling for real-time subscription


