# Flink SQL with Schema Registry

## Table of Contents
1. [Overview](#overview)
2. [Apache Flink (Open Source)](#apache-flink-open-source)
3. [Confluent Cloud Flink SQL](#confluent-cloud-flink-sql)
4. [Format Configuration Reference](#format-configuration-reference)
5. [Data Type Mapping](#data-type-mapping)
6. [Common Patterns](#common-patterns)

## Overview

Flink SQL integrates with Confluent Schema Registry to serialize and deserialize Kafka messages using registered schemas. There are two contexts:

1. **Apache Flink (open source)**: Uses the `avro-confluent` format with the Kafka connector
2. **Confluent Cloud for Apache Flink**: Schema Registry integration is automatic — tables create topics and register schemas

## Apache Flink (Open Source)

### Dependencies

```xml
<repositories>
  <repository>
    <id>confluent</id>
    <url>https://packages.confluent.io/maven/</url>
  </repository>
</repositories>

<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-avro-confluent-registry</artifactId>
  <version>1.20.0</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-avro</artifactId>
  <version>1.20.0</version>
</dependency>
```

For SQL Client, place JARs in `$FLINK_HOME/lib/`.

### CREATE TABLE with avro-confluent Value Format

```sql
CREATE TABLE users (
  the_kafka_key STRING,
  id STRING,
  name STRING,
  email STRING
) WITH (
  'connector' = 'kafka',
  'topic' = 'users',
  'properties.bootstrap.servers' = 'localhost:9092',
  'properties.group.id' = 'flink-group',
  'scan.startup.mode' = 'earliest-offset',
  'key.format' = 'raw',
  'key.fields' = 'the_kafka_key',
  'value.format' = 'avro-confluent',
  'value.avro-confluent.url' = 'http://localhost:8081',
  'value.fields-include' = 'EXCEPT_KEY'
);
```

### CREATE TABLE with avro-confluent Key AND Value

```sql
CREATE TABLE users (
  kafka_key_id STRING,
  id STRING,
  name STRING,
  email STRING
) WITH (
  'connector' = 'kafka',
  'topic' = 'users',
  'properties.bootstrap.servers' = 'localhost:9092',
  'key.format' = 'avro-confluent',
  'key.avro-confluent.url' = 'http://localhost:8081',
  'key.fields' = 'kafka_key_id',
  'key.fields-prefix' = 'kafka_key_',
  'value.format' = 'avro-confluent',
  'value.avro-confluent.url' = 'http://localhost:8081',
  'value.fields-include' = 'EXCEPT_KEY',
  'key.avro-confluent.subject' = 'users-key',
  'value.avro-confluent.subject' = 'users-value'
);
```

### Upsert Kafka with avro-confluent

```sql
CREATE TABLE users_upsert (
  kafka_key_id STRING,
  id STRING,
  name STRING,
  email STRING,
  PRIMARY KEY (kafka_key_id) NOT ENFORCED
) WITH (
  'connector' = 'upsert-kafka',
  'topic' = 'users',
  'properties.bootstrap.servers' = 'localhost:9092',
  'key.format' = 'raw',
  'key.fields-prefix' = 'kafka_key_',
  'value.format' = 'avro-confluent',
  'value.avro-confluent.url' = 'http://localhost:8081',
  'value.fields-include' = 'EXCEPT_KEY'
);
```

### Writing Data

```sql
INSERT INTO users
SELECT
  id AS the_kafka_key,
  id,
  name,
  email
FROM source_table;
```

## Confluent Cloud Flink SQL

On Confluent Cloud, Schema Registry integration is built-in. CREATE TABLE automatically creates the backing Kafka topic and registers schemas.

### Basic CREATE TABLE

```sql
CREATE TABLE orders (
  order_id BIGINT,
  customer_id BIGINT,
  product STRING,
  amount DECIMAL(10, 2),
  order_time TIMESTAMP_LTZ(3)
);
```

No `WITH` clause needed for basic tables — Confluent Cloud infers the Kafka topic and Schema Registry configuration from the environment.

### With Changelog Mode

```sql
CREATE TABLE orders (
  order_id BIGINT,
  customer_id BIGINT,
  product STRING,
  amount DECIMAL(10, 2)
) WITH (
  'changelog.mode' = 'retract'
);
```

### Querying Existing Topics

If a topic and schema already exist in Schema Registry, Flink can auto-discover the schema:

```sql
-- Topic 'orders' with registered Avro schema is automatically mapped
SELECT * FROM orders;
```

### INSERT AS SELECT

```sql
-- Create enriched_orders topic with schema derived from the query
INSERT INTO enriched_orders
SELECT
  o.order_id,
  o.product,
  c.name AS customer_name,
  o.amount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
```

## Format Configuration Reference

### avro-confluent Format Options

| Option | Required | Default | Description |
|--------|----------|---------|-------------|
| `avro-confluent.url` | Yes | — | Schema Registry URL |
| `avro-confluent.subject` | No | `<topic>-value` or `<topic>-key` | Subject for schema registration |
| `avro-confluent.schema` | No | — | Custom Avro schema (must match table) |
| `avro-confluent.basic-auth.user-info` | No | — | `username:password` for auth |
| `avro-confluent.bearer-auth.token` | No | — | Bearer token for auth |
| `avro-confluent.ssl.keystore.location` | No | — | SSL keystore path |
| `avro-confluent.ssl.keystore.password` | No | — | SSL keystore password |
| `avro-confluent.ssl.truststore.location` | No | — | SSL truststore path |
| `avro-confluent.ssl.truststore.password` | No | — | SSL truststore password |
| `avro-confluent.properties` | No | — | Additional SR client properties (map) |

Options are prefixed with `key.` or `value.` depending on position:
- `value.avro-confluent.url`
- `key.avro-confluent.url`

### Other Supported Formats

While `avro-confluent` is the primary Schema Registry format in open-source Flink, you can also use:

- **`json`**: Plain JSON without Schema Registry (no schema enforcement)
- **`avro`**: Plain Avro without Schema Registry (schema embedded in each file)
- **`csv`**: CSV format (no schema enforcement)
- **`raw`**: Pass-through bytes (useful for keys)

For Protobuf and JSON Schema with Schema Registry in Flink, use the Confluent-provided format plugins or Confluent Cloud.

## Data Type Mapping

### Flink SQL to Avro

| Flink SQL Type | Avro Type |
|---------------|-----------|
| `BOOLEAN` | `boolean` |
| `TINYINT`, `SMALLINT`, `INT` | `int` |
| `BIGINT` | `long` |
| `FLOAT` | `float` |
| `DOUBLE` | `double` |
| `STRING` | `string` |
| `BYTES` | `bytes` |
| `DECIMAL(p, s)` | `bytes` (logical type `decimal`) |
| `DATE` | `int` (logical type `date`) |
| `TIME` | `int` (logical type `time-millis`) |
| `TIMESTAMP(3)` | `long` (logical type `timestamp-millis`) |
| `TIMESTAMP_LTZ(3)` | `long` (logical type `timestamp-millis`) |
| `ARRAY<T>` | `array` |
| `MAP<K, V>` | `map` |
| `ROW<...>` | `record` |

Nullable Flink types map to Avro `union(type, null)`.

## Common Patterns

### Stream-Table Join with Schema Registry

```sql
-- Fact stream
CREATE TABLE orders (
  order_id BIGINT,
  customer_id BIGINT,
  amount DECIMAL(10, 2),
  order_time TIMESTAMP_LTZ(3),
  WATERMARK FOR order_time AS order_time - INTERVAL '5' SECOND
) WITH (
  'connector' = 'kafka',
  'topic' = 'orders',
  'properties.bootstrap.servers' = 'localhost:9092',
  'scan.startup.mode' = 'earliest-offset',
  'value.format' = 'avro-confluent',
  'value.avro-confluent.url' = 'http://localhost:8081'
);

-- Dimension table (compacted topic)
CREATE TABLE customers (
  customer_id BIGINT,
  name STRING,
  email STRING,
  PRIMARY KEY (customer_id) NOT ENFORCED
) WITH (
  'connector' = 'upsert-kafka',
  'topic' = 'customers',
  'properties.bootstrap.servers' = 'localhost:9092',
  'key.format' = 'raw',
  'value.format' = 'avro-confluent',
  'value.avro-confluent.url' = 'http://localhost:8081'
);

-- Join
SELECT o.order_id, c.name, o.amount
FROM orders o
JOIN customers FOR SYSTEM_TIME AS OF o.order_time AS c
  ON o.customer_id = c.customer_id;
```

### Windowed Aggregation

```sql
SELECT
  window_start,
  window_end,
  customer_id,
  SUM(amount) AS total_amount,
  COUNT(*) AS order_count
FROM TABLE(
  TUMBLE(TABLE orders, DESCRIPTOR(order_time), INTERVAL '1' HOUR)
)
GROUP BY window_start, window_end, customer_id;
```

### Schema Evolution in Flink

When a schema evolves in Schema Registry:
- **Reading**: Flink uses its table definition as the reader schema; the writer schema is fetched per-record via schema ID. Compatible differences are handled automatically (e.g., new fields get defaults).
- **Writing**: Flink infers the schema from the table definition and registers it. If the inferred schema is incompatible with the existing subject, the write fails.

To handle evolution, alter the Flink table to match the new schema, or use `use.latest.version` semantics.
