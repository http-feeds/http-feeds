HTTP Feeds
===

Asynchronously decouple systems over plain HTTP without shared infrastructure.

HTTP feeds is a specification for polling events over HTTP:

- An HTTP Feed provides a HTTP GET endpoint
- Returns a chronological sequence of events or aggregates
- Serialized in [CloudEvents](https://github.com/cloudevents/spec) event format with default to `application/cloudevents-batch+json`
- As batched results 
- Respects the `lastEventId` query parameter to scroll through further items
- To support polling for real-time feed subscriptions



