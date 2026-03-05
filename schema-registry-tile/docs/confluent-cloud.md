# Confluent Cloud Schema Registry

## Table of Contents
1. [Overview](#overview)
2. [Tiers and Features](#tiers-and-features)
3. [Authentication and Access](#authentication-and-access)
4. [API Endpoints](#api-endpoints)
5. [Data Contracts](#data-contracts)
6. [Schema Linking](#schema-linking)
7. [Stream Governance](#stream-governance)

## Overview

Confluent Cloud provides a fully managed Schema Registry as part of Stream Governance. Each Confluent Cloud environment has its own Schema Registry instance (one per environment, shared across clusters in that environment).

## Tiers and Features

### Essentials (included)
- Avro, Protobuf, JSON Schema support
- Compatibility checking (all modes)
- REST API access
- Schema versioning
- Basic schema management in the UI

### Advanced (requires Stream Governance Advanced package)
- Data contracts (rules, metadata, tags)
- Schema validation (broker-side)
- Data quality rules
- Data transformation rules
- Schema linking (cross-environment)
- Audit logging
- Business metadata and tags

## Authentication and Access

### API Key Authentication

```bash
# Create a Schema Registry API key via Confluent CLI
confluent api-key create --resource <schema-registry-cluster-id>

# Use in REST calls
curl -u <SR_API_KEY>:<SR_API_SECRET> \
  https://<SR_ENDPOINT>/subjects
```

### Finding Your Schema Registry Endpoint

```bash
# Via Confluent CLI
confluent schema-registry cluster describe

# Output includes endpoint URL like:
# https://psrc-xxxxx.us-east-2.aws.confluent.cloud
```

### Client Configuration

```properties
# Java producer/consumer
schema.registry.url=https://psrc-xxxxx.us-east-2.aws.confluent.cloud
basic.auth.credentials.source=USER_INFO
basic.auth.user.info=<SR_API_KEY>:<SR_API_SECRET>
```

### Confluent CLI Commands

```bash
# List schemas
confluent schema-registry schema list

# Describe a schema
confluent schema-registry schema describe --subject my-topic-value --version latest

# Create a schema
confluent schema-registry schema create --subject my-topic-value \
  --schema schema.avsc --type AVRO

# Delete a schema
confluent schema-registry schema delete --subject my-topic-value --version latest

# Get/set compatibility
confluent schema-registry compatibility describe --subject my-topic-value
confluent schema-registry compatibility update --subject my-topic-value \
  --compatibility FULL_TRANSITIVE
```

## API Endpoints

The REST API is identical to the open-source API but accessed over HTTPS with authentication:

```bash
# List subjects
curl -u <KEY>:<SECRET> https://psrc-xxxxx.us-east-2.aws.confluent.cloud/subjects

# Register schema
curl -u <KEY>:<SECRET> \
  -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "...", "schemaType": "AVRO"}' \
  https://psrc-xxxxx.us-east-2.aws.confluent.cloud/subjects/my-topic-value/versions

# Test compatibility
curl -u <KEY>:<SECRET> \
  -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "..."}' \
  https://psrc-xxxxx.us-east-2.aws.confluent.cloud/compatibility/subjects/my-topic-value/versions/latest?verbose=true
```

## Data Contracts

Data contracts extend schemas with rules, metadata, and tags (Advanced tier).

### Metadata

Attach business metadata to schemas:

```json
{
  "schema": "...",
  "metadata": {
    "properties": {
      "owner": "data-team",
      "classification": "PII"
    },
    "tags": {
      "*.name": ["PII", "SENSITIVE"]
    }
  }
}
```

### Rules

Rules execute during serialization/deserialization:

```json
{
  "schema": "...",
  "ruleSet": {
    "domainRules": [
      {
        "name": "encrypt-pii",
        "kind": "TRANSFORM",
        "type": "ENCRYPT",
        "mode": "WRITEREAD",
        "tags": ["PII"],
        "params": {
          "encrypt.kek.name": "my-kek",
          "encrypt.kms.type": "aws-kms"
        }
      }
    ],
    "migrationRules": [
      {
        "name": "upgrade-v1-to-v2",
        "kind": "TRANSFORM",
        "type": "JSONATA",
        "mode": "UPGRADE",
        "expr": "$ ~> |$|{\"new_field\": \"default\"}|"
      }
    ]
  }
}
```

**Rule types**:
- `CONDITION`: Validate data quality (fail if condition not met)
- `TRANSFORM`: Transform data on read/write (e.g., encryption, migration)

**Rule modes**:
- `WRITE`: Applied during serialization
- `READ`: Applied during deserialization
- `WRITEREAD`: Applied in both directions
- `UPGRADE`: Applied when reading older schema versions
- `DOWNGRADE`: Applied when reading newer schema versions

### Broker-Side Schema Validation

Enable at the topic level to reject messages that don't match registered schemas:

```bash
confluent kafka topic update my-topic \
  --config confluent.value.schema.validation=true
```

This ensures all messages on the topic conform to the registered schema, even from producers that don't use Schema Registry serializers.

## Schema Linking

Schema Linking mirrors schemas between Schema Registry instances (e.g., across environments or regions).

```bash
# Create a schema exporter
confluent schema-registry exporter create my-exporter \
  --subjects "orders-*" \
  --context-type CUSTOM \
  --context ".dest-context" \
  --config exporter-config.properties
```

Use cases:
- Disaster recovery (replicate schemas to DR environment)
- Multi-region deployment
- Dev → staging → prod schema promotion

## Stream Governance

### Tags

Classify schemas and fields with business tags:

```bash
# Create a tag definition
confluent schema-registry tag create --name PII --description "Personally Identifiable Information"

# Tag a field
confluent schema-registry tag add --entity-type sr_field \
  --entity-name ":.:<subject>:1:name" --tag PII
```

### Stream Catalog

Search and discover schemas, topics, and their relationships:

```bash
# Search by tag
confluent schema-registry search --tag PII

# Search by name
confluent schema-registry search --name "orders*"
```

### Schema Exporter/Importer

For migrating schemas between environments:

```bash
# Export schemas
confluent schema-registry exporter list
confluent schema-registry exporter describe my-exporter
confluent schema-registry exporter status my-exporter
```
