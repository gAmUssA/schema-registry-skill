---
name: schema-registry
description: Use when working with Schema Registry for Apache Kafka, Confluent Platform, or Confluent Cloud. Covers schema management (Avro, Protobuf, JSON Schema), compatibility modes, schema evolution, REST API, serializer/deserializer configuration, Kafka Connect converters, and Flink SQL integration with Schema Registry. Trigger this skill whenever the user mentions schema registry, schema evolution, Avro/Protobuf/JSON Schema serialization with Kafka, subject naming strategies, compatibility checking, or Flink SQL with Confluent formats (avro-confluent). Also trigger when users ask about data contracts, schema validation, or serializer/deserializer configuration for Kafka producers and consumers.
---

# Schema Registry Skill

## When to Read Reference Docs

This skill uses progressive disclosure. Read the appropriate doc file based on the user's question:

| User's Question | Doc File |
|----------------|----------|
| Schema concepts, compatibility modes, evolution rules, subject naming | `docs/fundamentals.md` |
| REST API endpoints, curl commands, schema operations | `docs/rest-api.md` |
| Kafka producer/consumer serializers, Kafka Connect converters | `docs/serdes.md` |
| Flink SQL with Schema Registry, avro-confluent format, CREATE TABLE | `docs/flink-sql.md` |
| Confluent Cloud Schema Registry, managed service specifics | `docs/confluent-cloud.md` |

If the question spans multiple areas, read multiple doc files.

## Core Quick Reference

### Subject Naming Strategies

| Strategy | Subject Name | Use Case |
|----------|-------------|----------|
| `TopicNameStrategy` (default) | `<topic>-key`, `<topic>-value` | One schema per topic |
| `RecordNameStrategy` | `<fully.qualified.RecordName>` | Multiple event types per topic |
| `TopicRecordNameStrategy` | `<topic>-<fully.qualified.RecordName>` | Multiple event types, topic-scoped |

### Compatibility Types

| Type | Allowed Changes | Upgrade Order |
|------|----------------|---------------|
| `BACKWARD` (default) | Delete fields, add optional fields with defaults | Consumers first |
| `FORWARD` | Add fields, delete optional fields with defaults | Producers first |
| `FULL` | Add/remove only optional fields with defaults | Any order |
| `NONE` | Any change | N/A |

Add `_TRANSITIVE` suffix to validate against ALL versions instead of just the last.

## Schema Evolution Workflow

Schema evolution is potentially breaking -- always follow this sequence:

**Step 1 -- Test compatibility before registering:**
```bash
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "<new-schema-json>"}' \
  http://localhost:8081/compatibility/subjects/my-topic-value/versions/latest
```
- `{"is_compatible":true}` -> proceed to Step 2
- `{"is_compatible":false}` -> adjust schema (add defaults, use union type), then re-test

**Step 2 -- Register the new schema version:**
```bash
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "<new-schema-json>"}' \
  http://localhost:8081/subjects/my-topic-value/versions
```

**Step 3 -- Verify registration:**
```bash
curl http://localhost:8081/subjects/my-topic-value/versions/latest
```

**Step 4 -- Deploy in correct order:**
- BACKWARD: consumers first, then producers
- FORWARD: producers first, then consumers
- FULL: any order

## Common Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| `Schema being registered is incompatible` | Violates compatibility rules | Add defaults to new fields, evolve in steps |
| `Subject not found` | Not registered yet | Register schema first, check naming strategy |
| `Serialization exception` | Schema mismatch | Ensure producer schema is compatible with registered schema |
| `401/403 from Schema Registry` | Auth issue | Check credentials, API keys, RBAC permissions |
