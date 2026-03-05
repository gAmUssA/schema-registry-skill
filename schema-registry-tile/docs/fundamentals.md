# Schema Registry Fundamentals

## Table of Contents
1. [Schemas, Subjects, and Topics](#schemas-subjects-and-topics)
2. [Subject Naming Strategies](#subject-naming-strategies)
3. [Schema IDs](#schema-ids)
4. [Compatibility Types](#compatibility-types)
5. [Schema Evolution Rules by Format](#schema-evolution-rules-by-format)
6. [Schema Contexts](#schema-contexts)
7. [Schema References](#schema-references)
8. [Backend Storage](#backend-storage)

## Schemas, Subjects, and Topics

These are distinct concepts:

- **Topic**: A Kafka topic that contains messages as key-value pairs
- **Schema**: Defines the structure of data (Avro record, Protobuf message, JSON Schema)
- **Subject**: The named scope under which schemas evolve — derived from topic name by default but configurable

The topic name and schema name are independent. A single topic can have multiple schemas (key and value), and a single schema can be used across multiple topics.

## Subject Naming Strategies

The subject name strategy determines how schemas are organized in the registry.

### TopicNameStrategy (default)
```
Subject = <topic>-key   (for keys)
Subject = <topic>-value (for values)
```
- One schema per topic per position (key/value)
- Simplest model: topic acts as the schema boundary
- Use when each topic has exactly one event type

### RecordNameStrategy
```
Subject = <fully.qualified.RecordName>
```
- Schema is identified by its record name, independent of topic
- Enables multiple event types on a single topic
- Different topics can share the same schema subject
- Use when you have a shared event schema across topics

### TopicRecordNameStrategy
```
Subject = <topic>-<fully.qualified.RecordName>
```
- Combines topic and record name for topic-scoped multi-schema subjects
- Use when multiple event types exist per topic but schemas should be topic-scoped

### Configuration
```java
// Producer/Consumer config
props.put("value.subject.name.strategy",
    "io.confluent.kafka.serializers.subject.TopicNameStrategy");
// or RecordNameStrategy, TopicRecordNameStrategy

props.put("key.subject.name.strategy",
    "io.confluent.kafka.serializers.subject.TopicNameStrategy");
```

## Schema IDs

- Globally unique integer assigned to each registered schema
- Monotonically increasing but not necessarily consecutive
- Unique within a (tenant, context) combination
- Embedded in the serialized message payload (5 bytes: magic byte + 4-byte ID)
- Used by deserializers to fetch the writer schema from the registry

Wire format:
```
[0x00][4-byte schema ID][serialized data]
```

## Compatibility Types

### BACKWARD (default)
- New schema can read data written with the previous schema version
- **Allowed**: Delete fields, add fields with default values
- **Not allowed**: Add required fields without defaults, remove required fields
- **Validates against**: Last registered version only
- **Upgrade order**: Consumers first, then producers

### BACKWARD_TRANSITIVE
- Same rules as BACKWARD but validated against ALL previously registered versions
- Prevents schemas that are backward-compatible with the latest but not with older versions
- Use when consumers might lag far behind on schema versions

### FORWARD
- Data written with the new schema can be read by consumers using the previous schema
- **Allowed**: Add new fields (consumers ignore unknown fields), delete optional fields with defaults
- **Not allowed**: Remove fields that consumers still expect, modify field types
- **Validates against**: Last registered version only
- **Upgrade order**: Producers first, then consumers

### FORWARD_TRANSITIVE
- Same rules as FORWARD but validated against ALL previously registered versions
- Use when producers might write data that very old consumers still need to read

### FULL
- Both backward AND forward compatible
- **Allowed**: Add or remove fields only if they have default values
- **Not allowed**: Any change that breaks either direction
- **Validates against**: Last registered version only
- **Upgrade order**: Any order

### FULL_TRANSITIVE
- Both backward and forward compatible with ALL previously registered versions
- Most restrictive mode — safest for long-lived systems
- Use for critical data pipelines where any breakage is unacceptable

### NONE
- No compatibility checking at all
- Any schema can be registered regardless of previous versions
- Use only for development/testing or when intentional breaking changes are needed
- Example: changing a field type from Number to String

## Schema Evolution Rules by Format

### Avro Evolution Rules

**Adding a field** (backward compatible):
```json
// V1
{"type": "record", "name": "User", "fields": [
  {"name": "id", "type": "int"},
  {"name": "name", "type": "string"}
]}

// V2 - added email with default
{"type": "record", "name": "User", "fields": [
  {"name": "id", "type": "int"},
  {"name": "name", "type": "string"},
  {"name": "email", "type": ["null", "string"], "default": null}
]}
```

**Key Avro rules:**
- Optional fields use union types: `["null", "string"]`
- Default value must match the first type in the union
- Adding fields requires a default for backward compatibility
- Removing fields requires the removed field to have had a default for forward compatibility
- Field renaming is done via aliases
- Type promotions allowed: int → long, float → double

### Protobuf Evolution Rules

**Adding a field** (backward compatible):
```protobuf
// V1
message User {
  int32 id = 1;
  string name = 2;
}

// V2
message User {
  int32 id = 1;
  string name = 2;
  string email = 3;  // new field, consumers ignore unknown fields
}
```

**Key Protobuf rules:**
- Never reuse field numbers — they are the wire identity
- Adding fields is backward compatible (old consumers ignore unknown fields)
- Removing fields is backward compatible if done correctly (don't reuse the number)
- Adding new message types is NOT forward compatible (use BACKWARD_TRANSITIVE)
- Scalar type widening allowed: int32 → int64
- `reserved` keyword prevents accidental reuse of deleted field numbers
- `oneof` fields have special compatibility considerations

### JSON Schema Evolution Rules

**Adding a field** (backward compatible with open content model):
```json
// V1
{
  "type": "object",
  "properties": {
    "id": {"type": "integer"},
    "name": {"type": "string"}
  },
  "required": ["id", "name"]
}

// V2
{
  "type": "object",
  "properties": {
    "id": {"type": "integer"},
    "name": {"type": "string"},
    "email": {"type": "string"}
  },
  "required": ["id", "name"]
}
```

**Key JSON Schema rules:**
- Open content model (`additionalProperties: true`, the default) is more evolution-friendly
- Closed content model (`additionalProperties: false`) restricts adding properties
- Adding a new `required` field is NOT backward compatible
- Removing a `required` constraint is backward compatible
- Lenient vs strict compatibility policies affect what's allowed
- `additionalProperties` setting dramatically impacts compatibility

## Schema Contexts

Schema contexts provide logical grouping — separate "sub-registries" within one Schema Registry instance.

- Context is a prefix on the subject name: `:.context-name:subject-name`
- The default context is the empty context (no prefix)
- Useful for multi-tenant environments or separating dev/staging/prod schemas
- Each context has independent schema ID sequences

## Schema References

All three formats support schema references for composing complex schemas:

**Avro**: References model imported schemas
**Protobuf**: References model `import` statements
**JSON Schema**: References model `$ref` pointers

```json
// Registering a schema with references
{
  "schema": "...",
  "schemaType": "AVRO",
  "references": [
    {
      "name": "com.example.Address",
      "subject": "address-value",
      "version": 1
    }
  ]
}
```

## Backend Storage

- Schemas stored in a compacted Kafka topic (`_schemas` by default)
- Acts as a write-ahead log — all schemas, metadata, and compatibility settings are appended
- Kafka's ordering guarantees ensure consistency
- Schema Registry instances can be restarted and rebuild state from this topic
- Supports multi-datacenter deployments via schema linking
