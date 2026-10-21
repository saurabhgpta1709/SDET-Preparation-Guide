# Apache Kafka for SDET Leads: Beginner-to-Advanced Learning & Interview Guide

**Audience:** SDET Lead / Senior QA Automation Engineer / SDET 4\
**Primary language:** Java\
**Focus:** Kafka fundamentals, event-driven architecture, reliable test
automation, distributed-system failure modes, observability,
performance, CI/CD, and leadership-level interview answers.

> **How to use this guide:** Read Parts 1--5 to build the mental model,
> implement the Java testing patterns in Parts 6--9, then practise the
> scenarios and interview answers in Parts 10--13. Kafka is not just a
> queue to assert against: it is a distributed log, and a strong SDET
> validates the complete business workflow, including persistence,
> retries, ordering, duplicates, replay, and recovery.

------------------------------------------------------------------------

## Table of contents

1.  [Kafka mental model](#1-kafka-mental-model)
2.  [Core architecture and
    terminology](#2-core-architecture-and-terminology)
3.  [Producer internals and delivery
    reliability](#3-producer-internals-and-delivery-reliability)
4.  [Consumer groups, offsets, and
    rebalancing](#4-consumer-groups-offsets-and-rebalancing)
5.  [Ordering, delivery semantics, and
    transactions](#5-ordering-delivery-semantics-and-transactions)
6.  [Java test architecture for
    Kafka](#6-java-test-architecture-for-kafka)
7.  [End-to-end asynchronous
    testing](#7-end-to-end-asynchronous-testing)
8.  [Failure handling: retries, DLQ, replay, and
    idempotency](#8-failure-handling-retries-dlq-replay-and-idempotency)
9.  [Schema, contracts, security, and test
    data](#9-schema-contracts-security-and-test-data)
10. [Performance, scalability, and
    observability](#10-performance-scalability-and-observability)
11. [CI/CD and environment strategy](#11-cicd-and-environment-strategy)
12. [SDET Lead scenario playbook](#12-sdet-lead-scenario-playbook)
13. [Interview questions with model
    answers](#13-interview-questions-with-model-answers)
14. [Hands-on lab plan](#14-hands-on-lab-plan)
15. [Cheat sheet and final
    checklist](#15-cheat-sheet-and-final-checklist)
16. [Glossary](#16-glossary)

------------------------------------------------------------------------

# 1. Kafka mental model

## 1.1 What is Apache Kafka?

Apache Kafka is a distributed event-streaming platform used to publish,
store, read, and process streams of events. It is designed to handle
high-volume data with durability, horizontal scalability, and fault
tolerance.

An **event** is a record that something happened. For example:

``` json
{
  "eventId": "evt-8f2a",
  "eventType": "BeneficiaryCreated",
  "customerId": "cust-1042",
  "correlationId": "req-7719",
  "occurredAt": "2026-10-21T10:15:30Z",
  "payload": {
    "accountStatus": "ACTIVE"
  }
}
```

Kafka can carry business events such as `PaymentInitiated`,
`PaymentCompleted`, `BeneficiaryCreated`, `OrderPlaced`, or technical
events such as audit and telemetry records.

### Why teams use Kafka

-   **Decoupling:** a producer does not need to call every downstream
    consumer directly.
-   **Buffering:** consumers can process events at their own pace,
    within retention and capacity limits.
-   **Replay:** retained events can be read again for recovery,
    reprocessing, or new consumers.
-   **Scale:** partitions allow work to be distributed across brokers
    and consumers.
-   **Durability:** replicated logs protect against certain broker
    failures.
-   **Multiple subscribers:** independent consumer groups can read the
    same topic for different purposes.

Kafka is not automatically a replacement for every message queue. It is
especially useful when a retained, replayable event log and
high-throughput stream processing are valuable.

## 1.2 Queue versus event log

In a traditional competing-consumer queue, a message is commonly removed
or acknowledged once processed. Kafka retains records according to topic
retention policy. A consumer tracks its own position (offset), so
another consumer group can read the same events independently.

Example:

-   `payment-service` publishes `PaymentCompleted`.
-   `notification-service` sends a notification.
-   `ledger-service` updates accounting records.
-   `analytics-service` builds reporting data.

Each service can have its own consumer group. The same event can be
consumed independently by all three groups.

## 1.3 Synchronous versus asynchronous workflow

**Synchronous:** service A calls service B and waits for a response. A
failure or latency in B can directly affect A's request.

**Asynchronous:** service A publishes an event and continues. Service B
processes it later. The request may return before every downstream
effect has completed.

For QA, this changes the assertion strategy. Do not assume that a
successful API response means all downstream records already exist.
Assert the immediate API contract first, then wait for the specific
asynchronous outcome with a bounded timeout.

## 1.4 Banking example: DBT onboarding

A representative onboarding workflow:

1.  A customer submits a DBT onboarding request through an API.
2.  The service validates the request and Aadhaar-related data through
    permitted integrations.
3.  It checks duplicate-account and beneficiary rules.
4.  It persists onboarding state.
5.  It publishes an event such as `BeneficiaryCreated`.
6.  A downstream consumer processes the event and updates a downstream
    store or workflow.
7.  The customer-facing status eventually becomes the expected state.

Potential defects include:

-   API says success but no event is published.
-   Event is published twice and creates duplicate downstream work.
-   Event contains the wrong customer or correlation ID.
-   Database transaction commits but event publication fails.
-   Event arrives but consumer fails to persist the result.
-   Retry creates an incorrect state transition.
-   Events for one customer are processed out of order.
-   A downstream dependency is slow and consumer lag grows.

An SDET Lead designs tests to detect these failures---not merely to
check that a record appears in a topic.

------------------------------------------------------------------------

# 2. Core architecture and terminology

## 2.1 The main components

-   **Producer:** writes records to Kafka.
-   **Consumer:** reads records from Kafka.
-   **Topic:** named logical stream of records.
-   **Partition:** ordered, append-only log within a topic.
-   **Offset:** position of a record within one partition.
-   **Broker:** Kafka server that stores partition replicas and serves
    clients.
-   **Cluster:** group of brokers coordinated as a Kafka deployment.
-   **Leader replica:** replica that handles reads/writes for a
    partition under the relevant configuration.
-   **Follower replica:** replica that fetches data from the leader.
-   **Replication factor:** number of replicas maintained for a
    partition.
-   **Consumer group:** consumers cooperating to divide partitions for a
    topic.
-   **Coordinator:** broker role responsible for group membership and
    offset/assignment coordination.
-   **Retention:** how long or how much data Kafka keeps.
-   **Compaction:** retention mode that keeps the latest value for each
    key, subject to configuration and tombstones.
-   **Schema Registry:** an optional service commonly used to manage
    Avro, Protobuf, or JSON Schema contracts.
-   **Kafka Connect:** framework for moving data between Kafka and
    external systems.
-   **Kafka Streams:** library for stream processing applications.

Kafka architecture evolves across versions and deployments. Modern Kafka
can use KRaft for metadata management rather than the older
ZooKeeper-based architecture. For interviews, understand the conceptual
responsibilities and know that operational details depend on the
deployed version.

## 2.2 Topics and partitions

A topic is divided into partitions. Each partition is an ordered
sequence of records. Ordering is guaranteed **within a partition**, not
across every partition in a topic.

Imagine `payment-events` has three partitions:

``` text
payment-events
  Partition 0: offset 0 -> offset 1 -> offset 2 -> ...
  Partition 1: offset 0 -> offset 1 -> offset 2 -> ...
  Partition 2: offset 0 -> offset 1 -> offset 2 -> ...
```

Partitions enable parallelism. A consumer group with three active
consumers can potentially process three partitions concurrently. A group
with five consumers and only three partitions will have at least two
consumers idle for that topic.

### Choosing a partition key

A key helps Kafka route related records to the same partition. If
per-account ordering matters, a key such as `accountId` may be
appropriate. If per-payment ordering matters, use `paymentId`. A null
key may be distributed according to the producer's partitioning
strategy, which may not preserve business-entity ordering.

A poor key can create a **hot partition**. For example, routing most
records to one popular merchant can overload one partition while others
are underutilized.

**SDET checks:** - Are records for the same entity assigned
consistently? - Does the partition key match the ordering requirement? -
Can tests identify the partition and offset for diagnosis? - Does
increasing partition count change key-to-partition mapping in a way the
business can tolerate?

## 2.3 Offsets

An offset identifies a record's position in a particular partition.
Offset `42` in partition 0 is unrelated to offset `42` in partition 1.

An offset is not a global event ID and should not be treated as one.
Offsets can have gaps from a consumer's perspective because of
compaction, transactions, or control records.

Consumers can commit offsets so that, after restart or rebalance, they
know where to resume. The exact position and replay behavior depend on
committed offsets, reset policy, isolation level, and application logic.

**Testing tip:** identify events by a unique business identifier
(`eventId`, `paymentId`, or `correlationId`) and use offsets as
diagnostic metadata---not as the only way to match a business event.

## 2.4 Replication, leaders, and in-sync replicas

A partition has a leader and replicas. Producers write to the leader.
Followers replicate data from the leader. The in-sync replica (ISR) set
contains replicas considered sufficiently caught up under Kafka's
replication rules.

Key settings to understand:

-   `replication.factor`: number of replicas per partition.
-   `min.insync.replicas`: minimum number of in-sync replicas required
    for writes when using `acks=all`.
-   Producer `acks`: acknowledgement policy.
-   `unclean.leader.election.enable`: whether an out-of-sync replica may
    be elected leader in scenarios where this can risk data loss.

A common durability setup is replication factor 3, `acks=all`, and
`min.insync.replicas=2`. This is an example, not a universal setting:
capacity, latency, availability, and failure objectives must be
considered together.

If only one ISR remains and `min.insync.replicas=2`, a producer using
`acks=all` can receive a not-enough-replicas error. That may be the
correct safety behavior rather than a defect.

## 2.5 Retention and compaction

Kafka does not necessarily delete a record after a consumer reads it.

-   **Time/size retention:** older log segments are removed based on
    configured policy.
-   **Log compaction:** Kafka retains the latest value per key over
    time, while tombstones signal deletion and are retained only for a
    configured period.
-   **Delete policy:** common default retention behavior.
-   **Compaction plus delete:** can be configured together.

QA implications: - A test must not assume a record remains forever. - A
new consumer group may replay retained history. - A compacted topic is
not a complete immutable audit trail of every intermediate value. -
Tombstones and cleanup timing can affect tests; avoid brittle
assumptions about exact deletion time.

## 2.6 Consumer groups and parallelism

Within a consumer group, a partition is assigned to one consumer at a
time for a given assignment. Different groups can consume the same topic
independently.

Example:

``` text
Topic: payment-events (4 partitions)

Group: ledger-service
  Consumer L1 -> P0, P1
  Consumer L2 -> P2, P3

Group: notification-service
  Consumer N1 -> P0, P1, P2, P3
```

Each group maintains its own progress. Adding consumers does not
automatically increase throughput if the topic has too few partitions or
the bottleneck is elsewhere.

## 2.7 Essential commands

Commands vary by Kafka version and installation. Verify syntax for your
environment.

``` bash
# List topics
kafka-topics.sh --bootstrap-server localhost:9092 --list

# Describe a topic, including partitions and replicas
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --topic payment-events

# Create a topic for a local lab
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic payment-events --partitions 3 --replication-factor 1

# Inspect consumer groups
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# Describe group members, assignments, offsets, and lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group ledger-service

# Read a topic from its beginning (lab use)
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic payment-events --from-beginning
```

A replication factor of 1 is suitable only for a single-broker local
lab, not a production durability recommendation.

------------------------------------------------------------------------

# 3. Producer internals and delivery reliability

## 3.1 Producer send path

At a high level:

1.  Application creates a record with topic, key, value, and optional
    headers.
2.  Serializer converts key and value to bytes.
3.  Partitioner selects a partition.
4.  Producer accumulates records in batches.
5.  Sender thread transmits batches to the relevant broker.
6.  Broker appends records and responds according to the acknowledgement
    policy.
7.  Callback or future reports success or failure.

A producer may batch records for throughput. Compression can reduce
network and storage cost. These optimizations can trade some latency for
throughput.

## 3.2 Important producer settings

  -------------------------------------------------------------------------------------------
  Setting                                   Meaning                  QA / interview relevance
  ----------------------------------------- ------------------------ ------------------------
  `bootstrap.servers`                       Initial broker addresses Connectivity and
                                                                     environment
                                                                     configuration

  `key.serializer` / `value.serializer`     Convert objects to bytes Serialization contract
                                                                     failures

  `acks`                                    Required acknowledgement Durability/latency
                                            level                    trade-off

  `enable.idempotence`                      Prevents duplicate       Not the same as
                                            writes caused by         business-level
                                            producer retries within  exactly-once processing
                                            supported producer       
                                            semantics                

  `retries`                                 Retry transient send     Verify retry metrics and
                                            failures, subject to     final outcome
                                            producer                 
                                            behavior/configuration   

  `delivery.timeout.ms`                     Upper bound for a send   Timeout and
                                            to complete, including   error-handling tests
                                            retries                  

  `request.timeout.ms`                      Request response timeout Network and broker delay
                                                                     behavior

  `linger.ms`                               Briefly waits to form    Latency/throughput
                                            larger batches           tuning

  `batch.size`                              Target batch size        Throughput and memory
                                                                     trade-offs

  `compression.type`                        Compression algorithm    CPU/network/throughput
                                                                     trade-offs

  `max.in.flight.requests.per.connection`   Outstanding requests per Ordering behavior with
                                            connection               retries depends on
                                                                     configuration and
                                                                     idempotence
  -------------------------------------------------------------------------------------------

Do not memorise settings in isolation. Explain the failure mode or
trade-off each setting controls.

## 3.3 Acknowledgements: `acks=0`, `acks=1`, `acks=all`

-   `acks=0`: producer does not wait for broker acknowledgement. Low
    latency, weaker delivery confirmation.
-   `acks=1`: leader acknowledges after its local append. A leader
    failure before replication can risk loss.
-   `acks=all` (also written `-1`): waits for acknowledgement from the
    current ISR according to Kafka's rules. Durability still depends on
    replication and `min.insync.replicas`.

For a critical financial event, discuss `acks=all`, replication, minimum
ISR, idempotence, monitoring, and business reconciliation---not just one
producer property.

## 3.4 Producer idempotence versus business idempotency

Producer idempotence prevents duplicate log entries that can result from
producer retries within Kafka's supported producer protocol. It does
**not** guarantee that a downstream business action happens once.

Example: - The consumer charges an external service. - The charge
succeeds. - The consumer crashes before committing its offset. - Kafka
delivers the event again. - The external charge may happen twice unless
the consumer uses a business idempotency mechanism.

Business idempotency can use an event ID, transaction ID, deduplication
table, unique constraint, idempotency key at the downstream API, or an
atomic state transition.

## 3.5 Producer failure scenarios

Test at least these classes:

1.  Broker unavailable before send.
2.  Broker becomes unavailable during send.
3.  Request times out but the broker may have appended the record.
4.  Serialization fails before the broker receives the record.
5.  Topic does not exist or authorization denies write.
6.  Message exceeds allowed size.
7.  Insufficient replicas for required acknowledgements.
8.  Producer is restarted and application retries.
9.  Producer emits the same business event twice.
10. Schema version is incompatible with the consumer.

The hardest case is an ambiguous outcome: the producer times out, but
the broker may have written the record. The application must not assume
"timeout means definitely not published." Validate business-level
deduplication and reconciliation.

------------------------------------------------------------------------

# 4. Consumer groups, offsets, and rebalancing

## 4.1 Consumer processing loop

A typical consumer workflow is:

1.  Poll records.
2.  Validate and process each record.
3.  Persist the business outcome.
4.  Commit offsets at the correct point.
5.  Continue polling.

The order of processing and committing is critical. If an offset is
committed before the business operation is safely completed, a crash can
lose processing. If processing succeeds but the offset is not committed,
a crash can lead to duplicate processing.

## 4.2 Auto-commit versus manual commit

-   **Auto-commit:** offsets are committed periodically by client
    behavior. Simpler, but can be unsafe if application processing is
    not aligned with commits.
-   **Manual commit:** application controls when offsets are committed.
    It can align progress with completed work, but must be implemented
    carefully.

A common safe principle is: do not commit progress beyond work that has
been durably handled. With parallel processing, per-partition ordering
and contiguous completed offsets require particular care.

For an SDET, ask: - At what point is the offset committed? - What
happens if the service crashes between database commit and offset
commit? - Can replay cause duplicate side effects? - Is the side effect
idempotent? - How are failed records isolated without blocking a
partition indefinitely?

## 4.3 Rebalancing

A rebalance occurs when partition ownership changes, for example when
consumers join, leave, fail, or subscriptions change. During a
rebalance, processing can pause and records may be reprocessed depending
on commit timing.

Common causes of frequent rebalances: - Consumers crash or restart. -
Processing takes too long between polls. - `max.poll.interval.ms` is
exceeded. - Deployment rolls out many instances simultaneously. -
Network instability or heartbeat failures. - Consumer group membership
changes frequently.

QA tests should verify that rebalances do not cause data loss,
unacceptable duplicate side effects, stuck partitions, or unacceptable
recovery time.

## 4.4 Polling and long processing

A consumer must continue polling within the configured limits. If
application processing takes too long, it can be removed from the group.
Possible approaches include: - Reduce per-record processing time. - Tune
poll limits and intervals based on measured behavior. - Hand off work to
a controlled worker pool while preserving partition ordering and safe
offset management. - Use pause/resume appropriately. - Avoid unbounded
queues that simply move the backlog into application memory.

Never tune `max.poll.interval.ms` blindly. First measure processing
time, batch size, rebalance frequency, and lag.

## 4.5 Consumer lag

Lag is commonly described as the difference between the latest available
offset and the consumer group's committed/progress offset. Exact
monitoring definitions can vary.

Lag is a symptom, not a root cause. It can grow because: - Producer rate
exceeds consumer processing capacity. - A downstream API or database is
slow. - A partition is hot. - Consumer instances are unhealthy. -
Rebalances repeatedly interrupt work. - Errors and retries slow
processing. - Consumers are not running or cannot authenticate.

Investigate per-partition lag, processing latency, incoming rate,
outgoing rate, error rate, rebalance count, CPU, memory, database
latency, and downstream dependency health.

------------------------------------------------------------------------

# 5. Ordering, delivery semantics, and transactions

## 5.1 Ordering guarantees

Kafka preserves record order within a partition. It does not provide a
total order across partitions.

If the business requires a sequence such as:

`PaymentCreated -> PaymentValidated -> PaymentSettled`

then design must ensure relevant events use a stable key and the
consumer checks valid state transitions. A key alone is not sufficient
if different services publish events independently or if the workflow
spans multiple topics and partitions.

Test: - same entity uses the expected key; - state transitions reject or
tolerate out-of-order events according to the contract; - retries do not
reorder business effects unexpectedly; - partition count changes are
understood; - parallel consumer processing does not break per-entity
ordering.

## 5.2 At-most-once, at-least-once, and exactly-once

**At-most-once:** a record is processed zero or one times; loss is
possible.

**At-least-once:** a record is eventually processed one or more times;
duplicates are possible.

**Exactly-once:** requires a precisely scoped guarantee. Kafka
transactions can support atomic writes across Kafka partitions and
offset commits for consume-transform-produce pipelines. Exactly-once
effects in an external database, payment gateway, or HTTP API require
additional coordination or idempotency; Kafka cannot magically make
arbitrary external side effects exactly once.

Interview-ready wording:

> "I avoid claiming end-to-end exactly-once by default. Kafka can
> provide exactly-once processing for specific Kafka-to-Kafka workflows
> when configured correctly. For external side effects, I design
> idempotent consumers, durable deduplication, transactional boundaries
> where available, and reconciliation."

## 5.3 Kafka transactions

Transactions can atomically publish records to multiple partitions and
commit consumed offsets as part of a Kafka transaction. Consumers that
need to hide aborted transactional records use the appropriate isolation
level, commonly `read_committed`.

Tests should cover: - committed transaction records are visible to
`read_committed` consumers; - aborted transaction records are not
treated as committed business events; - application recovery after
transaction failure; - transactional ID uniqueness and fencing behavior
where relevant; - downstream business effects remain correct during
retries.

## 5.4 The dual-write problem and transactional outbox

Suppose a service writes a database row and publishes a Kafka event as
two independent actions:

``` text
1. Save payment as COMPLETED in DB
2. Publish PaymentCompleted to Kafka
```

If the DB write succeeds and publication fails, database state and event
stream diverge. Reversing the order creates the opposite risk.

A **transactional outbox** pattern writes the business change and an
outbox record in the same database transaction. A relay/CDC process
later publishes the outbox record to Kafka. Consumers still need
idempotency because relay retries can publish duplicates.

SDET validation: - business row and outbox row are atomic; - failed
publication is retried; - published event contains the correct event ID
and payload; - duplicates are safe; - outbox backlog and age are
observable; - poison records do not silently disappear; - reconciliation
identifies records that are stuck.

------------------------------------------------------------------------

# 6. Java test architecture for Kafka

## 6.1 Recommended architecture

Keep test cases separate from Kafka client plumbing.

``` text
Test Layer
  ├── PaymentFlowTest
  ├── BeneficiaryOnboardingTest
  └── EventContractTest
          |
          v
Domain Validation Layer
  ├── PaymentAssertions
  ├── BeneficiaryAssertions
  └── EventAssertions
          |
          v
Kafka Test Support
  ├── KafkaConsumerManager
  ├── KafkaEventWaiter
  ├── KafkaRecordMatcher
  └── KafkaTestConfig
          |
          v
Kafka client + test environment
```

Recommended principles: - Reusable consumer utilities, not one ad hoc
consumer per test. - Unique correlation IDs and test data. - Bounded
waits; never sleep indefinitely. - Explicit cleanup and clear ownership
of consumers. - Environment configuration externalised. - Safe handling
of credentials. - Diagnostic output includes topic, partition, offset,
key, event ID, correlation ID, timestamp, and elapsed time. - Parallel
tests must not compete for the same records or consumer group.

## 6.2 Maven dependency

Use a Kafka client version compatible with your broker and
organisation's approved dependency policy. The following is
illustrative; pin a currently supported version in your project.

``` xml
<dependency>
  <groupId>org.apache.kafka</groupId>
  <artifactId>kafka-clients</artifactId>
  <version>${kafka.clients.version}</version>
  <scope>test</scope>
</dependency>
```

For a test suite, you may also use JUnit or TestNG, Jackson for JSON,
AssertJ/Hamcrest, and Testcontainers for isolated integration tests.
Keep test and production dependencies scoped intentionally.

## 6.3 Consumer configuration example

``` java
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
props.put(ConsumerConfig.GROUP_ID_CONFIG, "qa-" + uniqueRunId);
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
          StringDeserializer.class.getName());
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
          StringDeserializer.class.getName());
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
```

Important: - A unique group ID prevents unrelated test runs from sharing
committed offsets, but does not by itself isolate topic data. -
`auto.offset.reset=earliest` applies when there is no valid committed
offset for the group; it does not rewind an existing group
automatically. - `read_committed` is useful when validating
transactional topics, but choose the isolation level to match the system
contract. - Production security properties (TLS/SASL) must be supplied
securely when required.

## 6.4 Correlation IDs and event matching

Never assert "some event appeared" when the topic is shared by multiple
tests or environments. Match the event to the operation you initiated.

Recommended identifiers: - `correlationId`: ties service logs and events
to one request/workflow. - `eventId`: unique identity of an event. -
`paymentId` / `beneficiaryId`: business entity. - `idempotencyKey`:
identifies a retried client operation when supported.

Include correlation ID in request headers or payload where the
application contract supports it. Propagate it through logs and event
headers. Do not invent or assume a header name; confirm the service
contract.

## 6.5 A bounded event waiter

The following is a simplified pattern. Production-quality helpers should
also handle deserialization errors, authentication, shutdown, metrics,
and partition assignment carefully.

``` java
public Optional<ConsumerRecord<String, String>> awaitRecord(
        KafkaConsumer<String, String> consumer,
        Predicate<ConsumerRecord<String, String>> matcher,
        Duration timeout) {

    long deadline = System.nanoTime() + timeout.toNanos();

    while (System.nanoTime() < deadline) {
        Duration remaining = Duration.ofNanos(deadline - System.nanoTime());
        Duration pollDuration = remaining.compareTo(Duration.ofMillis(500)) > 0
                ? Duration.ofMillis(500)
                : remaining;

        ConsumerRecords<String, String> records = consumer.poll(pollDuration);

        for (ConsumerRecord<String, String> record : records) {
            if (matcher.test(record)) {
                return Optional.of(record);
            }
        }
    }

    return Optional.empty();
}
```

Cautions: - The consumer must be subscribed and assigned correctly
before waiting. - If the consumer starts after the event and uses a new
group with `latest`, it may miss the event. For deterministic tests,
establish the observation strategy before triggering the action. -
Polling a shared topic and discarding nonmatching records is acceptable
only when losing those records from this test consumer's view is
harmless. Use a unique test group or dedicated test topic when isolation
is required. - Use `System.nanoTime()` for elapsed-time deadlines. -
Return useful diagnostics on timeout.

## 6.6 Consumer lifecycle

Always close consumers in `finally` or try-with-resources where
appropriate. A consumer is not generally thread-safe: do not share one
KafkaConsumer instance across arbitrary parallel test threads.

For parallel TestNG execution: - use one consumer per test/thread or a
dedicated consumer service with a carefully designed concurrency
model; - use unique group IDs when test independence requires them; -
use unique business identifiers; - avoid concurrent consumers stealing
records from each other under the same group; - cap concurrency based on
broker, database, API, and test environment capacity.

------------------------------------------------------------------------

# 7. End-to-end asynchronous testing

## 7.1 The API → Kafka → DB → downstream pattern

A robust test verifies the business journey, not only a message in
isolation.

Example: payment API creates a payment.

1.  Generate a unique test payment and correlation ID.
2.  Call the API and assert status code, response schema, and payment
    ID.
3.  Validate immediate database state if the contract says it is
    synchronous.
4.  Await the expected Kafka event using a bounded waiter.
5.  Assert event type, event ID, key, correlation ID, amount, currency,
    and business entity.
6.  Poll the downstream database or API until the expected state
    appears.
7.  Assert invariants, such as exactly one logical payment and no
    duplicate ledger entry.
8.  On failure, capture response, event metadata, DB state, service
    logs, and correlation ID.

Do not assume all checks are synchronous. Define expected timing and
consistency boundaries per workflow.

## 7.2 Eventual consistency

Eventual consistency means replicas or downstream projections may
temporarily disagree but are expected to converge if processing succeeds
and no new changes intervene.

Bad test:

``` java
Thread.sleep(5000);
assertEquals("COMPLETED", database.readStatus(paymentId));
```

Why it is weak: - It may waste time when the system is fast. - It may
still fail when the system is slower. - It provides little diagnosis. -
It hides the actual wait budget.

Better: poll a condition until a deadline, with a short interval, and
include the last observed value in the failure message.

``` java
public <T> T awaitCondition(
        Supplier<T> read,
        Predicate<T> success,
        Duration timeout,
        Duration interval) throws InterruptedException {

    long deadline = System.nanoTime() + timeout.toNanos();
    T last = null;

    while (System.nanoTime() < deadline) {
        last = read.get();
        if (success.test(last)) {
            return last;
        }
        long remainingMs =
                Math.max(1, Duration.ofNanos(deadline - System.nanoTime()).toMillis());
        Thread.sleep(Math.min(interval.toMillis(), remainingMs));
    }

    throw new AssertionError("Condition not met within " + timeout
            + "; last observed value: " + last);
}
```

This helper is illustrative. In a production framework, use a
well-tested polling library or add interruption handling, logging, and
jitter where appropriate.

## 7.3 Negative assertions

Negative assertions are harder in asynchronous systems. "No event
exists" cannot be proven by checking for a few milliseconds unless the
contract defines a bounded time window and all relevant event paths are
observable.

Better strategies: - assert no event for a documented observation
window; - validate no database side effect and no downstream state
change; - check audit/reconciliation records; - use a deterministic
event trigger and correlation ID; - distinguish "no event observed" from
"event definitely never published."

For invalid requests, assert the documented rejection response and
confirm forbidden side effects do not occur.

## 7.4 Test isolation strategies

Preferred options, depending on environment: 1. Unique entity IDs and
correlation IDs. 2. Dedicated topic or isolated namespace for tests. 3.
Unique consumer groups and explicit offset strategy. 4. Test-specific
headers/metadata supported by the application. 5. Cleanup or expiry
strategy that does not mutate shared production-like data unsafely.

Avoid relying on a shared topic being empty. Avoid deleting shared
topics as routine test cleanup in a common environment.

## 7.5 Contract assertions

Validate both structure and business meaning: - required fields exist
and are non-null; - types and enums are valid; - event name/version is
correct; - timestamp format and timezone are correct; - IDs correlate
with the API response; - amounts use the expected precision and
currency; - schema compatibility is preserved; - sensitive information
is not accidentally emitted; - headers and partition key meet the
contract.

------------------------------------------------------------------------

# 8. Failure handling: retries, DLQ, replay, and idempotency

## 8.1 Retry types

-   **Producer retry:** retrying delivery to Kafka after transient
    failures.
-   **Consumer retry:** retrying processing of a record that failed.
-   **Application/API retry:** caller retries a request after timeout or
    transient response.
-   **Infrastructure retry:** connector or relay retries a failed
    operation.

These are different mechanisms and can compound. If multiple layers each
retry aggressively, load can multiply and a partial outage can become
worse.

## 8.2 Retry design

A sound retry policy considers: - transient versus permanent errors; -
maximum attempts or elapsed time; - exponential backoff and jitter; -
idempotency; - ordering requirements; - retry metrics and alerting; -
whether a record blocks later records; - poison-message handling; -
ownership of final failure.

Do not retry invalid schema, malformed data, or permanent business
validation errors indefinitely. Route or reject them according to an
explicit policy.

## 8.3 Dead-letter topic (DLQ)

A DLQ stores records that cannot be processed under the configured
failure policy. Naming and behavior are implementation-specific.

A useful DLQ record should preserve: - original key and payload (subject
to data policy); - original topic, partition, and offset; - event ID and
correlation ID; - failure category and safe error detail; - attempt
count and timestamps; - consumer/service version.

SDET checks: - only eligible failures go to DLQ; - transient failures
receive expected retries; - permanent failures are routed as designed; -
DLQ publishing failure is handled; - sensitive data is not leaked into
logs or DLQ; - replay is controlled, auditable, and safe; - DLQ growth
triggers monitoring.

A DLQ is not a garbage bin. It is an operational workflow requiring
ownership, alerting, investigation, and replay governance.

## 8.4 Idempotent consumer testing

Create the same logical event more than once. Verify: - one logical
business effect exists; - duplicate delivery is measurable or safely
ignored; - the deduplication key is stable; - concurrent duplicate
processing does not race; - deduplication retention covers the realistic
replay window; - a crash between side effect and offset commit does not
create duplicate financial effects.

For payments or ledger events, use database constraints and transaction
boundaries as appropriate. An in-memory set is not sufficient for a
horizontally scaled service or restart recovery.

## 8.5 Replay and recovery

Replay may be needed after a bug fix, consumer outage, or new projection
deployment. It can also repeat external side effects if safeguards are
weak.

Test: - reset/replay procedure is documented and access-controlled; -
replay range is correct; - downstream idempotency is preserved; -
ordering and state transitions remain valid; - replay rate does not
overload dependencies; - duplicate events are observable; -
reconciliation confirms expected counts and balances; - recovery time
meets the operational objective.

Never reset offsets in a shared or production environment casually.
Treat replay as a controlled operational action.

## 8.6 Poison messages

A poison message consistently fails due to malformed data, unsupported
schema, or a deterministic processing error.

Potential strategies: - bounded retries, then DLQ; - validation before
side effects; - quarantine with metadata; - schema evolution or consumer
fix followed by controlled replay.

Do not allow one poison message to block a partition forever unless
strict ordering and business policy explicitly require stopping the
stream and alerting.

------------------------------------------------------------------------

# 9. Schema, contracts, security, and test data

## 9.1 Schema evolution

A producer and consumer may deploy at different times. Schema
compatibility determines whether older and newer versions can
interoperate.

Common compatibility concepts: - **Backward compatibility:** new
consumers can read old data. - **Forward compatibility:** old consumers
can read new data. - **Full compatibility:** both directions, subject to
the registry's exact rules. - **Transitive compatibility:** checks
compatibility against more than the immediately previous schema version.

Compatibility rules differ by format and registry configuration.
Optional fields, defaults, enum changes, renamed fields, and type
changes deserve particular scrutiny.

SDET tests: - old consumer with new producer schema; - new consumer with
historical messages; - missing optional fields and defaults; - unknown
enum values; - malformed payloads; - schema ID/subject/version
correctness; - rolling deployment where versions overlap.

## 9.2 Contract testing

An event contract includes: - topic and event type; - key and
partitioning expectations; - headers; - schema; - field semantics and
requiredness; - producer/consumer ownership; - ordering assumptions; -
retry and error policy; - versioning and compatibility policy; - privacy
classification.

Contract tests catch drift earlier than end-to-end tests alone. Pair
them with a smaller number of true end-to-end workflows.

## 9.3 Security

Kafka security may use TLS encryption, SASL authentication, and ACL
authorization. The exact mechanism depends on the deployment.

Validate: - authorised producer can publish; - unauthorised producer is
denied; - authorised consumer can read only permitted topics; -
unauthorised consumer cannot read protected topics; - credentials and
tokens are not printed in logs; - TLS trust/certificate failures are
handled clearly; - secret rotation works; - test accounts have least
privilege.

Do not place production credentials in source control or plain-text test
reports. Avoid copying real customer PII into test payloads.

## 9.4 Test data design

Use deterministic, unique data: - `runId`: identifies a CI execution; -
`testCaseId`: identifies a test; - `correlationId`: identifies the
workflow; - `entityId`: identifies the business object.

A possible value format is `qa-{runId}-{testCaseId}-{randomSuffix}`.
Keep lengths and character sets compatible with application constraints.

Data rules: - use synthetic or masked data; - avoid shared hard-coded
customer identifiers; - make cleanup safe and idempotent; - make retries
generate a stable idempotency key when testing client retry behavior; -
keep event payloads small enough for operational constraints; - ensure
parallel workers cannot accidentally mutate the same entity.

------------------------------------------------------------------------

# 10. Performance, scalability, and observability

## 10.1 Kafka performance dimensions

Measure more than messages per second: - producer throughput and send
latency (p50/p95/p99); - consumer throughput and processing latency; -
end-to-end event-to-state latency; - consumer lag per group and
partition; - error and retry rates; - rebalance count and duration; -
broker disk, network, CPU, request latency; - under-replicated
partitions and ISR changes; - database/API dependency latency; - DLQ
rate and age; - outbox backlog, if used.

## 10.2 Load-test design

Define: - workload mix and event size; - target throughput and burst
profile; - partition count and key distribution; - number of producers
and consumers; - retention and replication configuration; - realistic
downstream latency; - test duration and warm-up; - acceptable p95/p99
latency and lag; - data correctness checks during load; - recovery
behavior after load stops.

Run a baseline, ramp gradually, sustain load, and then observe
drain/recovery. A system can meet throughput targets while building
unbounded lag, so measure whether consumers catch up.

## 10.3 Backpressure

Backpressure means a downstream stage cannot keep up with incoming work.
Kafka buffers the difference until capacity or retention constraints
become relevant, but it does not eliminate the bottleneck.

Symptoms: - lag increases continuously; - consumer processing time
rises; - downstream API throttles; - DB connection pool saturates; -
retry volume increases; - memory grows in application queues.

Possible responses: - fix slow dependencies or queries; - scale
consumers if partitions permit; - increase partitions only after
evaluating key distribution and ordering; - batch writes where safe; -
apply rate limiting; - use bounded queues and controlled backpressure; -
isolate slow workflows; - tune retries to avoid retry storms.

## 10.4 Capacity and partition planning

More partitions can improve parallelism but increase operational
overhead and may affect ordering assumptions. The maximum useful
consumer parallelism for a group is constrained by partition count for
that topic, while real throughput also depends on CPU, I/O, broker
capacity, and downstream systems.

Avoid simple claims such as "more partitions always means more
throughput." Benchmark with representative keys, payloads, replication,
and downstream behavior.

## 10.5 Observability and debugging

A production-grade workflow should propagate a correlation ID across: -
API gateway; - service logs; - database/outbox record where
appropriate; - Kafka headers/payload; - consumer logs; - downstream
API; - final persisted state.

Useful debugging sequence for "API succeeded but downstream state is
missing":

1.  Find the request using correlation ID.
2.  Confirm API response and business transaction state.
3.  Check outbox record, if the architecture uses one.
4.  Search Kafka for the event ID/key/correlation ID and inspect
    metadata.
5.  Check producer errors and broker/ACL/schema failures.
6.  Check consumer group status and per-partition lag.
7.  Check consumer exceptions, retry/DLQ records, and rebalance history.
8.  Check downstream DB/API health and transaction outcome.
9.  Reconcile event count with business-state count.
10. Capture a minimal reproducible case and identify the owning service.

Do not use message payloads with sensitive customer data in logs. Log
safe identifiers and redacted metadata.

------------------------------------------------------------------------

# 11. CI/CD and environment strategy

## 11.1 Testing pyramid for Kafka workflows

A practical mix:

1.  **Unit tests:** serialization, validation, state transitions, retry
    policy.
2.  **Component tests:** producer or consumer against controlled
    dependencies.
3.  **Contract tests:** schema and producer-consumer compatibility.
4.  **Integration tests:** real Kafka broker, often using Testcontainers
    or a dedicated test cluster.
5.  **End-to-end tests:** API → Kafka → downstream state.
6.  **Resilience tests:** broker/consumer/dependency failures, replay,
    lag recovery.
7.  **Performance tests:** sustained throughput, latency, and backlog
    drain.

Do not run every end-to-end workflow on every commit if the environment
is expensive and flaky. Choose test layers according to risk and
feedback speed.

## 11.2 Local integration versus shared environment

**Testcontainers/local broker** - reproducible and isolated; - useful
for serialization, partitioning, consumer logic, and integration
tests; - may not represent production security, cluster size,
networking, or operational behavior.

**Dedicated integration cluster** - closer to deployed security and
topology; - needs test-data isolation and resource coordination; -
shared capacity and test interference must be managed.

Use both where justified. Do not claim a single-broker local test
validates multi-broker failover.

## 11.3 Jenkins pipeline example

``` text
Checkout
  -> Compile + unit tests
  -> Schema/contract tests
  -> Start test dependencies / verify Kafka connectivity
  -> API + Kafka + DB integration tests
  -> Publish reports and diagnostics
  -> Smoke suite after deployment
  -> Scheduled resilience/performance suite
  -> Quality gate / release decision
```

Quality gates can include: - critical contract tests pass; - no
unexpected DLQ increase; - consumer lag drains within the target; -
end-to-end latency is within threshold; - no duplicate business
effects; - test environment cleanup succeeded.

Make reports actionable: test name, correlation ID, event metadata,
expected/actual state, elapsed time, last observed value, and links to
logs.

## 11.4 Flaky test controls

Common causes: - fixed sleeps; - wrong offset reset assumptions; -
shared consumer group; - event matching too broadly; - concurrent tests
reusing IDs; - tests starting consumers too late; - test data cleanup
racing with assertions; - overloaded shared broker; - clock/timezone
assumptions.

Fix root causes before adding retries. A retry can reduce transient
noise but must not hide real defects. Record first-attempt failures and
classify flaky tests separately from product failures.

------------------------------------------------------------------------

# 12. SDET Lead scenario playbook

For every scenario, structure the answer as: **risk → design →
assertions → observability → failure/recovery → exit criteria**.

## Scenario 1: API returns 200 but no Kafka event appears

**Risks:** event publication failure, outbox relay delay, producer
ACL/schema error, wrong topic, consumer observation mistake.

**Test design:** - capture request and correlation ID; - validate the
API response and DB/outbox state; - observe the correct topic before or
at the time of triggering the request; - wait for a bounded SLA; -
search by event ID/correlation ID; - inspect producer/relay errors and
consumer lag if the event exists but downstream state is missing.

**Exit criteria:** expected event is published within the documented SLA
and the business state converges. A timeout report should show the last
observed state and diagnostics.

## Scenario 2: Duplicate payment event

Publish or replay the same logical event twice. Verify one logical
ledger effect, stable deduplication behavior, correct metrics, and no
duplicate notification/charge. Also test concurrent duplicate delivery
and crash/restart around the side effect.

## Scenario 3: Events are processed out of order

Determine the actual ordering contract. Verify partition key, consumer
concurrency, retry behavior, and state-transition rules. Do not assume a
topic provides global ordering. If cross-entity order is irrelevant,
tests should not enforce unnecessary global sequencing.

## Scenario 4: Consumer lag increases after deployment

Compare pre/post deployment processing latency, throughput, error rate,
rebalance frequency, resource usage, partition assignments, and
downstream dependency latency. Determine whether the deployment reduced
capacity or introduced a retry loop. Verify lag stabilises and drains
after rollback/fix.

## Scenario 5: One consumer crashes mid-processing

Reproduce a crash between side effect and offset commit. Verify
redelivery behavior and idempotency. Check that the group rebalances,
the partition is reassigned, and processing resumes within the recovery
objective.

## Scenario 6: Database is down

Check bounded retries/backoff, pause/backpressure behavior, lag growth,
memory safety, and recovery. After DB recovery, verify events are
processed, no logical records are missing or duplicated, and lag returns
to baseline.

## Scenario 7: Schema change breaks an older consumer

Run compatibility checks before deployment. Test mixed-version
producer/consumer combinations and historical messages. Confirm rollout
order and rollback compatibility. Unknown fields or enum values should
be handled according to contract.

## Scenario 8: Tests pass sequentially but fail in parallel

Investigate shared IDs, shared mutable state, consumer groups, offset
commits, topic cleanup, DB rows, rate limits, and environment
saturation. Add run-specific identifiers and controlled concurrency. Do
not solve this only by disabling parallel execution unless
capacity/isolation constraints justify it.

## Scenario 9: Kafka is healthy but the business outcome is wrong

A record in Kafka is not proof of successful business processing. Follow
the correlation ID through consumer logs, retries, downstream API/DB,
and reconciliation. Assert business invariants, not merely transport
delivery.

## Scenario 10: How do you decide release readiness?

Combine: - contract compatibility; - critical API-to-event workflows; -
duplicate/idempotency and retry behavior; - lag and recovery checks; -
DLQ/retry thresholds; - observability and alert coverage; - known risks
and rollback plan; - defect severity and residual risk.

A lead communicates the evidence, risk, and decision---not just a green
test count.

------------------------------------------------------------------------

# 13. Interview questions with model answers

## Fundamentals

### 1. What is Kafka?

Kafka is a distributed event-streaming platform. Producers write records
to partitioned topics; brokers store replicated logs; consumer groups
read independently and track offsets. Its retention and replay model
makes it useful for high-throughput asynchronous workflows and
event-driven microservices.

### 2. Is Kafka a queue or a pub/sub system?

It can support both usage patterns. Within one consumer group, consumers
share partitions like competing consumers. Different groups can
independently read the same topic, which resembles publish/subscribe.
The key distinction is that Kafka retains records and consumers track
offsets rather than each message being universally removed when one
consumer reads it.

### 3. What is a partition and why does it matter?

A partition is an ordered log and the basic unit of parallelism and
replication. More partitions can allow more consumer parallelism, but
ordering is only guaranteed within a partition. Partition key choice
affects ordering and load distribution.

### 4. What is an offset?

An offset is a record position within one partition. It supports
consumer progress and replay. It is not a globally unique event ID, so
tests should use business identifiers to correlate events.

### 5. What is a consumer group?

A consumer group is a set of consumers that cooperatively consume topic
partitions. A partition is assigned to one consumer in the group at a
time, while other groups can independently read the same records.

### 6. What is consumer lag?

Lag estimates how far a group is behind the latest available records. It
can indicate insufficient processing capacity, a slow dependency,
repeated rebalances, or failures. I investigate lag per partition
alongside processing latency, error rate, throughput, and dependency
metrics.

## Reliability and internals

### 7. Explain `acks=all`.

The producer waits for acknowledgement according to Kafka's all-ISR
semantics. Durability also depends on replication factor and
`min.insync.replicas`. It is stronger than leader-only acknowledgement
but can reject writes when the cluster cannot meet the configured
durability requirement.

### 8. Does Kafka guarantee exactly-once delivery?

Not as an unrestricted end-to-end promise. Kafka transactions can
provide atomic Kafka-to-Kafka processing when correctly configured.
External database or HTTP side effects still require idempotency,
durable deduplication, or a suitable transactional integration.

### 9. What happens if a consumer crashes after processing but before committing the offset?

The record may be delivered again. If the business side effect already
happened, it can be duplicated unless the consumer is idempotent. I test
this failure window explicitly and verify deduplication survives
restarts and concurrent delivery.

### 10. What happens if the offset is committed before processing finishes?

A crash can cause the record to be skipped on restart even though the
business operation never completed. The commit point must align with
durable processing; the exact strategy depends on processing
architecture and ordering requirements.

### 11. How does Kafka preserve order?

Kafka preserves append order within a partition. A stable key can route
related events to the same partition, but it does not create global
ordering across partitions or independent services. Consumer concurrency
and retries must also preserve business-level order.

### 12. What causes rebalancing?

Consumer joins/leaves, failures, subscription changes, heartbeat
problems, or exceeding poll-processing limits can cause rebalances.
Frequent rebalances interrupt progress and can increase duplicates and
lag. I inspect group events, poll duration, deployment churn, and
dependency latency.

### 13. What is the transactional outbox pattern?

The application writes the business state and an outbox event in one
database transaction. A relay or CDC process publishes the outbox record
to Kafka later. It avoids the dual-write inconsistency, but publication
can be repeated, so consumers still need idempotency.

### 14. What is a DLQ?

A dead-letter topic holds records that fail processing under a defined
policy. It should retain safe diagnostic metadata and be monitored,
triaged, and replayed through a controlled process. A DLQ without
ownership and alerting simply hides failures.

## SDET design

### 15. How would you test an API that publishes to Kafka?

I generate unique test data and a correlation ID, call the API, assert
the response contract, await the expected event with a bounded timeout,
validate key/headers/schema/business fields, then verify downstream
DB/API state and invariants. On failure I report the correlation ID and
Kafka metadata.

### 16. How do you avoid flaky Kafka tests?

Avoid fixed sleeps, broad event matching, shared consumer groups, and
reused test data. Establish the observation strategy before triggering
the action, use bounded condition polling, isolate records with unique
IDs, and capture diagnostics. Retries are not a substitute for fixing
test races.

### 17. How would you test that no event is published for an invalid request?

Assert the expected API rejection, then verify no forbidden DB/outbox or
downstream effect and observe the relevant event stream for the
contractually defined window using a unique correlation ID. Phrase the
conclusion carefully: the test establishes no event was observed within
the defined scope/window, not an absolute mathematical proof of
non-publication.

### 18. How would you test duplicate events?

Publish/replay the same logical event with the same event/business ID,
verify one logical business effect, then test concurrent duplicates and
restart timing. Validate durable deduplication, metrics, and
reconciliation---not just that the consumer logs "duplicate ignored."

### 19. How do you test eventual consistency?

Poll the expected downstream state until it matches or a bounded
deadline expires. Record the last observed state and elapsed time.
Assert both correctness and a latency SLA. Use fixed sleeps only when
there is a very specific reason, not as the default synchronization
method.

### 20. How would you test a consumer retry mechanism?

Inject a transient dependency failure, assert the configured
attempts/backoff behavior, verify eventual success and no duplicate side
effect, and confirm retry metrics. Then test permanent failure and DLQ
routing, including DLQ failure and controlled replay.

### 21. How would you test schema evolution?

Run compatibility checks in CI, test old and new producer/consumer
combinations, read historical records with the new consumer, and
validate optional fields, defaults, enum behavior, and malformed payload
handling. Test the real rollout order, not just one isolated schema.

### 22. How do you debug "event exists but DB is not updated"?

Find the event by correlation ID and inspect its key, partition, offset,
schema, and payload. Check group membership and lag, consumer
logs/errors, retry/DLQ status, downstream dependency health, and
transaction outcomes. Then verify whether the event is malformed,
unconsumed, repeatedly failing, or processed but not committed.

### 23. How would you test Kafka under load?

Generate a representative event mix with realistic keys and payload
sizes, ramp and sustain load, and measure throughput, p95/p99 latency,
lag, errors, rebalances, resource usage, and downstream latency. Verify
correctness during load and that backlog drains within the recovery
target afterward.

### 24. How do you test Kafka failover?

In a controlled non-production environment, simulate broker/leader
failure within approved test procedures. Verify producer outcomes,
ISR/replica behavior, consumer recovery, data integrity, and recovery
time. A single-broker local test cannot prove multi-broker failover.

### 25. How do you run Kafka tests in parallel with TestNG?

Use unique business data and correlation IDs, isolate consumers/groups
where appropriate, avoid sharing a KafkaConsumer instance across
threads, and control concurrency based on broker and downstream
capacity. Ensure cleanup and offset strategies cannot affect another
test. Diagnose parallel-only failures as possible race, isolation, or
capacity defects.

### 26. What should be included in a Kafka test report?

Test and environment version, topic, key, event ID, correlation ID,
partition/offset, schema version, expected versus actual fields, API
response, downstream state, wait duration, last observed value,
retry/DLQ details, and relevant log links. Redact credentials and
sensitive payload fields.

### 27. How would you measure release risk for an event-driven service?

Assess contract compatibility, critical event flows, idempotency and
replay, lag/recovery, DLQ/retry rates, downstream correctness,
observability, and rollback compatibility. Present residual risk and
evidence to stakeholders rather than relying only on pass percentage.

### 28. What is the difference between producer idempotence and consumer idempotency?

Producer idempotence protects against duplicate writes caused by
supported producer retries. Consumer/business idempotency ensures
repeated delivery does not repeat a business effect. They solve
different failure windows and both may be needed.

### 29. Why might adding consumers not reduce lag?

There may be fewer partitions than consumers, a hot partition,
downstream bottlenecks, serial processing, insufficient broker capacity,
or repeated rebalances. Diagnose partition-level lag and throughput
before scaling.

### 30. What does an SDET Lead own in Kafka testing?

The lead defines event contracts and risk-based coverage, builds
reusable test support, makes async assertions deterministic, validates
recovery and business invariants, establishes observability and CI
gates, controls parallelism, and coordinates with service/platform teams
on release risks. The goal is confidence in the whole workflow, not
merely proving that a message exists.

------------------------------------------------------------------------

# 14. Hands-on lab plan

Use a local Kafka environment or an approved test cluster. Do not run
failure injection or offset resets against production.

## Lab 1 --- Fundamentals

-   Create a topic with three partitions.
-   Publish records with keys and values.
-   Inspect partition and offset metadata.
-   Compare keyed and keyless records.
-   Consume from the beginning with a fresh consumer group.

**Deliverable:** explain why the same key is useful and why offsets are
partition-specific.

## Lab 2 --- Java producer and consumer

-   Create a Java producer and consumer.
-   Add event ID and correlation ID.
-   Validate serialization/deserialization.
-   Add bounded waiting and useful timeout diagnostics.
-   Close clients correctly.

**Deliverable:** a reusable `KafkaEventWaiter` utility.

## Lab 3 --- Consumer groups and lag

-   Start two consumers in one group.
-   Observe partition assignments.
-   Stop one consumer and watch reassignment.
-   Increase production rate and observe lag.
-   Compare with a second independent group.

**Deliverable:** a short incident note explaining lag and rebalancing.

## Lab 4 --- API → Kafka → DB

-   Trigger a test API request.
-   Capture its business ID and correlation ID.
-   Verify event publication.
-   Verify the final downstream state.
-   Add an eventual-consistency deadline.

**Deliverable:** one deterministic end-to-end TestNG test.

## Lab 5 --- Duplicate and crash window

-   Deliver the same event twice.
-   Simulate a consumer restart at a controlled point.
-   Verify idempotent business behavior.
-   Check event and business-effect counts.

**Deliverable:** documented failure windows and evidence.

## Lab 6 --- Retry and DLQ

-   Simulate a transient downstream error.
-   Verify retry/backoff and eventual success.
-   Simulate a permanent invalid record.
-   Verify DLQ metadata and replay procedure.

**Deliverable:** retry/DLQ test matrix.

## Lab 7 --- Schema evolution

-   Add an optional field with a default.
-   Test old payloads against the new consumer.
-   Try an incompatible change in a safe environment.
-   Make compatibility a CI gate.

**Deliverable:** producer-consumer compatibility matrix.

## Lab 8 --- Load and recovery

-   Define event rate, size, key distribution, and latency target.
-   Ramp and sustain load.
-   Observe throughput, p95/p99 latency, lag, and errors.
-   Stop load and measure backlog drain time.

**Deliverable:** performance summary with bottleneck hypothesis and
evidence.

## Suggested 4-week schedule (60--90 minutes/day)

  --------------------------------------------------------------------------
  Week              Focus             Daily practice       Exit outcome
  ----------------- ----------------- -------------------- -----------------
  1                 Fundamentals and  Topics, partitions,  Explain Kafka
                    architecture      offsets, keys,       architecture
                                      replication,         without notes
                                      consumer groups      

  2                 Java automation   Producer/consumer,   Reusable Kafka
                                      bounded waiter,      test utility
                                      event assertions,    
                                      TestNG integration   

  3                 Reliability       Idempotency,         Design tests for
                                      retries, DLQ,        failure windows
                                      outbox, schema       
                                      evolution, replay    

  4                 Lead-level        Lag, performance,    Deliver a
                    systems           recovery, CI/CD,     complete
                                      incident scenarios,  release-risk
                                      mock interviews      answer
  --------------------------------------------------------------------------

Suggested daily split: 20 minutes concept review, 30--45 minutes
hands-on implementation, and 10--20 minutes verbal interview practice.

------------------------------------------------------------------------

# 15. Cheat sheet and final checklist

## Quick definitions

-   **Topic:** named event stream.
-   **Partition:** ordered log and unit of parallelism.
-   **Offset:** position within a partition.
-   **Key:** routing input often used to preserve per-entity partition
    affinity.
-   **Broker:** Kafka server.
-   **Replication factor:** number of partition replicas.
-   **ISR:** replicas currently considered in sync.
-   **Consumer group:** consumers sharing partition assignments.
-   **Lag:** difference between the latest available position and group
    progress.
-   **Retention:** policy for keeping records.
-   **Compaction:** keeps latest values by key, subject to tombstone and
    cleanup rules.
-   **DLQ:** destination for records that cannot be processed under a
    defined failure policy.
-   **Idempotency:** repeated processing does not repeat the logical
    effect.
-   **Outbox:** database-persisted event intent relayed to Kafka.
-   **Rebalance:** reassignment of partitions among group members.

## Common misconceptions to avoid

-   "Kafka deletes a message after a consumer reads it." --- Not
    generally; retention controls cleanup.
-   "Kafka guarantees global ordering." --- Ordering is per partition.
-   "More consumers always increase throughput." --- Partition count and
    downstream bottlenecks limit scaling.
-   "Producer idempotence means the business action happens once." ---
    It does not.
-   "Exactly-once means every external side effect is exactly once." ---
    It does not automatically.
-   "A 200 API response proves the event and downstream state are
    complete." --- Async workflows may not have converged.
-   "A fixed sleep makes async tests reliable." --- It usually makes
    them slower and still flaky.
-   "A DLQ means the failure is handled." --- Ownership, alerting,
    investigation, and replay are still required.
-   "Consumer lag always means Kafka is slow." --- Often the consumer or
    a dependency is the bottleneck.
-   "A single-broker test proves failover." --- It cannot validate
    multi-broker resilience.

## Before approving a Kafka-backed release

-   [ ] Event schemas and compatibility checks pass.
-   [ ] Critical API-to-event-to-state workflows pass.
-   [ ] Correlation IDs and business IDs are asserted.
-   [ ] Duplicate delivery and idempotency are tested.
-   [ ] Retry, poison-message, and DLQ behavior are tested.
-   [ ] Consumer restart/rebalance behavior is understood.
-   [ ] Lag and end-to-end latency are within target.
-   [ ] Backlog drains after a dependency outage.
-   [ ] Replay is controlled and safe.
-   [ ] No forbidden side effects occur for invalid requests.
-   [ ] Dashboards, logs, and alerts support incident diagnosis.
-   [ ] Security, ACLs, secrets, and PII handling are verified.
-   [ ] CI reports contain actionable diagnostics.
-   [ ] Residual risks and rollback plan are communicated.

## A strong 60-second interview answer

> "I test Kafka as part of the complete event-driven business workflow.
> For example, after a payment or beneficiary API call, I validate the
> API contract, capture the business ID and correlation ID, wait for the
> expected Kafka event with a bounded timeout, verify its key, schema,
> headers and payload, and then poll the downstream database or API
> until the expected state converges. At lead level, I also test
> duplicate delivery, idempotency, retry and DLQ handling, ordering,
> consumer restarts, lag recovery, schema compatibility, and
> parallel-test isolation. I use event metadata and correlation IDs to
> make failures diagnosable, and I integrate contract, integration, and
> critical end-to-end tests into CI/CD. I do not assume that a Kafka
> record guarantees a successful business outcome or claim end-to-end
> exactly-once without validating the external side effects."

------------------------------------------------------------------------

# 16. Glossary

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  Asynchronous processing             Work completes after the initiating
                                      request may have returned

  Backpressure                        A downstream stage cannot keep up
                                      with incoming work

  Bootstrap server                    Initial broker address used by
                                      clients to discover the cluster

  Consumer lag                        How far consumer progress trails
                                      the latest available records

  Correlation ID                      Identifier linking logs and events
                                      across a workflow

  Dead-letter topic                   Topic for records that fail
                                      processing under a configured
                                      policy

  Event                               Record describing something that
                                      happened

  Eventual consistency                Different system views converge
                                      after asynchronous processing

  Idempotent operation                Repeating the operation has the
                                      same logical effect as doing it
                                      once

  ISR                                 In-sync replica set for a partition

  Key                                 Record field used for routing and
                                      often entity affinity

  Offset                              Position in a partition log

  Outbox                              Durable database record
                                      representing an event to publish

  Partition                           Ordered, replicated log segment of
                                      a topic

  Rebalance                           Redistribution of partition
                                      assignments in a consumer group

  Retention                           Rules determining how long/how much
                                      data is kept

  Schema Registry                     Service for storing and validating
                                      message schemas

  Tombstone                           Record with a null value commonly
                                      used to represent deletion in
                                      compacted topics
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## Final learning principle

For every Kafka concept, ask three questions:

1.  **What guarantee does the platform provide---and what guarantee does
    it not provide?**
2.  **What failure window could break the business outcome?**
3.  **How would my automated test detect the failure and produce
    evidence an engineer can act on?**

Answering those consistently is what separates a Kafka-aware automation
engineer from an SDET Lead who can own event-driven quality.
