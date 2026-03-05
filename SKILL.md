---
name: schema-registry
description: Use when working with Schema Registry for Apache Kafka, Confluent Platform, or Confluent Cloud. Covers schema management (Avro, Protobuf, JSON Schema), compatibility modes, schema evolution, REST API, serializer/deserializer configuration, Kafka Connect converters, and Flink SQL integration with Schema Registry. Trigger this skill whenever the user mentions schema registry, schema evolution, Avro/Protobuf/JSON Schema serialization with Kafka, subject naming strategies, compatibility checking, or Flink SQL with Confluent formats (avro-confluent). Also trigger when users ask about data contracts, schema validation, or serializer/deserializer configuration for Kafka producers and consumers.
---

# Schema Registry Skill

## When to Read Reference Files

This skill uses progressive disclosure. Read the appropriate reference file based on the user's question:

| User's Question | Reference File |
|----------------|----------------|
| Schema concepts, compatibility modes, evolution rules, subject naming | `references/fundamentals.md` |
| REST API endpoints, curl commands, schema operations | `references/rest-api.md` |
| Kafka producer/consumer serializers, Kafka Connect converters | `references/serdes.md` |
| Flink SQL with Schema Registry, avro-confluent format, CREATE TABLE | `references/flink-sql.md` |
| Confluent Cloud Schema Registry, managed service specifics | `references/confluent-cloud.md` |

If the question spans multiple areas, read multiple reference files.

## Core Concepts (Quick Reference)

### Key Terminology

- **Schema**: The structure definition (Avro record, Protobuf message, JSON Schema object)
- **Subject**: The scope under which schemas evolve (e.g., `my-topic-value`, `my-topic-key`)
- **Schema ID**: Globally unique integer assigned on registration
- **Compatibility**: Rules governing how a schema can change between versions

### Subject Naming Strategies

| Strategy | Subject Name | Use Case |
|----------|-------------|----------|
| `TopicNameStrategy` (default) | `<topic>-key`, `<topic>-value` | One schema per topic |
| `RecordNameStrategy` | `<fully.qualified.RecordName>` | Multiple event types per topic |
| `TopicRecordNameStrategy` | `<topic>-<fully.qualified.RecordName>` | Multiple event types, topic-scoped |

### Compatibility Types

| Type | Checks Against | Allowed Changes |
|------|---------------|-----------------|
| `BACKWARD` (default) | Last version | Delete fields, add optional fields with defaults |
| `BACKWARD_TRANSITIVE` | All versions | Same as BACKWARD, validated across all history |
| `FORWARD` | Last version | Add fields, delete optional fields with defaults |
| `FORWARD_TRANSITIVE` | All versions | Same as FORWARD, validated across all history |
| `FULL` | Last version | Add/remove only optional fields with defaults |
| `FULL_TRANSITIVE` | All versions | Same as FULL, validated across all history |
| `NONE` | Nothing | Any change allowed (no validation) |

**Upgrade order matters:**
- BACKWARD: upgrade consumers first, then producers
- FORWARD: upgrade producers first, then consumers
- FULL: any order

### Common REST API Patterns

```bash
# Register a schema
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"int\"},{\"name\":\"name\",\"type\":\"string\"}]}"}' \
  http://localhost:8081/subjects/my-topic-value/versions

# Get schema by ID
curl http://localhost:8081/schemas/ids/1

# List subjects
curl http://localhost:8081/subjects

# Get latest schema for a subject
curl http://localhost:8081/subjects/my-topic-value/versions/latest

# Test compatibility
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "..."}' \
  http://localhost:8081/compatibility/subjects/my-topic-value/versions/latest

# Get/set compatibility level
curl http://localhost:8081/config
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "FULL"}' \
  http://localhost:8081/config/my-topic-value
```

### Quick Serializer Setup (Java)

```java
// Producer with Avro
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put("schema.registry.url", "http://localhost:8081");

// Consumer with Avro
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "io.confluent.kafka.serializers.KafkaAvroDeserializer");
props.put("schema.registry.url", "http://localhost:8081");
props.put("specific.avro.reader", "true");
```

### Quick Flink SQL Setup

```sql
CREATE TABLE users (
  id BIGINT,
  name STRING,
  email STRING
) WITH (
  'connector' = 'kafka',
  'topic' = 'users',
  'properties.bootstrap.servers' = 'localhost:9092',
  'value.format' = 'avro-confluent',
  'value.avro-confluent.url' = 'http://localhost:8081'
);
```

## Schema Evolution Workflow

Schema evolution is potentially breaking — always follow this sequence:

**Step 1 — Test compatibility before registering:**
```bash
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "<new-schema-json>"}' \
  http://localhost:8081/compatibility/subjects/my-topic-value/versions/latest
```
- Response `{"is_compatible":true}` → proceed to Step 2.
- Response `{"is_compatible":false}` or HTTP 409 → review the error message, adjust the schema (e.g., add missing defaults, use a union type, or remove the breaking field change), then re-test before continuing.

**Step 2 — Register the new schema version:**
```bash
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "<new-schema-json>"}' \
  http://localhost:8081/subjects/my-topic-value/versions
```
Note the returned `id` for verification.

**Step 3 — Verify registration:**
```bash
# Confirm the new version is present and the ID matches
curl http://localhost:8081/subjects/my-topic-value/versions/latest
```

**Step 4 — Deploy consumers/producers in the correct order:**
- **BACKWARD**: deploy consumers first, then producers.
- **FORWARD**: deploy producers first, then consumers.
- **FULL**: any order.

## Deployment Options

- **Open-source**: Apache Kafka + Confluent Schema Registry (community edition)
- **Confluent Platform**: Self-managed with enterprise features (RBAC, schema linking, data contracts)
- **Confluent Cloud**: Fully managed Schema Registry (Essentials and Advanced tiers)

## Common Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| `Schema being registered is incompatible` | New schema violates compatibility rules | Check compatibility level, add defaults to new fields, or evolve in steps |
| `Subject not found` | Subject hasn't been registered yet | Register schema first, or check subject naming strategy |
| `Serialization exception` | Schema mismatch between producer and registry | Ensure producer schema matches or is compatible with registered schema |
| `401/403 from Schema Registry` | Authentication/authorization issue | Check credentials, API keys, or RBAC permissions |
