## Kafka

### Overview

It's a distributed commit log. A log (a.k.a. {write-ahead, commit, transaction} log) is the simplest data structure. A log is an ordered structure that is append-only and immutable.

Benefits:
- ordered
- immutable
- O(1) write/read
- efficient for HDDs (linear writes/reads, no random block accessing)
- speed does not degrade with size
- efficient for concurrent reads

---
### Core structure

**Structure**

Kafka stores data in topics. They’re split into partitions, and replicated across brokers, i.e.:

- topics
    - partitions
        - replicas
            - logs

Brokers host replicas of partitions.

A partition is replicated according to the replication factor (usually 3).
One of those 3 brokers is the leader for the partition, meaning all writes go to it.
Followers can read from any replica.

if a broker/disk breaks, another broker becomes the leader and usage continues (Durability / HA).

**Flow**

- Clients, or *producers*, send messages (records) to a Kafka node (broker).
- Other clients, called *consumers*, read and processed said messages.

These connect to the broker via *TCP* and use a custom protocol (the Kafka protocol) to communicate to the broker.

The producer/consumer clients are simply a *Java* library that implements the Kafka protocol and gives you a nice interface to interact with. Nowadays there’s an implementation in virtually every language out there.

---
### Controllers

A distributed system requires coordination.

Kafka achieves this by having specific brokers called *controllers*.

At any one point, there is only *one* active controller.

The controller is the main source of truth for metadata within the cluster.
It’s responsible for a bunch of things, one of the main being handling broker failures by changing partition leadership.

- Leader election between regular brokers is done through the controller.
- Leader election between the controllers (picking the active one) is done through a variant of Raft (KRaft).
    - Raft is a popular consensus algorithm used in may distributed systems, e.g. ScyllaDB uses Raft to manage schema updates and to manage cluster topology.

All the cluster’s metadata is stored inside a simple (but special) Kafka topic whose leader is the active controller.

All brokers replicate this topic - that’s how they get metadata.

---
### Extentions

To support Kafka's role as a core component of stream data processing and orchestration, it offers two key open-source tools: Kafka Connect and Kafka Streams.

- Streams - an embeddable lightweight library for stream processing with Kafka.

- Connect - a framework and runtime for plugins which integrate Kafka with external systems.

#### Streams
Kafka Streams is a client library for building applications that transform / react to stream data. The input and output data are stored in Kafka topics. It combines the simplicity of writing standard Java and Scala applications on the client side with the benefits of Kafka's server-side cluster technology for fault-tolerant, stateful stream processing.

Features:
- Stateful processing - provides tools (like aggregations and windowing) to keep track of the state. State can be recovered if the application crashes (using Kafka's fault-tolerant design).
- Queriable state allows applications to query the state stored in Kafka directly, making data more accessible.
- Exactly-once semantics ensure that each record is processed exactly once, thereby preventing data duplication and ensuring data integrity, even in a failure.
- Join operations provide the capability to join streams and tables in various ways, enabling complex stream processing use cases.
- Custom partitioning offers control over how data is partitioned and processed, allowing for more optimized and efficient stream processing.
- No need for extra distributed systems besides a running Kafka cluster.

Streams topology:
A *processor topology* or simply *topology* defines the stream processing computational logic for your application, i.e., how input data is transformed into output data. A topology is a graph of stream processors (nodes) that are connected by streams (edges) or shared state stores. There are two special processors:
- Source Processor: A stream processor without upstream processors. It consumes records from Kafka topics and forwards them downstream.
- Sink Processor: A stream processor without downstream processors. It sends records from upstream processors to a specified Kafka topic.

#### Connect

Kafka Connect is a scalable framework for streaming data between Kafka and external systems like databases, files, or cloud services, enabling seamless integration without extensive custom coding.

Use Cases:
- Sync database changes with Kafka for real-time analytics, e.g. Postgres.
- Stream Kafka data to data lakes for storage and processing, e.g. S3.
- Aggregate logs into Kafka (ready-made connectors for log sources (e.g., syslog, file systems)).
- Connect IoT devices to Kafka for real-time processing (supports various protocols (e.g., MQTT, HTTP)).
- Update search indexes like Elasticsearch in real time.

#### Flink
In 2023, Confluent acquired Immerok, a leading contributor to Apache Flink.

In 2024, Confluent dropped ksqlDB in favor of Flink.
Apache Flink is a data processing framework that can act on both data streams and batches. It has advanced processing capabilities but requires a data ingestion tool like Kafka.

Flink can be used instead of Kafka Streams for additional stream processing and transformation functions.  It is a running engine on which processing jobs run; this implies that if users want to leverage Flink for stream processing, they will need to work with two systems.

Features:
- Machine learning library: Flink has a machine learning library, making it possible to run ML algorithms on data streams - directly within Flink.
- Temporal table joins: Flink supports joining a dynamic table (a stream) with a static table, enabling temporal queries and simplifying stream processing logic.
- Dynamic scaling: demonstrates superior scalability, capable of managing complex and voluminous data streams.
- Supports various programming languages and offers API abstractions like PyFlink and Table APIs for enhanced flexibility.
- Savepoints and state migration allow for creating savepoints, facilitating updates and migrations without data loss, and ensuring business continuity.

---
### Avro
Apache Avro is a data serialization framework commonly used in big data applications.
Serialization is the process of converting data structures or objects into a format that can be easily stored or transmitted (e.g., a binary or text format).

Avro is often chosen as the serialization format for the messages because it provides a schema-based approach that ensures data compatibility and efficiency.

Avro in action:
- Producer:
    - Serializes data using Avro before sending it to Kafka topics.
    - Includes the schema ID (or fingerprint) with the message.

- Consumer:
    - Deserializes messages using the schema referenced by the schema ID.
    - Ensures the data is understood, even as schemas evolve.

- Schema Registry:
    - An external service (e.g., Confluent Schema Registry) stores Avro schemas.
    - Producers and consumers communicate with the registry to fetch or validate schemas.

---
### Sources
- https://blog.2minutestreaming.com/p/summary-introduction-apache-kafka
- https://www.redpanda.com/guides/event-stream-processing-kafka-streams-vs-flink
- https://bitrock.it/blog/technology/apache-flink-and-kafka-stream-a-comparative-analysis.html
- https://docs.confluent.io/platform/current/streams/architecture.html
- https://raft.github.io/
- https://opensource.docs.scylladb.com/stable/architecture/raft.html
- https://www.confluent.io/press-release/confluent-plans-immerok-acquisition-to-accelerate-cloud-native-apache-flink/
- https://x.com/jaykreps/status/1796992401791385640?prefetchTimestamp=1736593960941&mx=2
- https://docs.confluent.io/platform/current/schema-registry/fundamentals/serdes-develop/serdes-avro.html
