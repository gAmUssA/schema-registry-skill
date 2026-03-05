# Flink SQL Pipeline for Order Enrichment

## Problem/Feature Description

The data engineering team at RetailCo needs to build a real-time order enrichment pipeline using Apache Flink. The Kafka cluster already has two topics — `orders` and `customers` — both of which store messages serialized with Avro and backed by a Confluent Schema Registry instance. The `orders` topic has both a key (the order ID as a string) and a value serialized using the Confluent wire format. The `customers` topic uses upsert semantics with a raw string key and an Avro value.

The team wants to join the streaming orders with the customer dimension table to produce enriched records with the customer name included. They need a set of Flink SQL DDL statements and a pom.xml that shows the correct Maven dependencies so the pipeline can be compiled and submitted to a Flink cluster. A data engineer will review the files to make sure the setup will connect properly to Schema Registry and that schema changes in the registry will be picked up correctly at runtime.

## Output Specification

Produce the following files:

- `pom.xml` — A Maven POM file showing the correct dependencies (and repository declarations) required to run Flink SQL with Schema Registry support. Include at minimum the necessary Flink + Schema Registry JARs. Use Flink version 1.20.0.
- `pipeline.sql` — Flink SQL statements that:
  - Define the `orders` table reading from the `orders` Kafka topic, where both the key and the value are Avro-encoded via Schema Registry. The key field should be named `order_key`.
  - Define the `customers` table reading from the `customers` Kafka topic as an upsert source, where the value is Avro-encoded via Schema Registry and the key is a raw string `customer_id`.
  - Include a SELECT query that joins orders to customers to produce enriched output (order_id, product, customer_name, amount).
- `notes.md` — A short technical note (bullet points are fine) explaining any configuration choices made for Schema Registry integration in the DDL, and what each relevant WITH property does.

Assume the Schema Registry is at `http://schema-registry:8081`, Kafka bootstrap servers are at `kafka:9092`, and the orders topic has two subjects registered: `orders-key` and `orders-value`. The customers topic has subject `customers-value`.
