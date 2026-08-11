# Kafka Deep Dive — Consolidated Project Notes

> A practical reference distilled from the Kafka discussions in this project, with emphasis on Java, Spring Kafka, producer/consumer internals, delivery semantics, transactions, retries, error handling, DLQs, SSL, and architecture trade-offs.

## Table of Contents

1. [Kafka Mental Model](#1-kafka-mental-model)
2. [Producer Architecture](#2-producer-architecture)
3. [Producer Batching](#3-producer-batching)
4. [Producer Buffering and Application Shutdown](#4-producer-buffering-and-application-shutdown)
5. [Producer Send Lifecycle](#5-producer-send-lifecycle)
6. [Retries, Ordering, and Idempotence](#6-retries-ordering-and-idempotence)
7. [Partition Selection](#7-partition-selection)
8. [Producer Transactions](#8-producer-transactions)
9. [Batch Failures and Callbacks](#9-batch-failures-and-callbacks)
10. [Consumer Polling Model](#10-consumer-polling-model)
11. [Offsets and Auto Commit](#11-offsets-and-auto-commit)
12. [Spring Kafka Ack Modes](#12-spring-kafka-ack-modes)
13. [RECORD Mode vs Auto Commit](#13-record-mode-vs-auto-commit)
14. [MANUAL vs MANUAL_IMMEDIATE](#14-manual-vs-manual_immediate)
15. [max.poll.records and Processing](#15-maxpollrecords-and-processing)
16. [Offset Reset and Replay](#16-offset-reset-and-replay)
17. [Deserialization Failures](#17-deserialization-failures)
18. [DefaultErrorHandler](#18-defaulterrorhandler)
19. [Retries and Dead Letter Topics](#19-retries-and-dead-letter-topics)
20. [Consumer Failure Architecture](#20-consumer-failure-architecture)
21. [Exactly-Once Semantics](#21-exactly-once-semantics)
22. [Spring Kafka Transactions](#22-spring-kafka-transactions)
23. [Kafka SSL and Certificate Identity](#23-kafka-ssl-and-certificate-identity)
24. [Spring Boot and @EnableKafka](#24-spring-boot-and-enablekafka)
25. [Kafka vs IBM MQ](#25-kafka-vs-ibm-mq)
26. [Transactional Outbox Pattern](#26-transactional-outbox-pattern)
27. [Configuration Cheat Sheet](#27-configuration-cheat-sheet)
28. [Interview-Level Mental Models](#28-interview-level-mental-models)

---

## 1. Kafka Mental Model

Kafka is a distributed append-only log.

```text
Producer
   |
   v
Topic
   |
   +---- Partition 0: [0][1][2][3][4]...
   |
   +---- Partition 1: [0][1][2][3]...
   |
   +---- Partition 2: [0][1][2]...
                           |
                           v
                      Consumer Group
```

Important ideas:

- A topic is divided into partitions.
- Ordering is guaranteed only within a partition.
- Each record receives its own offset.
- Producer batching is a transport optimization; a batch is **not** one Kafka record and does not receive one shared offset.
- Consumers track progress using committed offsets.
- Kafka stores records independently of whether a consumer has processed them.

---

## 2. Producer Architecture

A simplified Kafka producer:

```text
Application Thread
      |
      | producer.send(record)
      v
Serializer
      |
      v
Partitioner / partition selection
      |
      v
RecordAccumulator
(per-partition batches in memory)
      |
      v
Sender Thread
      |
      v
Kafka Broker
```

`producer.send()` normally does not synchronously perform the complete network write.

It generally:

1. serializes the key/value,
2. determines the partition,
3. appends the record to an in-memory batch for that partition,
4. returns a future,
5. lets the producer sender thread transmit ready batches asynchronously.

This architecture is why Kafka producers can achieve high throughput.

---

## 3. Producer Batching

Important settings:

```properties
batch.size=16384
linger.ms=5
buffer.memory=33554432
```

### `batch.size`

This is approximately the target maximum amount of data accumulated into one producer batch for a partition.

Suppose:

```text
batch.size = 5 KB
record A   = 2 KB
record B   = 2 KB
record C   = 2 KB
```

Conceptually:

```text
Batch 1
+---------+---------+
|   2 KB  |   2 KB  |
+---------+---------+
    4 KB

Batch 2
+---------+
|   2 KB  |
+---------+
```

Kafka does not split an individual record merely to make it fit a configured batch target.

### Record larger than `batch.size`

If:

```text
batch.size = 16 KB
record     = 30 KB
```

the record can still be sent using a larger batch allocation, subject to other limits such as:

- producer `max.request.size`,
- broker/topic maximum message size,
- memory availability.

Therefore:

> `batch.size` is not the maximum Kafka record size.

### `linger.ms`

`linger.ms` allows the producer to wait briefly for additional records so that a batch can become fuller.

A batch can become sendable because:

```text
batch full
   OR
linger time reached
   OR
other producer/network conditions trigger sending
```

It is incorrect to think:

> Kafka sends only when `batch.size` becomes full.

### Serialization and linger

Serialization occurs as part of `send()` before the record is accumulated.

So the model is not:

```text
wait for serialization until linger expires
```

Rather:

```text
send()
  -> serialize
  -> select partition
  -> append to accumulator
  -> linger may delay transmission of accumulated data
```

---

## 4. Producer Buffering and Application Shutdown

The producer's `RecordAccumulator` is in application memory.

```text
Application JVM
+----------------------------------+
| KafkaProducer                    |
|                                  |
| RecordAccumulator                |
|   P0 -> [batch]                  |
|   P1 -> [batch]                  |
|                                  |
| Sender Thread -----------------------> Broker
+----------------------------------+
```

If the process crashes before buffered records are successfully acknowledged by Kafka, those in-memory records can be lost.

Kafka cannot recover records that never reached the broker.

### Graceful shutdown

Use:

```java
producer.flush();
producer.close();
```

`close()` attempts to flush pending sends within its allowed shutdown behavior.

But graceful shutdown does not protect against:

- JVM kill -9,
- machine crash,
- container termination without enough grace time,
- process failure before the record is durably acknowledged.

For business-critical events requiring durability across application crashes, use a durable pattern such as the transactional outbox.

---

## 5. Producer Send Lifecycle

```text
producer.send(record)
       |
       v
serialization
       |
       v
partition selection
       |
       v
append to RecordAccumulator
       |
       v
sender finds batch ready
       |
       v
ProduceRequest
       |
       v
Broker leader
       |
       +--> append record(s)
       |
       +--> replicate according to acks/min ISR
       |
       v
ProduceResponse
       |
       v
Future / Callback
```

Example:

```java
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        log.error("Send failed", exception);
    } else {
        log.info(
            "topic={}, partition={}, offset={}",
            metadata.topic(),
            metadata.partition(),
            metadata.offset()
        );
    }
});
```

The callback is associated with the individual `send()` operation even though multiple records may travel together in one broker request/batch.

---

## 6. Retries, Ordering, and Idempotence

Kafka producer retries many retriable failures internally.

Relevant settings include:

```properties
retries=...
retry.backoff.ms=...
delivery.timeout.ms=...
acks=all
enable.idempotence=true
```

### Same partition on retry

Once a record has been assigned to a partition, retrying the failed send does not normally repartition that record to another partition.

```text
Record
  |
  v
Partition 2
  |
 first attempt -> failure
  |
 retry
  v
Partition 2
```

This is essential for partition-local ordering.

### Idempotent producer

With:

```properties
enable.idempotence=true
```

Kafka assigns producer identity/sequence information so the broker can detect duplicate retries.

Conceptually:

```text
Producer ID = P42

Partition 0:
sequence 0
sequence 1
sequence 2
```

Suppose sequence 2 is written, but the acknowledgment is lost:

```text
Producer ---- seq=2 ----> Broker
Producer <--- ACK lost ---
Producer ---- seq=2 ----> Broker
```

The broker recognizes that the retried sequence was already appended and avoids creating another logical copy.

Idempotence therefore addresses duplicates caused by producer retries.

It does **not** by itself make arbitrary database/API side effects exactly once.

---

## 7. Partition Selection

A producer record may explicitly specify a partition:

```java
ProducerRecord<String, String> record =
    new ProducerRecord<>("orders", 2, null, payload);
```

A key is not required when an explicit partition is supplied.

Conceptually:

```text
ProducerRecord(topic, partition=2, key=null, value)
                         |
                         v
                    Partition 2
```

If a partition is not explicitly specified, Kafka's producer partition-selection behavior determines the target based on the key and producer partitioning strategy.

A critical batching rule:

> Producer batches are partition-specific.

Records for partition 0 and partition 1 are not placed in the same producer batch.

---

## 8. Producer Transactions

Transactions provide atomicity across Kafka operations.

```java
producer.initTransactions();

producer.beginTransaction();

producer.send(record1);
producer.send(record2);
producer.send(record3);

producer.commitTransaction();
```

If the transaction must fail:

```java
producer.abortTransaction();
```

Conceptually:

```text
Transaction T1
 |
 +-- Batch -> Topic A / P0
 +-- Batch -> Topic A / P1
 +-- Batch -> Topic B / P2
 |
 +-- COMMIT
```

A transaction can span:

- multiple records,
- multiple batches,
- multiple partitions,
- multiple Kafka topics.

The transaction is **not** equivalent to a producer batch.

```text
Batch = transport/performance unit
Transaction = atomic visibility unit
```

### Visibility

A consumer using:

```properties
isolation.level=read_committed
```

only exposes committed transactional records.

Aborted transactional data remains in the log internally but is skipped by `read_committed` consumers.

---

## 9. Batch Failures and Callbacks

Consider three application sends:

```text
send(A)
send(B)
send(C)
```

They may be placed into one producer batch.

Callbacks remain associated with individual records.

If a request encounters a retriable error, Kafka generally retries the relevant batch internally before finally completing callbacks with success/failure.

A conceptual view:

```text
Application
   |
   +-- send A -> callback A
   +-- send B -> callback B
   +-- send C -> callback C

RecordAccumulator
   |
   +-- [ A | B | C ] batch
            |
            v
          Broker
```

Do not design application logic around an assumption that Kafka normally reports a physically half-written producer batch as “A and B succeeded but C failed” in the simplistic sense. Kafka's record batch/request protocol, retry behavior, broker append semantics, and callback completion determine the actual outcome.

### Without a transaction

Successful records remain successful even if another independent send ultimately fails.

There is no application-level rollback across separate records merely because they were conceptually related.

### With a transaction

The application can abort the transaction so `read_committed` consumers do not observe the transaction's records as committed.

### Should all records go to a DLQ?

Usually no.

A producer-side DLQ is an application policy, not a Kafka producer automatic mechanism.

Only records that are truly unrecoverable should normally be routed to a failure store/topic. Sending already-successful records again can itself introduce duplicate business processing unless the design explicitly requires whole-business-operation compensation.

---

## 10. Consumer Polling Model

```text
Kafka Broker
     |
     v
consumer.poll()
     |
     v
ConsumerRecords
     |
     +--> record 1
     +--> record 2
     +--> record 3
     |
     v
Application processing
     |
     v
Offset commit
```

Important settings:

```properties
group.id=my-group
max.poll.records=500
max.poll.interval.ms=300000
enable.auto.commit=false
```

### `max.poll.records`

This limits how many records a single `poll()` returns to the application.

It does **not** mean Kafka fetches only that many records from the broker at the network layer.

Kafka may have fetched/buffered more internally while returning up to `max.poll.records` to the application per poll.

---

## 11. Offsets and Auto Commit

A committed offset represents the position from which the consumer group should resume.

If record offset `9` has been successfully processed, committing `10` means:

```text
next record expected = offset 10
```

### `enable.auto.commit=true`

The Kafka consumer periodically commits offsets according to its auto-commit behavior.

This commit is not inherently tied to the completion of your business logic for each individual record.

That creates an important risk:

```text
poll returns records
      |
      v
auto-commit advances position
      |
      v
business processing fails
```

The consumer may restart after the committed position and therefore not automatically replay the failed business operation.

For controlled Spring Kafka processing, applications commonly use:

```properties
enable.auto.commit=false
```

and let the listener container manage commits according to its acknowledgment mode.

---

## 12. Spring Kafka Ack Modes

Common listener-container modes include:

```text
RECORD
BATCH
MANUAL
MANUAL_IMMEDIATE
```

### RECORD

After each successfully processed record, its offset becomes eligible for container-managed commit.

```text
record 10 -> listener succeeds -> commit
record 11 -> listener succeeds -> commit
record 12 -> failure
```

### BATCH

Offsets for the records returned by the poll are committed after the listener successfully processes the batch/poll unit according to the container's mode.

### MANUAL

Listener explicitly acknowledges:

```java
@KafkaListener(topics = "orders")
public void consume(String value, Acknowledgment ack) {
    process(value);
    ack.acknowledge();
}
```

Acknowledgment is application-controlled, while the actual commit follows the container's manual acknowledgment semantics.

### MANUAL_IMMEDIATE

The application also calls:

```java
ack.acknowledge();
```

but Spring attempts an immediate commit when the acknowledgment is performed on the consumer thread. Calling it from another thread can change the effective behavior, so thread ownership matters.

---

## 13. RECORD Mode vs Auto Commit

A key point from the project discussions:

```text
AckMode.RECORD
+
enable.auto.commit=true
```

does **not** mean Spring gets reliable “commit exactly after every record” semantics.

When Kafka consumer auto commit is enabled, Kafka's auto-commit mechanism controls offset commits rather than Spring's normal record-level container commit behavior.

So this combination defeats the purpose of choosing `RECORD` for deterministic container-managed commits.

Preferred conceptual setup:

```properties
enable.auto.commit=false
```

plus:

```java
factory.getContainerProperties()
       .setAckMode(ContainerProperties.AckMode.RECORD);
```

Then:

```text
poll()
  |
  +-- record A -> listener success -> container commit
  +-- record B -> listener success -> container commit
```

### What if `max.poll.records=1`?

With:

```text
max.poll.records = 1
AckMode.RECORD
enable.auto.commit = false
```

each poll delivers at most one record and successful processing allows a commit after that record.

However, `RECORD` does not require `max.poll.records=1`.

With ten records returned from one poll, Spring can process them one at a time and perform record-level offset management.

---

## 14. MANUAL vs MANUAL_IMMEDIATE

### MANUAL

```text
Listener
  |
  | ack.acknowledge()
  v
Container marks offset acknowledged
  |
  v
Commit according to container commit boundary
```

Useful when the application wants to decide **whether** processing is complete.

### MANUAL_IMMEDIATE

```text
Listener running on consumer thread
  |
  | ack.acknowledge()
  v
commit requested immediately
```

Use this when the timing of the commit itself must be tightly associated with the acknowledgment.

The important distinction is:

```text
MANUAL
  -> application chooses acknowledgement point

MANUAL_IMMEDIATE
  -> application chooses acknowledgement point
     + requests immediate commit when conditions permit
```

---

## 15. max.poll.records and Processing

Suppose:

```properties
max.poll.records=10
```

and the broker supplies:

```text
offsets 100 ... 109
```

With `RECORD`:

```text
100 -> process -> success
101 -> process -> success
102 -> process -> success
...
```

The acknowledgment semantics are record-oriented even though the records came from one poll.

With `BATCH`, the commit boundary is associated with the poll batch.

This is why these are different concepts:

```text
max.poll.records = delivery size to application

AckMode          = when Spring considers offsets ready to commit
```

---

## 16. Offset Reset and Replay

```properties
auto.offset.reset=earliest
```

means:

> If the consumer group has no valid committed offset for the partition, begin at the earliest available offset.

```properties
auto.offset.reset=latest
```

means:

> If there is no valid committed offset, start from the end and consume new records.

This setting is not a general “always replay from beginning” switch.

If valid committed offsets exist, those offsets are used.

### Replay

Replay usually does not require Kafka to create a brand-new consumer implementation internally.

A consumer can reposition itself using `seek()`:

```java
consumer.seek(topicPartition, targetOffset);
```

Conceptually:

```text
current position = 500
       |
       | seek(200)
       v
position = 200
       |
       v
reprocess records
```

Spring Kafka retry/error handling can seek/reposition partitions depending on configuration and error-handler strategy.

---

## 17. Deserialization Failures

A deserialization error occurs before the listener receives a normal domain object.

```text
Broker bytes
    |
    v
Deserializer
    |
    +--> success -> listener
    |
    +--> failure -> listener cannot receive normal object
```

For example, invalid JSON cannot normally be passed as a successfully constructed domain object.

### ErrorHandlingDeserializer

Spring Kafka's `ErrorHandlingDeserializer` wraps a real deserializer.

Conceptually:

```text
Kafka record bytes
      |
      v
ErrorHandlingDeserializer
      |
      +--> delegate deserializer
              |
              +--> valid -> object
              |
              +--> invalid -> capture deserialization failure metadata
```

This allows the listener container/error-handling infrastructure to deal with poison records instead of repeatedly failing at the raw Kafka deserializer boundary.

Example configuration conceptually:

```properties
spring.kafka.consumer.key-deserializer=
  org.springframework.kafka.support.serializer.ErrorHandlingDeserializer

spring.kafka.consumer.value-deserializer=
  org.springframework.kafka.support.serializer.ErrorHandlingDeserializer

spring.kafka.consumer.properties.spring.deserializer.value.delegate.class=
  org.springframework.kafka.support.serializer.JsonDeserializer
```

---

## 18. DefaultErrorHandler

`DefaultErrorHandler` handles failures that occur during listener processing and can also participate in handling deserialization failures surfaced through Spring's deserialization infrastructure.

Conceptual flow:

```text
Kafka record
    |
    v
deserialize
    |
    v
listener
    |
    +--> success
    |
    +--> exception
           |
           v
     DefaultErrorHandler
           |
           +--> retry
           +--> seek/recover
           +--> DLT
```

Example:

```java
@Bean
DefaultErrorHandler errorHandler(
        DeadLetterPublishingRecoverer recoverer) {

    FixedBackOff backOff =
        new FixedBackOff(1000L, 2L);

    return new DefaultErrorHandler(
        recoverer,
        backOff
    );
}
```

This means a failed record can be retried according to the configured backoff and then handed to a recoverer.

---

## 19. Retries and Dead Letter Topics

A common Spring Kafka pattern:

```text
orders
  |
  v
@KafkaListener
  |
  +--> success --------------------> commit
  |
  +--> failure
          |
          v
   DefaultErrorHandler
          |
          +--> retry
          +--> retry
          |
          v
        DLT
```

Example:

```java
@Bean
DeadLetterPublishingRecoverer recoverer(KafkaTemplate<Object, Object> template) {
    return new DeadLetterPublishingRecoverer(template);
}
```

A common destination convention is:

```text
orders -> orders.DLT
```

### What belongs in a DLT?

Typically:

- poison messages,
- invalid business data,
- records that repeatedly fail processing,
- unrecoverable downstream failures after retry policy is exhausted.

A DLT is valuable because it prevents one permanently failing record from blocking normal processing forever while preserving evidence for investigation/replay.

A DLT should not become a substitute for understanding failure categories.

---

## 20. Consumer Failure Architecture

A useful architecture separates failures:

```text
                         +----------------------+
Kafka record ---------->| Deserialization      |
                         +----------+-----------+
                                    |
                   +----------------+----------------+
                   |                                 |
                success                           failure
                   |                                 |
                   v                                 v
              Listener                        Error handling
                   |
          +--------+--------+
          |                 |
       success            exception
          |                 |
        commit      DefaultErrorHandler
                            |
                    +-------+-------+
                    |               |
                  retry           recover
                                    |
                                    v
                                   DLT
```

Classify failures into:

1. **Deserialization failure**  
   Bytes cannot become the expected Java object.

2. **Transient processing failure**  
   Database timeout, temporary downstream outage, etc.

3. **Permanent business failure**  
   Invalid state/data that retries will not fix.

4. **Infrastructure/consumer-liveness failure**  
   Processing exceeds `max.poll.interval.ms`, broker/network issues, rebalance, etc.

Retry only when retrying has a realistic chance of success.

---

## 21. Exactly-Once Semantics

Kafka terminology must be interpreted carefully.

### At-most-once

```text
commit
  |
  v
process
```

A crash after commit but before processing can cause loss from the application's perspective.

### At-least-once

```text
process
  |
  v
commit
```

A crash after processing but before commit can cause the record to be processed again.

Therefore business logic should often be idempotent.

### Kafka exactly-once

Kafka transactions can atomically combine consumed offsets and produced Kafka records.

```text
Consume input
    |
    v
Process
    |
    v
Produce output
    |
    v
Commit:
  output records
  +
  consumed offsets
```

Consumers using `read_committed` see only committed transactional output.

But:

> Kafka exactly-once semantics do not magically make an arbitrary external REST call, email, filesystem write, or ordinary database update exactly once.

External side effects need their own consistency/idempotency strategy.

---

## 22. Spring Kafka Transactions

A typical Spring Kafka transaction flow:

```text
@KafkaListener
      |
      v
Kafka transaction starts
      |
      +--> process input
      |
      +--> KafkaTemplate.send(...)
      |
      +--> send consumed offsets to transaction
      |
      v
transaction commit
```

Producer configuration uses a transactional producer identity, and Spring manages transactional producer instances.

A conceptual configuration:

```properties
spring.kafka.producer.transaction-id-prefix=my-app-
```

For consume-transform-produce:

```text
Input topic
   |
   v
Listener
   |
   +--> output topic
   |
   +--> input offset
          |
          v
   SAME Kafka transaction
```

If processing fails:

```text
transaction abort
   |
   +--> output is not visible to read_committed consumers
   |
   +--> consumed offset is not transactionally advanced
```

The input can therefore be retried.

---

## 23. Kafka SSL and Certificate Identity

In an enterprise Kafka setup, a consumer/producer may register certificate identity attributes such as:

```text
CN = Common Name
OU = Organizational Unit
```

The conceptual flow:

```text
Spring Kafka Application
       |
       | TLS handshake
       | client certificate
       v
Kafka endpoint / security layer
       |
       | extracts certificate identity
       v
Principal
       |
       | authorization check
       v
Topic ACL / organization portal registration
```

A portal that asks for certificate attributes is often registering the certificate identity/principal that is allowed to access a topic.

Client-side settings commonly involve:

```properties
security.protocol=SSL
ssl.truststore.location=...
ssl.truststore.password=...
ssl.keystore.location=...
ssl.keystore.password=...
ssl.key.password=...
```

With mutual TLS:

```text
Client verifies broker certificate
          +
Broker verifies client certificate
```

Authentication and authorization are distinct:

```text
TLS certificate proves identity
          |
          v
Kafka/security platform maps identity to principal
          |
          v
ACL/policy determines topic permissions
```

---

## 24. Spring Boot and @EnableKafka

In a normal Spring Boot application with Spring Kafka on the classpath and Kafka properties configured, Boot auto-configuration provides common Kafka infrastructure such as producer/consumer factories and `KafkaTemplate`.

For `@KafkaListener`, modern Spring Boot Kafka auto-configuration commonly provides the required infrastructure when the relevant auto-configuration is active; explicitly adding `@EnableKafka` is often unnecessary in a standard Boot setup.

Use explicit infrastructure configuration when you need to customize/replace the defaults.

Conceptually:

```text
application.yml
      |
      v
KafkaProperties
      |
      v
KafkaAutoConfiguration
      |
      +--> ProducerFactory
      +--> ConsumerFactory
      +--> KafkaTemplate
      +--> listener infrastructure/configuration
```

Avoid adding annotations simply out of habit; understand what Boot already auto-configures.

---

## 25. Kafka vs IBM MQ

Kafka and IBM MQ solve overlapping but different messaging problems.

### Kafka excels at

- event streaming,
- high-throughput append-only logs,
- replay,
- event sourcing,
- multiple independent consumer groups,
- stream processing,
- retaining event history.

### IBM MQ excels at

- traditional enterprise messaging,
- queue-centric point-to-point delivery,
- mature request/reply patterns,
- strong integration with legacy enterprise systems,
- workflows where “message consumed from a queue” is the central abstraction.

Mental model:

```text
IBM MQ

Producer ---> Queue ---> Consumer
                |
         message delivery
         is queue-centric
```

```text
Kafka

Producer ---> Durable Log
                |
        +-------+-------+
        |               |
   Consumer A       Consumer B
    group A          group B
```

Kafka consumers maintain their own logical positions in retained data, which makes replay and independent consumption natural.

---

## 26. Transactional Outbox Pattern

A critical problem:

```text
Database commit succeeds
Kafka send fails
```

Naive dual-write:

```text
Service
  |
  +--> UPDATE database  ---- success
  |
  +--> send Kafka event ---- failure

Result: inconsistent state
```

Transactional outbox:

```text
             ONE DB TRANSACTION
Service
  |
  +--> business table update
  |
  +--> outbox row insert
  |
  v
COMMIT
```

Then:

```text
Outbox table
     |
     +--> CDC (for example Debezium)
     |       OR
     +--> application publisher
             |
             v
           Kafka
```

The durable database row survives application crashes and Kafka downtime.

This solves a different problem from Kafka producer idempotence:

```text
Idempotent producer
    -> protects Kafka retries from duplicate Kafka appends

Transactional outbox
    -> protects DB + Kafka dual-write consistency
```

---

## 27. Configuration Cheat Sheet

### Reliable producer baseline

```properties
acks=all
enable.idempotence=true
```

Also understand:

```properties
retries
retry.backoff.ms
delivery.timeout.ms
request.timeout.ms
max.in.flight.requests.per.connection
batch.size
linger.ms
buffer.memory
max.block.ms
max.request.size
```

### Controlled consumer commits

```properties
enable.auto.commit=false
```

Then choose the Spring Kafka `AckMode` based on the business requirement.

### Consumer liveness

```properties
max.poll.records
max.poll.interval.ms
session.timeout.ms
heartbeat.interval.ms
```

### Replay starting point

```properties
auto.offset.reset=earliest
```

Remember: this applies only when no valid committed offset exists.

### Transactional consumers

```properties
isolation.level=read_committed
```

### SSL

```properties
security.protocol=SSL
ssl.truststore.location=...
ssl.keystore.location=...
```

---

## 28. Interview-Level Mental Models

### Batch vs record vs transaction

```text
Record
  = individual Kafka message
  = individual offset

Batch
  = producer/network efficiency unit
  = partition-specific
  = contains multiple records

Transaction
  = atomic Kafka visibility boundary
  = may contain multiple batches
  = may span topics and partitions
```

### Producer reliability ladder

```text
send()
  |
  v
acks
  |
  v
retries
  |
  v
idempotence
  |
  v
transactions
  |
  v
transactional outbox
```

Each solves a different problem.

### Consumer reliability ladder

```text
poll
 |
 v
deserialize
 |
 v
process
 |
 v
commit offset
```

Failures can happen between every step.

A robust design explicitly answers:

```text
What gets retried?
When is the offset committed?
Can processing be repeated safely?
Where do poison records go?
How is replay performed?
What happens during rebalance?
What happens if the app dies after the side effect but before commit?
```

### Most important distinction

Kafka's strongest guarantees exist **inside Kafka**.

```text
Kafka transaction
    |
    +--> Kafka record atomicity
    +--> Kafka offset atomicity
```

Once an external system is involved:

```text
Kafka ---> Database / REST API / Email / Filesystem
```

you need additional patterns such as:

- idempotent consumers,
- unique business keys,
- deduplication,
- transactional outbox/inbox,
- compensating operations.

---

## Final Summary

The most useful way to reason about Kafka is to keep four independent layers separate:

```text
1. Producer delivery
   batching -> retries -> idempotence -> transactions

2. Broker storage
   partitions -> offsets -> replication -> retention

3. Consumer progress
   poll -> process -> commit -> replay

4. Business consistency
   idempotency -> DLT -> outbox/inbox -> external side effects
```

When debugging a Kafka system, first identify **which layer owns the guarantee you need**. Many Kafka misunderstandings come from expecting a guarantee from one layer—such as batching, auto commit, idempotence, or transactions—to solve a problem that actually belongs to another.
