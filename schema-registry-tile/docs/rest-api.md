# Schema Registry REST API Reference

## Table of Contents
1. [General Information](#general-information)
2. [Schemas Endpoints](#schemas-endpoints)
3. [Subjects Endpoints](#subjects-endpoints)
4. [Compatibility Endpoints](#compatibility-endpoints)
5. [Config Endpoints](#config-endpoints)
6. [Mode Endpoints](#mode-endpoints)

## General Information

**Base URL**: `http://localhost:8081` (default)

**Content Type**: `application/vnd.schemaregistry.v1+json`

**Authentication** (Confluent Cloud):
```bash
curl -u <API_KEY>:<API_SECRET> https://<SR_ENDPOINT>
```

**Error Response Format**:
```json
{
  "error_code": 42201,
  "message": "Schema being registered is incompatible with an earlier schema"
}
```

Common error codes:
- `40401` — Subject not found
- `40402` — Version not found
- `40403` — Schema not found
- `42201` — Invalid schema
- `42202` — Invalid version
- `409` — Incompatible schema

## Schemas Endpoints

### Get Schema by ID
```bash
GET /schemas/ids/{id}

curl http://localhost:8081/schemas/ids/1

# Response
{
  "schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[...]}"
}
```

### Get Schema by ID (specific format)
```bash
GET /schemas/ids/{id}?format=serialized
```

### Get Schema Types
```bash
GET /schemas/types

curl http://localhost:8081/schemas/types

# Response
["AVRO", "PROTOBUF", "JSON"]
```

### Get Schema Versions by ID
```bash
GET /schemas/ids/{id}/versions

# Response — lists subjects and versions using this schema
[
  {"subject": "my-topic-value", "version": 1}
]
```

## Subjects Endpoints

### List All Subjects
```bash
GET /subjects

curl http://localhost:8081/subjects

# Response
["my-topic-key", "my-topic-value", "orders-value"]
```

### List Versions Under a Subject
```bash
GET /subjects/{subject}/versions

curl http://localhost:8081/subjects/my-topic-value/versions

# Response
[1, 2, 3]
```

### Get Schema by Subject and Version
```bash
GET /subjects/{subject}/versions/{version}

# {version} can be an integer or "latest"
curl http://localhost:8081/subjects/my-topic-value/versions/latest

# Response
{
  "subject": "my-topic-value",
  "id": 1,
  "version": 1,
  "schemaType": "AVRO",
  "schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[...]}"
}
```

### Register a New Schema Under a Subject
```bash
POST /subjects/{subject}/versions

curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{
    "schema": "{\"type\":\"record\",\"name\":\"User\",\"namespace\":\"com.example\",\"fields\":[{\"name\":\"id\",\"type\":\"int\"},{\"name\":\"name\",\"type\":\"string\"}]}"
  }' \
  http://localhost:8081/subjects/my-topic-value/versions

# Response
{"id": 1}
```

**For Protobuf**:
```bash
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{
    "schemaType": "PROTOBUF",
    "schema": "syntax = \"proto3\"; message User { int32 id = 1; string name = 2; }"
  }' \
  http://localhost:8081/subjects/my-topic-value/versions
```

**For JSON Schema**:
```bash
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{
    "schemaType": "JSON",
    "schema": "{\"type\":\"object\",\"properties\":{\"id\":{\"type\":\"integer\"},\"name\":{\"type\":\"string\"}},\"required\":[\"id\",\"name\"]}"
  }' \
  http://localhost:8081/subjects/my-topic-value/versions
```

**With Schema References**:
```bash
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{
    "schema": "...",
    "schemaType": "AVRO",
    "references": [
      {
        "name": "com.example.Address",
        "subject": "address-value",
        "version": 1
      }
    ]
  }' \
  http://localhost:8081/subjects/my-topic-value/versions
```

### Check if Schema is Registered Under Subject
```bash
POST /subjects/{subject}

curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "..."}' \
  http://localhost:8081/subjects/my-topic-value

# Response (if registered)
{
  "subject": "my-topic-value",
  "id": 1,
  "version": 1,
  "schema": "..."
}
```

### Delete Subject (all versions)
```bash
DELETE /subjects/{subject}

curl -X DELETE http://localhost:8081/subjects/my-topic-value

# Response — list of deleted versions
[1, 2, 3]
```

### Delete Specific Version
```bash
DELETE /subjects/{subject}/versions/{version}

curl -X DELETE http://localhost:8081/subjects/my-topic-value/versions/1

# Response
1
```

### Permanent Delete (hard delete)
```bash
# First soft-delete, then hard-delete with ?permanent=true
DELETE /subjects/{subject}?permanent=true
DELETE /subjects/{subject}/versions/{version}?permanent=true
```

## Compatibility Endpoints

### Test Compatibility Against a Specific Version
```bash
POST /compatibility/subjects/{subject}/versions/{version}

curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "..."}' \
  http://localhost:8081/compatibility/subjects/my-topic-value/versions/latest

# Response
{"is_compatible": true}

# With verbose mode for details on failures
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "..."}' \
  "http://localhost:8081/compatibility/subjects/my-topic-value/versions/latest?verbose=true"

# Response (incompatible)
{
  "is_compatible": false,
  "messages": [
    "Incompatibility{type:READER_FIELD_MISSING_DEFAULT_VALUE, ...}"
  ]
}
```

### Test Compatibility Against All Versions
```bash
POST /compatibility/subjects/{subject}/versions

# Same request format, tests against all versions based on configured compatibility
```

**Query Parameters**:
- `verbose=true` — Returns detailed incompatibility messages
- `normalize=true` — Normalizes schema before checking

## Config Endpoints

### Get Global Compatibility Level
```bash
GET /config

curl http://localhost:8081/config

# Response
{"compatibilityLevel": "BACKWARD"}
```

### Update Global Compatibility Level
```bash
PUT /config

curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "FULL"}' \
  http://localhost:8081/config

# Response
{"compatibility": "FULL"}
```

### Get Subject-Level Compatibility
```bash
GET /config/{subject}

curl http://localhost:8081/config/my-topic-value

# With fallback to global if not set
curl "http://localhost:8081/config/my-topic-value?defaultToGlobal=true"
```

### Update Subject-Level Compatibility
```bash
PUT /config/{subject}

curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "NONE"}' \
  http://localhost:8081/config/my-topic-value
```

### Delete Subject-Level Compatibility (revert to global)
```bash
DELETE /config/{subject}
```

## Mode Endpoints

### Get Global Mode
```bash
GET /mode

# Response
{"mode": "READWRITE"}
```

Modes: `READWRITE` (default), `READONLY`, `READONLY_OVERRIDE`, `IMPORT`

### Set Global Mode
```bash
PUT /mode

curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"mode": "READONLY"}' \
  http://localhost:8081/mode
```

### Get/Set Subject-Level Mode
```bash
GET /mode/{subject}
PUT /mode/{subject}
```
