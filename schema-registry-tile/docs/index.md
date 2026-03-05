# Schema Registry

Schema Registry provides a centralized repository for managing and validating schemas (Avro, Protobuf, JSON Schema) used with Apache Kafka. It enforces compatibility rules during schema evolution and integrates with Kafka producers, consumers, Kafka Connect, and Flink SQL.

## Key Concepts

- **Schema**: Structure definition (Avro record, Protobuf message, JSON Schema object)
- **Subject**: Named scope under which schemas evolve (e.g., `my-topic-value`)
- **Schema ID**: Globally unique integer, embedded in serialized messages (`[0x00][4-byte ID][data]`)
- **Compatibility**: Rules governing allowed schema changes between versions

## Contents

- [Fundamentals](./fundamentals.md) - Schemas, subjects, naming strategies, compatibility types, evolution rules by format
- [REST API](./rest-api.md) - All Schema Registry REST endpoints with curl examples
- [Serializers and Deserializers](./serdes.md) - Kafka producer/consumer serializers, Kafka Connect converters, Maven dependencies
- [Flink SQL Integration](./flink-sql.md) - avro-confluent format, CREATE TABLE patterns, Confluent Cloud Flink SQL
- [Confluent Cloud](./confluent-cloud.md) - Managed Schema Registry, data contracts, schema linking, stream governance
