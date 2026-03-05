# Expanding User Profile Schema for a New Feature

## Problem/Feature Description

The platform engineering team at Acme Corp manages a high-volume Kafka topic called `user-profiles` that streams user profile update events to dozens of downstream consumers — recommendation engines, audit loggers, and analytics dashboards. The topic currently uses an Avro schema with three fields: `user_id` (int), `username` (string), and `email` (string). All consumers and producers are deployed and actively processing live data.

A new product initiative requires adding a `phone_number` field and a `preferred_language` field to the user profile schema so that the notification service can personalize delivery. Not all users have a phone number or preferred language on file, so these fields are optional. The data-platform team needs to evolve this schema safely without disrupting any running consumers, and without violating the existing compatibility mode configured for this subject. The team also needs to produce a runnable shell script that captures the entire migration workflow so that it can be reviewed, version-controlled, and re-run in other environments.

## Output Specification

Produce the following files:

- `v2_schema.json` — The updated Avro schema (version 2) with the new optional fields added correctly.
- `evolve.sh` — A self-contained bash script that carries out the complete schema evolution process. The script must:
  - Check whether the new schema is compatible with the current registered schema before attempting to register it.
  - Register the new schema if compatibility is confirmed.
  - Verify the registration by retrieving the latest version.
  - Print the recommended service deployment order given the compatibility mode in use.
- `deploy_order.txt` — A short text file stating which service type (consumers or producers) should be updated first after the schema is registered, and why.

## Input Files

The following files are provided as inputs. Extract them before beginning.

=============== FILE: inputs/current_schema.json ===============
{
  "type": "record",
  "name": "UserProfile",
  "namespace": "com.acme.profiles",
  "fields": [
    {"name": "user_id", "type": "int"},
    {"name": "username", "type": "string"},
    {"name": "email", "type": "string"}
  ]
}
=============== END FILE ===============

=============== FILE: inputs/registry_info.txt ===============
Schema Registry URL: http://localhost:8081
Subject: user-profiles-value
Compatibility mode: BACKWARD
=============== END FILE ===============
