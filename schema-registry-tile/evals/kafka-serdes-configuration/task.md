# Kafka Producer and Consumer with Schema Registry for an Event-Driven Service

## Problem/Feature Description

The backend team at LogiTrack is building an event-driven parcel-tracking service. Parcel scan events are produced to a Kafka topic called `parcel-scans`, serialized using Avro, and consumed by multiple downstream microservices. The team decided to use code-generated Avro classes (from `.avsc` schema files) rather than generic records so that each service has compile-time type safety. A senior engineer has asked a team member to produce a reference Java project structure — with `pom.xml`, producer code, and consumer code — so that new services can onboard quickly and consistently.

The parcel-scans topic sends events where both the key (parcel ID string) and value (scan event) are Avro-serialized through Schema Registry. Multiple event types are expected in the future on the same topic, and the team wants a schema organization strategy that will scale gracefully as new event types are added without tight coupling to topic names. For now, they want a reference implementation that can be reviewed and adapted. The producer and consumer code does not need to run against a real cluster — it should demonstrate correct configuration and usage patterns.

## Output Specification

Produce the following files:

- `pom.xml` — A Maven POM showing correct dependencies and repository configuration for Avro Schema Registry serialization. Include the Avro Maven plugin configuration for code generation from `.avsc` files.
- `ParcelScanEvent.avsc` — An Avro schema file defining a `ParcelScanEvent` record in the `com.logitrack.events` namespace, with fields: `parcel_id` (string), `location` (string), `timestamp` (long), and an optional `notes` field (nullable string).
- `ProducerConfig.java` — A Java class (no main method required) that builds a `Properties` object for a Kafka producer that uses Avro serialization with Schema Registry. Choose a subject naming strategy appropriate for the multi-event-type scenario. Include all required and relevant configuration properties as comments or code.
- `ConsumerConfig.java` — A Java class (no main method required) that builds a `Properties` object for a Kafka consumer. The consumer should be configured to deserialize into the generated `ParcelScanEvent` type-safe class rather than a generic untyped representation. Include all required configuration properties.
- `design_notes.md` — A brief explanation of the subject naming strategy chosen and why it suits the multi-event-type scenario described above.
