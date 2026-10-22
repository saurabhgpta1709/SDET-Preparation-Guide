# Apache Kafka Interview Questions and Answers
## Beginner to Advanced — SDET Lead / Senior QA Automation Engineer

**Audience:** SDET Lead, Senior QA Automation Engineer, QA Lead, and Software Engineer  
**Focus:** Kafka fundamentals, architecture, reliability, producer/consumer behavior, stream processing, Java automation, end-to-end testing, monitoring, troubleshooting, and system design.

---

## Table of Contents

1. Kafka Fundamentals
2. Architecture, Brokers, and KRaft
3. Replication, Durability, and Availability
4. Topics, Partitions, Retention, and Storage
5. Producer Internals and Reliability
6. Consumers, Offsets, and Consumer Groups
7. Serialization, Schemas, and Validation
8. Kafka Streams and Stream Processing
9. Transactions and Delivery Guarantees
10. Kafka CLI and Administration
11. Monitoring, Security, and Production Support
12. Ordering, Deduplication, and Failure Handling
13. SDET Lead Interview Scenarios
14. Interview Revision Checklist

---

# Part 1: Kafka Fundamentals

## Q1. What is Apache Kafka?

**Answer:**

Apache Kafka is a distributed event-streaming platform used to publish, store, consume, and process streams of records in real time. It allows independent applications to exchange events without requiring direct, synchronous communication between them.

Kafka stores records in partitioned logs across brokers. Producers publish records to topics, and consumers read those records at their own pace.

### Key characteristics

- **Distributed:** Data is spread across multiple brokers.
- **Scalable:** Capacity can be increased through additional brokers and partitions.
- **Durable:** Records are persisted to disk and can be replicated.
- **High throughput:** Batching, sequential writes, and efficient data transfer support large event volumes.
- **Replayable:** Consumers can reread retained records by resetting their offsets.
- **Decoupled:** Producers do not need to know which downstream applications consume their events.

### Example: Banking application

Suppose a customer completes Direct Benefit Transfer (DBT) onboarding.

1. The onboarding service validates the customer's information.
2. After successful registration, it publishes a `DBT_REGISTRATION_COMPLETED` event.
3. A notification service consumes the event and sends a confirmation.
4. An analytics service consumes the same event to update reporting.
5. Another service updates its own customer-related state.

The onboarding service does not need to call every downstream service synchronously.

### What should an SDET test?

- Whether the correct event is published after successful onboarding.
- Whether failed onboarding avoids publishing a success event.
- Whether the event contains the correct customer or correlation identifier.
- Whether downstream consumers process the event correctly.
- Whether retrying a failed operation causes duplicate business effects.
- Whether the event can be replayed when a downstream service recovers.

**Interview takeaway:** Kafka is more than a message queue. It is a durable event log and a platform for building asynchronous, event-driven systems.

---

## Q2. How is Kafka different from traditional message queues such as RabbitMQ or JMS?

**Answer:**

Both Kafka and traditional messaging systems support communication between applications, but their storage, consumption, and processing models differ.

| Feature | Kafka | Traditional message queues |
|---|---|---|
| Primary model | Distributed event log | Message broker and queueing model |
| Message retention | Configurable time/size retention or compaction | Often removed or acknowledged after consumption |
| Replay | Consumers can reread retained records | Depends on broker and queue configuration |
| Ordering | Guaranteed within a partition | Depends on queue and configuration |
| Scaling | Partitions and distributed brokers | Depends on queue topology and broker implementation |
| Consumer progress | Tracked using offsets | Often tracked through acknowledgements |
| Common use cases | Event streaming, analytics, CDC, event-driven systems | Task queues, work distribution, request processing |

These are general architectural differences, not absolute rules. Traditional brokers can support persistence, replay-like patterns, and high throughput, while Kafka can also be used for work distribution.

### Scenario

A banking platform publishes a transaction event.

- The fraud-detection service consumes it.
- The analytics service consumes it.
- The notification service consumes it.

Each service can use a separate consumer group and independently process the same retained event stream.

If three workers belong to the **same consumer group**, however, Kafka distributes partitions among those workers rather than delivering every record independently to all three.

### SDET perspective

Test the intended delivery model:

- Different groups should independently receive relevant events.
- Consumers in the same group should not concurrently own the same partition under a stable assignment.
- A restarted consumer should resume according to its committed offsets and reset policy.

---

## Q3. What are the advantages of Kafka?

**Answer:**

1. **High throughput:** Batching, compression, sequential log writes, and efficient network transfers reduce overhead.
2. **Horizontal scalability:** Partitions can be distributed across brokers and processed by multiple consumers.
3. **Durability:** Records are persisted and can be replicated across brokers.
4. **Fault tolerance:** Replicas can take over when a broker fails, provided an eligible replica is available.
5. **Replay:** Retained records can be consumed again.
6. **Loose coupling:** Producers and consumers can evolve independently when event contracts remain compatible.
7. **Multiple independent consumers:** Different consumer groups can use the same event stream for different purposes.
8. **Ecosystem:** Kafka Connect, Kafka Streams, and schema-management tools support integration and processing workflows.

### Practical testing example

If the notification service is unavailable for 20 minutes, the event stream can retain events while the service is recovering. Once it resumes, it can process the backlog, assuming retention has not expired and its offsets are valid.

An SDET should verify that:
- Events remain available during the outage.
- The consumer catches up after recovery.
- Processing is idempotent where duplicate delivery is possible.
- Backlog recovery does not create unacceptable latency or overload dependencies.

---

## Q4. What are Kafka's limitations or disadvantages?

**Answer:**

- **Operational complexity:** Cluster capacity, partitions, replication, security, and upgrades require careful management.
- **Partition-level ordering:** Kafka does not guarantee a global order across all partitions of a topic.
- **Large-message overhead:** Very large records can increase memory, network, and storage costs.
- **Partition design matters:** Increasing partitions can change key-to-partition mapping and complicate ordering assumptions.
- **Consumer processing is application-owned:** The application must handle business failures, retries, database writes, and idempotency.
- **External side effects are not automatically transactional:** A Kafka transaction does not automatically make a database update and Kafka publication atomic together.
- **Schema compatibility requires planning:** A new event format can break older consumers if compatibility is not managed.
- **Replay has consequences:** Reprocessing historical events can repeat external side effects unless the application is designed to handle it safely.

### SDET Lead response

I would identify risks in four areas:

1. **Data integrity:** Can events be lost, duplicated, or processed out of order?
2. **Resilience:** What happens during broker, consumer, database, or network failure?
3. **Scalability:** Can partitions and consumers handle peak load?
4. **Compatibility:** Can new producers and consumers coexist during deployment?

I would convert these risks into automated tests and measurable release criteria.

---

## Q5. What are the common real-world use cases of Kafka?

**Answer:**

Common use cases include:

- Event-driven microservices communication.
- Real-time transaction and fraud monitoring.
- Centralized log aggregation.
- Database change-data-capture pipelines.
- Analytics and dashboards.
- Notification workflows.
- Audit-event distribution.
- Data integration between applications.
- Streaming transformations and aggregations.

### Banking example

A payment service publishes a `PAYMENT_COMPLETED` event. Fraud monitoring, customer notifications, reconciliation, and analytics consume it independently.

### SDET validation

Verify that:
- The event is published only when the relevant business transition occurs.
- The event contains the correct transaction identifier, amount, currency, and status.
- A failed consumer does not corrupt another consumer group's progress.
- Reprocessing does not create duplicate payments or duplicate financial effects.
- Event-processing latency stays within the defined service-level objective.

---

# Part 2: Kafka Architecture, Brokers, and KRaft

## Q6. What are the key components of Kafka?

**Answer:**

- **Producer:** Publishes records to topics.
- **Consumer:** Reads records from topics.
- **Broker:** A Kafka server that stores partitions and serves client requests.
- **Topic:** A named stream of records.
- **Partition:** An ordered log within a topic.
- **Offset:** A record's position within a partition.
- **Replica:** A copy of a partition maintained for resilience.
- **Consumer group:** Consumers cooperating to process a topic's partitions.
- **Controller quorum:** In KRaft mode, manages cluster metadata and coordination.
- **Coordinator:** A broker-side component that manages a consumer group's membership and offset coordination.

### How they work together

A producer sends a record to a topic. Kafka routes it to a partition, where the record receives an offset. The broker stores the record and replicates it according to the partition's replication configuration. A consumer group reads the record and tracks its progress using offsets.

### SDET perspective

An end-to-end test should verify the entire chain:

`Business action → Producer → Topic/partition → Consumer → Database or downstream result`

A successful producer acknowledgement alone does not prove that the downstream business operation completed successfully.

---

## Q7. What is a Kafka topic?

**Answer:**

A topic is a named stream to which producers publish records and from which consumers read records.

A topic contains one or more partitions. Topics are logical categories, such as:

- `payment-events`
- `customer-onboarding-events`
- `dbt-registration-events`
- `notification-events`

Kafka topics are not normally consumed destructively in the traditional queue sense. Multiple consumer groups can independently read the same retained records.

### Example

The `dbt-registration-events` topic may be consumed by a notification service and a reporting service. Each group maintains its own progress.

### SDET checks

- Verify that the producer uses the expected topic.
- Validate the event key, headers, payload, and timestamp.
- Confirm the intended consumer group receives the record.
- Check the configured retention and cleanup policy.
- Verify access permissions for producers and consumers.

---

## Q8. What is a partition in Kafka?

**Answer:**

A partition is an ordered, append-only log of records. Every record in a partition has an offset that identifies its position.

Partitions enable parallel processing and distribute a topic across brokers.

### Example

A topic has six partitions and six consumers in one group. If each consumer receives one partition, all six can process data in parallel.

However, six consumers do not guarantee six-way parallelism if the topic has fewer than six partitions. Within one consumer group, a partition is assigned to at most one consumer at a time.

### Important considerations

- Ordering is guaranteed within a partition.
- Records with the same key are normally routed to the same partition by the default keyed partitioning strategy.
- More partitions can increase parallelism, but also add operational overhead.
- Partition count affects consumer-group scaling and key distribution.

### Interview scenario

**Question:** A topic has four partitions and eight consumers in the same group. How many consumers can actively own partitions?

**Answer:** At most four consumers can own partitions at a time under a normal assignment. The remaining four are idle until assignments or group membership change.

---

## Q9. What is a Kafka broker?

**Answer:**

A broker is a Kafka server that stores partition replicas and handles client requests.

A broker can:
- Accept produce requests.
- Serve consumer fetch requests.
- Store partition log segments.
- Replicate data as a leader or follower.
- Participate in cluster operations.

A topic can have partitions on multiple brokers. Each partition replica has a role, and the leader handles normal reads and writes for that partition.

### SDET troubleshooting

If a producer cannot publish, investigate:
- Broker availability.
- Bootstrap server configuration.
- Network connectivity and DNS.
- TLS/SASL authentication.
- Topic authorization.
- Partition leader availability.
- Producer timeouts and retry configuration.

Do not assume every producer failure means the Kafka cluster is down.

---

## Q10. What was ZooKeeper's role in Kafka?

**Answer:**

Historically, ZooKeeper managed Kafka cluster metadata and coordination, including broker registration and controller-related coordination.

ZooKeeper was replaced by Kafka's built-in KRaft architecture. Apache Kafka 4.0 removed ZooKeeper mode, so Kafka 4.x deployments use KRaft.

For an interview, distinguish legacy ZooKeeper-based deployments from modern KRaft-based deployments instead of describing ZooKeeper as mandatory for current Kafka.

### SDET relevance

During a migration or upgrade, test:
- Cluster startup and controller availability.
- Topic and partition metadata.
- Leader election and recovery.
- Producer and consumer continuity.
- Monitoring and alerting.
- Operational procedures and rollback plans.

---

## Q11. What is KRaft, and why did Kafka move away from ZooKeeper?

**Answer:**

KRaft stands for Kafka Raft. It is Kafka's built-in metadata-management and consensus architecture. Controller nodes use a Raft-based quorum to maintain replicated cluster metadata without requiring a separate ZooKeeper ensemble.

### Benefits

- Fewer external components to operate.
- A simpler cluster architecture.
- More integrated metadata management.
- Improved metadata scalability and recovery characteristics.
- Easier operational management in modern deployments.

### SDET Lead perspective

A KRaft migration is an infrastructure change with application-level risk. I would build tests for:
1. Cluster health after deployment.
2. Topic and partition metadata consistency.
3. Produce/consume continuity.
4. Leader failover behavior.
5. Consumer lag and recovery time.
6. Security configuration and monitoring.

---

## Q12. What is the `__cluster_metadata` topic?

**Answer:**

In KRaft mode, Kafka stores its cluster metadata in an internal metadata log associated with the controller quorum. The metadata log records cluster changes such as topic and partition metadata.

It is not an application topic and should not be treated as a normal business-event stream.

### Interview point

The metadata log supports recovery of cluster state. The controller quorum uses its replicated metadata state to coordinate cluster operations.

---

## Q13. What is the role of the Kafka controller?

**Answer:**

The controller manages important cluster metadata and coordination activities, including partition leadership changes and topic/partition administrative operations.

In KRaft mode, controller nodes form a quorum. One controller is active at a time, while the others can participate in quorum operations and take over according to the protocol.

### SDET scenario

If a broker hosting a partition leader fails:
1. The cluster detects the failure.
2. The controller coordinates leader election.
3. An eligible in-sync replica may become leader.
4. Clients refresh metadata and retry as appropriate.
5. The failed broker may later rejoin and catch up.

Test the impact on publishing, consumption, lag, duplicates, and end-to-end business completion.

---

# Part 3: Replication, Durability, and Availability

## Q14. How does Kafka ensure fault tolerance?

**Answer:**

Kafka replicates partitions across multiple brokers. One replica is the leader; followers replicate the leader's log.

When the leader fails, Kafka can elect an eligible replica as the new leader. This reduces disruption, provided a suitable replica is available.

### Important conditions

- Replication factor determines the configured number of replicas.
- The ISR (In-Sync Replicas) tracks replicas considered sufficiently caught up.
- Producer acknowledgements determine when a write is reported successful.
- `min.insync.replicas` can require a minimum number of in-sync replicas for writes using `acks=all`.
- If no eligible replica is available, a partition may become unavailable rather than risk unsafe data loss.

**Important:** Replication does not automatically guarantee that every acknowledged write survives every possible failure. Durability depends on the producer configuration, ISR state, replication setup, and failure scenario.

---

## Q15. How does Kafka ensure high availability?

**Answer:**

Kafka supports high availability through:
- Partition replication across brokers.
- Automatic leader election.
- Distributed storage and processing.
- Configurable replication and acknowledgement policies.
- Rack-aware replica placement.
- Client retries and metadata refresh.

### Production example

With a replication factor of three, Kafka aims to maintain three copies of each partition across brokers. If one broker fails and the remaining replicas are healthy, a suitable replica can take over leadership.

### SDET checks

- Publish and consume during a controlled broker failure.
- Measure recovery time.
- Check for missing or duplicate business events.
- Verify producer retry behavior.
- Confirm lag recovers after the broker returns.
- Validate the behavior when too few in-sync replicas remain.

---

## Q16. What happens when a Kafka broker goes down?

**Answer:**

A typical sequence is:

1. The cluster detects that the broker is unavailable.
2. Partitions led by that broker may need new leaders.
3. The controller coordinates leader election from eligible replicas.
4. Producers and consumers refresh metadata and retry operations as configured.
5. When the failed broker returns, its replicas catch up with their current leaders.

If no eligible replica remains, the affected partition may be unavailable until a safe recovery path exists.

### SDET test plan

- Start producing uniquely identified test events.
- Stop a broker in a controlled environment.
- Continue producing and consuming.
- Record producer errors and recovery time.
- Validate all acknowledged event IDs against consumed IDs.
- Check for duplicates and ordering violations.
- Restart the broker and verify replica recovery.

Do not run destructive failure tests in production without approved procedures and safeguards.

---

## Q17. What is `broker.rack`, and why is rack-aware replica placement important?

**Answer:**

`broker.rack` identifies the rack or availability zone in which a broker resides. Rack-aware replica placement tries to spread replicas across different failure domains.

For example, placing replicas in separate availability zones reduces the risk that a single zone failure will remove all replicas of a partition.

### SDET Lead perspective

Validate resilience at the failure-domain level, not only by stopping one broker. The test plan should consider correlated failures, recovery time, replica placement, and whether the remaining replicas can safely serve writes.

---

## Q18. What is a controlled broker shutdown?

**Answer:**

A controlled shutdown allows a broker to leave the cluster in an orderly way. Depending on the cluster configuration and version, Kafka can move partition leadership away from the broker and reduce disruption compared with an abrupt termination.

### Testing considerations

- Verify planned maintenance procedures.
- Observe leader transitions.
- Confirm clients recover without persistent failures.
- Check under-replicated partitions during and after shutdown.
- Validate consumer lag and producer errors.
- Confirm the broker catches up after restarting.

---

## Q19. How does Kafka support scalability?

**Answer:**

Kafka scales through partitioning, distributing partitions across brokers, and allowing consumer groups to process partitions in parallel.

Adding brokers can increase cluster capacity, but existing data and partition assignments may need reassignment to use the new capacity effectively.

### Example

A topic with eight partitions can support up to eight active partition-owning consumers in one group. Increasing the group to sixteen consumers does not create sixteen-way partition parallelism for that topic.

### SDET perspective

Performance tests should vary:
- Number of partitions.
- Number of consumer instances.
- Event size.
- Production rate.
- Processing time per event.
- Replication factor.
- Dependency latency.

Measure throughput, end-to-end latency, consumer lag, error rate, and recovery time.

---

## Q20. How does Kafka handle data replication?

**Answer:**

Each partition can have multiple replicas stored on different brokers. One replica is the leader and handles normal reads and writes. Followers fetch and replicate the leader's records.

The replication factor is configured per topic or partition assignment. Replication improves resilience but increases storage and network requirements.

### Interview example

With replication factor three, the partition has three configured replicas. If one replica falls behind significantly, it may no longer be considered in sync. The number of healthy replicas is therefore not necessarily equal to the configured replication factor.

---

## Q21. What is ISR, and how is it different from replication factor?

**Answer:**

- **Replication factor:** Total configured number of replicas for a partition.
- **ISR:** The subset of replicas currently considered in sync with the leader.

For example, replication factor three means there are three configured replicas. If only two are currently in the ISR, the third replica is not currently considered in sync.

### Why it matters

Producer durability settings such as `acks=all` interact with ISR and `min.insync.replicas`.

A partition with replication factor three does not necessarily have three healthy replicas at every moment.

### SDET scenario

Simulate replica lag in a test environment and verify:
- ISR changes are observable.
- Writes behave according to the configured durability policy.
- Alerts fire when replication health deteriorates.
- Recovery restores replica health.

---

## Q22. What is `min.insync.replicas`, and how does it relate to `acks`?

**Answer:**

`min.insync.replicas` specifies the minimum number of in-sync replicas required for a write when the producer uses `acks=all`.

For example, with replication factor three and `min.insync.replicas=2`, Kafka requires at least two in-sync replicas for the write to succeed with `acks=all`. If the ISR falls below that minimum, the write fails instead of proceeding under weaker durability conditions.

### Relationship

- `acks=0`: Producer does not wait for broker acknowledgement.
- `acks=1`: Producer waits for the leader's acknowledgement.
- `acks=all`: Producer waits for acknowledgement from the current ISR, subject to the configured minimum.

### Recommended testing

Test writes when:
- All replicas are healthy.
- One replica is unavailable.
- ISR falls below the minimum.
- The producer retries after recovery.

Verify that the application distinguishes a failed write from a successful one.

---

## Q23. What is `unclean.leader.election.enable`?

**Answer:**

This setting controls whether Kafka can elect a replica that is not in the ISR as leader when no in-sync replica is available.

- If enabled, availability may improve, but records missing from the out-of-sync replica can be lost.
- If disabled, Kafka avoids electing an out-of-sync replica, prioritizing data safety over immediate availability.

The appropriate setting depends on the system's data-loss tolerance.

### Banking example

For financial transaction events, losing acknowledged records may have serious consequences. A system should make its durability and availability trade-offs explicit rather than enabling unclean election simply to restore service quickly.

---

## Q24. How does Kafka support data consistency?

**Answer:**

Kafka supports ordered append operations within each partition, replication, producer acknowledgements, and idempotent production. Transactions can also provide atomicity for supported Kafka read-process-write workflows.

However, these mechanisms do not automatically make an external database transaction atomic with a Kafka transaction.

### Important distinction

Kafka's log guarantees are not the same as end-to-end business consistency. An application can still create inconsistent outcomes if it commits a database update but fails before publishing the corresponding event.

### SDET approach

- Use unique business and event IDs.
- Compare emitted events with authoritative business state.
- Test crash windows between database writes and event publication.
- Validate consumer idempotency.
- Check ordering assumptions for events belonging to the same entity.
- Reconcile source records with downstream state.

---

# Part 4: Topics, Partitions, Retention, and Storage

## Q25. What is log compaction?

**Answer:**

Log compaction retains the latest known value for each key, subject to Kafka's compaction behavior and cleanup process. It is useful for changelog topics and representing the latest state of an entity.

For example, records for the key `customer-101` might contain several status updates. A compacted topic can eventually remove superseded values while retaining the latest value for that key.

Compaction runs asynchronously. It does not mean every consumer immediately sees only one record per key.

### Compaction versus deletion

A record with a null value in a compacted topic can act as a tombstone, indicating that the key should be removed from the compacted state after the relevant cleanup process.

### SDET checks

- Confirm the topic cleanup policy.
- Verify keys are assigned consistently.
- Test state restoration from a compacted topic.
- Validate tombstone handling.
- Avoid assuming compaction happens immediately after publication.

---

## Q26. What is the role of Kafka's log cleaner?

**Answer:**

The log cleaner runs background compaction for topics configured with `cleanup.policy=compact`. It removes obsolete versions of records for keys while preserving the latest relevant state and respecting compaction rules.

### Why it matters

Compaction allows Kafka to retain a key's latest state without retaining every historical update indefinitely.

### Testing considerations

Test application behavior against the topic's logical state rather than assuming that old records will disappear immediately. If state restoration depends on a compacted topic, verify restoration and deletion behavior explicitly.

---

## Q27. How does Kafka handle data retention?

**Answer:**

Kafka retention is configured using time-based or size-based policies. Old log segments become eligible for deletion when retention criteria are met.

Common settings include:
- `retention.ms`: Time-based retention.
- `retention.bytes`: Size-based retention, applied per partition.
- `cleanup.policy`: Determines whether retention deletion, compaction, or both are used.

### Example

A topic might retain records for seven days. A consumer that falls behind beyond the retained range may no longer be able to read the missing history.

### SDET checks

- Verify the intended retention configuration.
- Simulate a consumer outage that approaches the retention window.
- Confirm recovery works while required records remain available.
- Test behavior when a committed offset has expired or become unavailable.
- Verify replay and audit requirements against actual retention policies.

---

## Q28. What is `log.segment.bytes`?

**Answer:**

`log.segment.bytes` controls the approximate maximum size of a log segment before Kafka rolls to a new segment.

Retention generally operates on whole segments, so segment size and segment age influence how precisely Kafka can enforce retention boundaries.

### Operational impact

Smaller segments may allow more granular retention deletion but create more segment files and metadata overhead. Larger segments reduce file-management overhead but can affect deletion granularity and operational work during reassignment.

---

## Q29. How does Kafka handle message deletion?

**Answer:**

Kafka generally removes records as part of log-segment deletion under retention policies, rather than exposing arbitrary per-record deletion as the normal consumer operation.

Log compaction is a different mechanism that removes superseded values for the same key over time.

### SDET perspective

Do not assume that acknowledging or consuming a record deletes it. Verify retention, compaction, and replay behavior separately.

---

## Q30. What happens when a topic is deleted?

**Answer:**

Topic deletion is a cluster administrative operation. Kafka updates cluster metadata and removes the topic's stored data asynchronously according to its implementation and configuration.

### Testing considerations

- Confirm that topic deletion is authorized.
- Verify the topic is no longer available after deletion completes.
- Check application behavior if it tries to publish to or consume from the deleted topic.
- Confirm recreation behavior and topic configuration.
- Protect production topics from accidental deletion using appropriate permissions and operational safeguards.

---

## Q31. How do you increase partitions? Can partitions be decreased?

**Answer:**

Partitions can be increased with the Kafka topic administration CLI. For example:

```bash
bin/kafka-topics.sh --alter \
  --topic payment-events \
  --partitions 6 \
  --bootstrap-server localhost:9092
```

Partition count cannot normally be decreased in place using the standard topic-alter operation. A common migration approach is to create a new topic with the desired partition count and migrate or republish data.

### Risks of increasing partitions

- Existing key-to-partition mappings may change.
- Events for the same key can be routed to different partitions after the change.
- Ordering assumptions may be affected during migration.
- More partitions increase broker and consumer-group overhead.
- Rebalances may affect processing latency.

### SDET Lead answer

Before increasing partitions, I would assess throughput requirements, key distribution, ordering constraints, consumer capacity, and migration risk. I would run regression tests covering ordering, lag, and event completeness before and after the change.

---

## Q32. What are Kafka's message size limits?

**Answer:**

Kafka imposes configurable size limits at multiple points, including producer requests, broker message acceptance, and consumer fetches. The commonly cited default limit is around 1 MB, but actual effective limits depend on the Kafka version and relevant configuration.

Settings to review include:
- Producer `max.request.size`.
- Broker `message.max.bytes` or the applicable topic-level message-size setting.
- Consumer fetch limits such as `max.partition.fetch.bytes` and `fetch.max.bytes`.

### Best practice

Avoid sending very large payloads when a smaller event plus a reference to external storage will suffice. Large messages can increase memory pressure, network overhead, and latency.

### SDET tests

- Test payloads below, at, and above the agreed limit.
- Verify error handling for rejected messages.
- Confirm that valid messages still flow after an oversized record is rejected.
- Check logs and metrics for actionable errors.

---

## Q33. How does Kafka handle message timestamps?

**Answer:**

Kafka supports timestamp semantics including:

- **CreateTime:** The timestamp supplied by the producer.
- **LogAppendTime:** The broker assigns a timestamp when it appends the record, when configured.

Timestamp behavior is relevant to event-time processing, monitoring, and time-based logic.

### SDET scenario

If an application calculates the time between a payment request and completion, verify which timestamp represents the business event time and which represents broker ingestion time. Clock skew and delayed delivery can otherwise create misleading results.

---

# Part 5: Producer Internals and Reliability

## Q34. What is the Kafka producer API?

**Answer:**

The producer API publishes records to Kafka topics. It handles partition selection, batching, compression, network requests, and retries according to its configuration.

A producer record commonly contains:
- Topic.
- Key.
- Value.
- Optional timestamp.
- Optional headers.

### Java example

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

public class BasicKafkaProducer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
                  "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                  StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                  StringSerializer.class.getName());
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        try (KafkaProducer<String, String> producer =
                 new KafkaProducer<>(props)) {
            ProducerRecord<String, String> record =
                new ProducerRecord<>(
                    "payment-events",
                    "payment-123",
                    "{\"status\":\"COMPLETED\"}"
                );

            producer.send(record, (metadata, exception) -> {
                if (exception != null) {
                    exception.printStackTrace();
                } else {
                    System.out.println(
                        metadata.topic() + "-" +
                        metadata.partition() + "-" +
                        metadata.offset()
                    );
                }
            });

            producer.flush();
        }
    }
}
```

**Interview note:** A real application should use structured logging, externalized configuration, appropriate error handling, and secret management. It should not print sensitive business data or credentials.

---

## Q35. What is the significance of the producer `acks` parameter?

**Answer:**

`acks` controls how much broker acknowledgement the producer waits for before treating a send as successful.

| Setting | Behavior | Main trade-off |
|---|---|---|
| `acks=0` | No broker acknowledgement is awaited | Lowest waiting overhead, weakest delivery confirmation |
| `acks=1` | Leader acknowledges the write | Better confirmation, but a leader failure before replication can risk data loss |
| `acks=all` | Waits for the current ISR's acknowledgement under Kafka's durability rules | Stronger durability, potentially more latency |

For important banking events, `acks=all` combined with suitable replication, `min.insync.replicas`, and idempotent production is a common starting point.

### SDET validation

Test:
- Successful sends with healthy replicas.
- Failure behavior when the ISR drops below the configured minimum.
- Producer retries.
- Duplicate prevention.
- Application behavior when the producer ultimately reports failure.

---

## Q36. What is an idempotent producer?

**Answer:**

An idempotent producer prevents duplicate log entries caused by producer retries within Kafka's producer protocol.

Kafka uses producer identity and sequence numbers to identify duplicate retry requests. Idempotence can be enabled with:

```properties
enable.idempotence=true
```

It is supported with compatible producer settings; avoid configurations that conflict with idempotence requirements.

### Important limitation

Producer idempotence does not automatically prevent duplicates created by:
- The application publishing the same business event twice.
- A client restarting and creating a new logical send.
- A consumer repeating an external side effect.
- A workflow retrying without a stable event or business identifier.

### SDET strategy

Use a stable business event ID and verify both Kafka-level publication and downstream business effects. Producer idempotence and consumer idempotency solve different problems.

---

## Q37. What is the difference between round-robin and key-based partitioning?

**Answer:**

- **Round-robin or sticky distribution:** Spreads records across partitions to improve load distribution, depending on the configured partitioner and record keys.
- **Key-based partitioning:** Uses the key to determine the partition, normally routing the same key consistently under a stable partition count and partitioning strategy.

The exact behavior for unkeyed records depends on the producer's partitioner and Kafka client version. Modern clients can use sticky partitioning to improve batching for unkeyed records.

### Example

Use `accountId` as the key when events for the same account need to preserve their partition order.

### SDET checks

- Verify the correct key is supplied.
- Confirm same-key events reach the intended partition under stable configuration.
- Test key distribution for hotspots.
- Revalidate ordering assumptions after partition-count changes.

---

## Q38. What is the purpose of the producer `Partitioner` interface?

**Answer:**

A partitioner determines the partition to which a producer record is sent. Kafka's default behavior uses the record key when present and follows the configured client partitioning strategy.

A custom partitioner can implement specialized distribution logic, but should be introduced only when the business need justifies the added complexity.

### Testing requirements

- Deterministic partition assignment where required.
- Even distribution for expected workloads.
- No accidental concentration of traffic on one partition.
- Correct ordering for records that must be processed sequentially.
- Compatibility when partitions or producer versions change.

Avoid designing a partitioner that creates a single hot partition unless strict ordering requires it and capacity has been assessed.

---

## Q39. How does Kafka producer batching work?

**Answer:**

The producer buffers records and groups records for the same partition into batches. Batching reduces request overhead and can improve throughput.

Relevant settings include:
- `batch.size`
- `linger.ms`
- `compression.type`

Larger batches may improve throughput but can add latency or increase memory use.

### SDET performance tests

Measure throughput and p95/p99 latency while varying batch size and linger time. Run the tests with realistic message sizes and key distributions rather than assuming one setting is best for every workload.

---

## Q40. What is `linger.ms`?

**Answer:**

`linger.ms` specifies how long the producer may wait for additional records to arrive before sending a batch, subject to other conditions such as batch capacity and delivery deadlines.

Higher values can improve batching and throughput at the cost of added latency for some records.

### Example

A latency-sensitive event might use a small linger value. A high-throughput analytics stream might tolerate a larger value to improve batching efficiency.

The correct value should be established through workload testing.

---

## Q41. What is `max.block.ms`?

**Answer:**

`max.block.ms` bounds how long certain producer operations, such as metadata retrieval or waiting for buffer space, may block before a timeout is reported. It is not a universal deadline for the entire delivery lifecycle.

Producer delivery is also affected by settings such as `delivery.timeout.ms` and request timeout behavior.

### SDET scenario

Fill the producer buffer or make broker metadata unavailable in a controlled environment. Verify that:
- The call does not block indefinitely.
- The expected timeout is reported.
- The application logs useful context.
- Retry or fallback behavior does not silently drop a business event.

---

## Q42. How does Kafka handle delivery timeouts?

**Answer:**

Producer delivery is governed by settings such as `delivery.timeout.ms`, which bounds the time allowed for a record to be successfully delivered, including applicable waiting and retry time.

The consumer has a separate concern: `max.poll.interval.ms` bounds the time between calls to `poll()` before a consumer may be considered unresponsive and removed from the group.

These settings address different sides of the pipeline.

### SDET test cases

- Broker temporarily unavailable.
- Slow network response.
- Producer retries until the delivery deadline.
- Consumer processing takes longer than expected.
- Consumer misses its poll interval and triggers a rebalance.

Verify business-event completeness and avoid treating all timeouts as proof that the event was never written. A timeout can leave the application uncertain about the outcome, so stable IDs and idempotency matter.

---

## Q43. How does Kafka handle message compression?

**Answer:**

Kafka producer compression reduces the size of batches sent to brokers and stored in the log. Supported codecs depend on the Kafka version and client, and commonly include gzip, Snappy, LZ4, and Zstandard.

### Trade-offs

- Compression reduces network and storage usage.
- Compression consumes CPU.
- Compression effectiveness depends on payload structure.
- Batch size influences compression efficiency.

### SDET perspective

Benchmark compression options using realistic event payloads. Compare throughput, CPU, storage, network usage, and end-to-end latency.

---

## Q44. How does the broker handle already-compressed messages?

**Answer:**

Kafka generally works with batches and their compression metadata. Depending on the protocol, message format, and configuration, the broker may preserve the compressed batch rather than repeatedly decompressing and recompressing each record.

Do not assume that every broker always follows one of three fixed behaviors. Exact processing depends on Kafka version and the record format.

### Testing angle

Validate data integrity after compression, compare producer and consumer throughput, and check that all clients use compatible serialization and compression settings.

---

## Q45. What is Kafka's zero-copy mechanism?

**Answer:**

Kafka can use efficient operating-system data-transfer mechanisms, including `sendfile`-style paths, to serve log data without unnecessary copying through application memory.

This can reduce CPU overhead and improve throughput when serving consumers.

### Interview point

Zero-copy is one optimization among several. Kafka's performance also depends on sequential I/O, batching, page cache, network behavior, partition distribution, and producer/consumer configuration.

### SDET performance validation

Do not infer performance solely from broker CPU or disk metrics. Measure the end-to-end pipeline and correlate throughput, latency, network utilization, disk behavior, and consumer lag.

---

# Part 6: Consumers, Offsets, and Consumer Groups

## Q46. What is the difference between a consumer and a consumer group?

**Answer:**

A consumer is an application instance that reads records. A consumer group is a set of consumers that cooperate to consume partitions.

Within a consumer group, each partition is assigned to at most one consumer at a time. Different groups can independently read the same topic.

### Example

- Three consumers in group `notification-service` share the topic's partitions.
- One consumer in group `analytics-service` independently reads the same topic.

The analytics group has its own offsets and does not advance the notification group's progress.

### SDET test cases

- Scale consumers within one group.
- Restart one consumer and observe reassignment.
- Add another independent group.
- Verify that each group processes the expected events.
- Confirm that group offsets are independent.

---

## Q47. What is an offset in Kafka?

**Answer:**

An offset is the position of a record within a particular partition. It is not a globally unique identifier across all topics and partitions.

Consumers use offsets to track progress and resume processing.

### Important distinctions

- The offset belongs to a partition.
- Different partitions have independent offset sequences.
- A consumer group's committed offset records progress for a partition.
- The last committed offset and the next offset to read can differ by one, depending on the API and representation.

### SDET usage

Use topic, partition, offset, and a stable event ID together when diagnosing event-processing issues. Offsets alone are not enough to correlate business outcomes across topics.

---

## Q48. How does Kafka store consumer offsets?

**Answer:**

Kafka stores committed consumer-group offsets in the internal `__consumer_offsets` topic.

Consumers can commit offsets automatically or explicitly through APIs such as `commitSync()` and `commitAsync()`.

### Why it matters

After a restart, a consumer normally resumes according to the last committed offset, unless no usable committed offset exists or the configured reset behavior applies.

### SDET scenario

Crash a consumer before and after committing an offset. Verify which records are replayed and whether replay creates duplicate business effects.

This test is especially important when processing includes a database update or external API call.

---

## Q49. What is the consumer group coordinator?

**Answer:**

The group coordinator is a broker-side component responsible for managing consumer-group membership and offset coordination. It participates in group assignment and rebalance coordination.

The coordinator is not the same as the cluster controller. The controller manages cluster metadata and leadership coordination, while the group coordinator manages a consumer group's protocol and offset operations.

### SDET perspective

When a group cannot make progress, inspect:
- Group state and membership.
- Rebalance events.
- Committed offsets.
- Consumer errors and heartbeat health.
- Partition assignments.
- Processing time and poll intervals.

---

## Q50. What is the role of the consumer heartbeat?

**Answer:**

Heartbeats help the group coordinator determine whether a consumer remains active in the group.

If a consumer stops sending heartbeats beyond the configured timeout, the coordinator can remove it and trigger reassignment.

The exact threading and heartbeat behavior depends on the client and group protocol. The consumer must still meet the poll and processing requirements of its configuration.

### Common failure causes

- Process crash.
- Network partition.
- Long garbage-collection pause.
- Blocking processing.
- Incorrect timeout settings.
- Consumer overload.

### SDET test

Introduce controlled processing delays and observe group stability, partition reassignment, duplicates, and recovery.

---

## Q51. What partition assignment strategies does Kafka support?

**Answer:**

Common assignment strategies include:

- **Range:** Assigns ranges of partitions, typically per topic.
- **RoundRobin:** Distributes partitions across group members.
- **Sticky:** Attempts balanced assignments while minimizing unnecessary movement.
- **CooperativeSticky:** Supports incremental reassignment to reduce the impact of rebalances.

The exact available strategies depend on the Kafka client version and configuration.

### SDET Lead perspective

Measure more than assignment balance. Track:
- Rebalance frequency.
- Time spent rebalancing.
- Consumer lag before and after reassignment.
- Processing interruptions.
- Duplicate side effects during recovery.

---

## Q52. What is KIP-848, and how does it affect consumer rebalancing?

**Answer:**

KIP-848 introduces the newer consumer group protocol, which moves more group coordination and assignment work to the server and supports a more incremental rebalance model. It became generally available in Kafka 4.0.

With older eager rebalancing, consumers may have to relinquish all partitions during a rebalance. Incremental approaches can reduce disruption by limiting reassignment to partitions that need to move.

The actual protocol in use depends on broker/client versions and configuration; upgrading a broker alone does not mean every consumer automatically uses the new protocol.

### SDET migration tests

- Confirm the intended protocol is active.
- Add and remove consumers under load.
- Observe rebalance duration and consumer lag.
- Check partition ownership and processing continuity.
- Validate offset handling during rolling upgrades.
- Compare behavior across mixed client versions where supported.

---

## Q53. What happens during a consumer group rebalance?

**Answer:**

A rebalance occurs when group membership or assignment conditions change. Kafka coordinates partition ownership among the active consumers.

Depending on the protocol, affected consumers may revoke partitions, assignment may be recalculated, and consumers then resume processing assigned partitions.

### Common triggers

- Consumer starts or stops.
- Consumer fails heartbeat or poll requirements.
- Subscription changes.
- Partition count or subscribed topic metadata changes.
- A deployment restarts group members.

### SDET checks

- Verify no partition is concurrently assigned to multiple active consumers in the same group.
- Check for processing pauses and lag spikes.
- Verify committed-offset behavior.
- Ensure reprocessing does not duplicate external side effects.
- Test rolling deployments and scale-up/scale-down operations.

---

## Q54. What is the difference between `subscribe()` and `poll()`?

**Answer:**

`subscribe()` registers the consumer's topic subscription and allows consumer-group coordination and automatic partition assignment.

`poll()` fetches available records and participates in the consumer's group activity. It must be called repeatedly in the normal consumer loop.

### Java example

```java
consumer.subscribe(List.of("payment-events"));

while (running) {
    ConsumerRecords<String, String> records =
        consumer.poll(Duration.ofMillis(500));

    for (ConsumerRecord<String, String> record : records) {
        process(record);
    }
}
```

This is a simplified example. Production code must define shutdown behavior, exception handling, offset commits, and processing guarantees.

---

## Q55. What does `position()` return?

**Answer:**

`position(partition)` returns the next offset the consumer will use for that partition. It can be useful for understanding the consumer's local read position.

It is different from `committed(partition)`, which represents the consumer group's committed progress.

### Example

A consumer can have fetched and processed several records but not yet committed its progress. Its local position may therefore be ahead of its committed offset.

### SDET scenario

Compare position, committed offset, and business processing state during a failure test. This helps identify replay windows and possible duplicate processing.

---

## Q56. What is the difference between `commitSync()` and `commitAsync()`?

**Answer:**

- `commitSync()` waits for the commit operation to complete and reports errors. The API retries certain retriable failures, subject to its behavior and timeouts.
- `commitAsync()` submits the commit without waiting for completion. It is more suitable for throughput-sensitive flows, but failures need careful handling.

### Trade-off

Synchronous commits provide a clearer commit boundary but add latency. Asynchronous commits reduce waiting but can complicate error handling and commit ordering.

### SDET recommendation

Choose offset-commit behavior based on the processing guarantee required. Test crash scenarios around processing and committing rather than assuming one method automatically guarantees exactly-once business processing.

---

## Q57. What does `auto.offset.reset` control?

**Answer:**

`auto.offset.reset` controls what happens when a consumer group has no usable committed offset for a partition, such as when it is starting for the first time or its prior offset is no longer available.

Common values include:

- `earliest`: Start at the earliest available offset.
- `latest`: Start at the end of the log, consuming new records.
- `none`: Throw an error when no usable offset exists.

It does not normally override a valid committed offset.

### SDET example

For a test consumer that must inspect previously published events, `earliest` may be appropriate. For a service that should only process new events when no prior offset exists, `latest` may be intended.

Always make the test's expected starting position explicit.

---

## Q58. What is consumer lag, and how do you monitor it?

**Answer:**

Consumer lag describes how far behind a consumer group is relative to the available end of a partition's log. A common approximation is the difference between the log-end offset and the group's committed offset.

Offset lag is not identical to time lag. A consumer can have a large offset lag but low elapsed delay if production is extremely fast, or a small offset lag but high time delay if records arrive infrequently.

### Common monitoring tools

- `kafka-consumer-groups.sh --describe`
- Kafka AdminClient APIs.
- JMX metrics.
- Prometheus and Grafana.
- Platform-specific Kafka monitoring tools.

### Common causes of high lag

- Slow processing.
- Database or API latency.
- Too few consumers or partitions.
- Repeated retries.
- Consumer errors.
- Rebalances.
- Insufficient downstream capacity.

### SDET Lead response

Set lag thresholds based on business latency requirements. Test peak load and dependency failures, and verify both the alert and the time required to drain the backlog after recovery.

---

## Q59. Does Kafka push or pull messages?

**Answer:**

Kafka uses a pull model: consumers request records from brokers.

This lets consumers control their fetch rate and processing pace. Records can remain available in the log while a consumer catches up, subject to retention.

### SDET implication

A slow consumer does not normally force the broker to push records at an uncontrollable rate. However, the backlog can grow until the consumer catches up or records expire under the retention policy.

---

## Q60. What consumption patterns does Kafka support?

**Answer:**

### Queue-style processing

Consumers within the same group share partitions. Each partition is assigned to at most one consumer at a time, allowing parallel work distribution.

### Publish-subscribe processing

Different consumer groups independently consume the same topic. Each group tracks its own offsets.

### Example

A `payment-events` topic can feed:
- `notification-group`
- `analytics-group`
- `reconciliation-group`

Each group can process the same event stream independently.

### SDET tests

Validate independent group offsets, parallel processing within a group, and the behavior of each group when one downstream service fails.

---

# Part 7: Serialization, Schemas, and Validation

## Q61. How does Kafka handle serialization and deserialization?

**Answer:**

Kafka transports record keys and values as byte arrays. Producer serializers convert application objects into bytes, and consumer deserializers convert bytes back into application representations.

Common formats include strings, JSON, Avro, Protobuf, and JSON Schema-based formats.

### Example

A producer may serialize a payment event to JSON. A consumer deserializes the bytes and maps the fields into a Java event object.

### SDET validation

Test:
- Valid payloads.
- Missing required fields.
- Invalid types.
- Malformed data.
- Unsupported schema versions.
- Null keys and values where applicable.
- Compatibility between producer and consumer deployments.

---

## Q62. What is Schema Registry?

**Answer:**

A Schema Registry stores and serves schemas for supported serialization formats and can enforce configured compatibility rules. It is commonly used with Avro, Protobuf, and JSON Schema.

Kafka itself does not require Schema Registry. It is an additional component that helps manage event contracts.

### Benefits

- Central schema management.
- Schema version tracking.
- Compatibility validation.
- Reduced risk of incompatible producer/consumer deployments.
- Consistent event contracts across teams.

### SDET perspective

Add contract tests that verify:
- New producer schemas satisfy the selected compatibility policy.
- Older consumers can handle supported changes.
- Required fields and data types remain valid.
- Incompatible schema registration fails as expected.
- Rolling deployments work when producers and consumers temporarily run different versions.

---

## Q63. How does Kafka handle schema evolution?

**Answer:**

Kafka stores records as bytes and does not inherently enforce the business schema of every event. Schema evolution is handled by the producer/consumer serialization layer and, optionally, a Schema Registry.

Common compatibility concepts include:

- **Backward compatibility:** A new reader can read data written with an older schema.
- **Forward compatibility:** An older reader can read data written with a newer schema.
- **Full compatibility:** Both backward and forward compatibility are maintained under the selected rules.

Exact compatibility behavior depends on the serialization format and registry configuration.

### Example

Adding an optional field with a safe default may be compatible under a particular schema format and policy. Removing a required field or changing its type may not be compatible.

### SDET testing

Run compatibility tests across the versions that will coexist during deployment, not only against the newest producer and consumer versions.

---

## Q64. How is message versioning handled in Kafka?

**Answer:**

Kafka does not provide an automatic business-schema version field for every event. Applications commonly manage versions through schema identifiers, Schema Registry, or an explicit version field in the event contract.

### Example

A customer event may contain:
- `eventId`
- `eventType`
- `eventVersion`
- `customerId`
- `occurredAt`
- `payload`

The precise format should be standardized across teams.

### SDET checks

- Verify supported versions are accepted.
- Verify unsupported versions fail safely.
- Validate old and new consumers during rolling deployment.
- Test optional fields, defaults, and removed fields.
- Confirm dead-letter or quarantine behavior for incompatible messages.

---

## Q65. How does Kafka hash message keys?

**Answer:**

With the default keyed partitioning behavior, the producer hashes a key to choose a partition. Kafka Java clients commonly use Murmur2 for default keyed partition selection, but custom partitioners and client differences can alter this behavior.

### Why it matters

A stable key helps route related events to the same partition, preserving their partition order.

### Important caveat

Increasing the partition count can change the mapping from keys to partitions. If strict per-key order is important, partition changes need careful planning and validation.

---

## Q66. How is message validation implemented?

**Answer:**

Kafka does not automatically validate business fields in a record. Validation is usually performed by the producer, consumer, a schema-validation layer, or a stream-processing application.

### Recommended validation layers

1. **Producer validation:** Reject invalid business data before publishing.
2. **Schema validation:** Check field types, required fields, and schema compatibility.
3. **Consumer validation:** Verify assumptions before performing side effects.
4. **Business validation:** Confirm that the event is valid for the current business state.
5. **Monitoring:** Track rejected and malformed records.

### SDET scenario

Publish an event with a missing account identifier. Verify whether the producer rejects it or the consumer routes it to a controlled error path. Ensure it does not silently corrupt downstream state.

---

## Q67. How can message deduplication be achieved?

**Answer:**

Kafka provides idempotent production to suppress duplicates caused by producer retries within the Kafka producer protocol. Kafka transactions and Kafka Streams can support exactly-once processing within the Kafka processing boundary.

Application-level duplicates require additional controls, such as:
- Stable event IDs.
- Idempotency keys.
- A database uniqueness constraint.
- A processed-event ledger.
- Idempotent state transitions.
- Safe retry policies.

### Banking example

A payment event might be delivered more than once after a consumer restart. The consumer should not create a second payment merely because it receives the event again.

### SDET test

1. Publish an event with a stable ID.
2. Process it successfully.
3. Replay or redeliver it.
4. Verify the consumer recognizes the duplicate.
5. Confirm the business effect occurs only once.
6. Verify the duplicate is logged or counted appropriately.

---

# Part 8: Kafka Streams and Stream Processing

## Q68. What is the Kafka Streams API?

**Answer:**

Kafka Streams is a Java library for building stream-processing applications using Kafka topics as inputs and outputs. It supports stateless and stateful processing, joins, aggregations, windowing, and state stores.

Unlike a standalone distributed processing engine, Kafka Streams runs within the application process and uses Kafka's partitioning and coordination model to scale its tasks.

### Example

A stream-processing application could read transaction events, filter suspicious patterns, aggregate transactions over a time window, and publish alerts to another topic.

### SDET checks

- Input-to-output transformation correctness.
- Window boundary behavior.
- Late and out-of-order event handling.
- Recovery and state restoration.
- Duplicate and replay behavior.
- Processing latency and throughput.

---

## Q69. What is the difference between a Kafka consumer and a Kafka Streams application?

**Answer:**

A Kafka consumer primarily reads records and exposes them to application code. The application is responsible for its processing and external side effects.

Kafka Streams provides a higher-level framework for defining processing topologies. It handles many concerns associated with task assignment, state stores, changelogging, and fault-tolerant stream processing.

| Area | Kafka consumer | Kafka Streams |
|---|---|---|
| Primary role | Read records | Process and transform streams |
| Processing model | Application-defined loop | Declarative or processor-based topology |
| Stateful operations | Implemented by the application | Supported by state stores and stream-processing APIs |
| Output handling | Application-defined | Integrated output topics and processing |
| Operational complexity | Lightweight client, but application owns many details | More built-in stream-processing capabilities |

### SDET perspective

Choose tests based on the processing model. For a basic consumer, test offset commits and external side effects. For Kafka Streams, also test state restoration, joins, windows, topology behavior, and exactly-once configuration.

---

## Q70. What does the Kafka Streams DSL provide?

**Answer:**

The Domain Specific Language (DSL) offers a high-level way to define stream transformations, filtering, grouping, joining, aggregation, and windowing.

### Example

A topology might:
1. Read payment events.
2. Filter failed or invalid records.
3. Group valid events by account.
4. Aggregate totals over a time window.
5. Publish a derived event.

### SDET approach

Validate both individual transformation rules and the complete topology. Include boundary values, empty inputs, duplicate events, late-arriving events, and restart recovery.

---

## Q71. What is the difference between `KStream` and `KTable`?

**Answer:**

- **KStream:** Represents a stream of individual events. Each record is treated as an event that occurred.
- **KTable:** Represents a continuously updated table-like view keyed by records. New records for a key update its current state.

### Example

A stream of account transactions contains individual debit and credit events. A table can represent the latest calculated state for each account.

A KTable is not simply an ordinary database table; its semantics depend on the stream-processing topology, keys, and updates.

### SDET checks

- Verify key-based update behavior.
- Confirm aggregates after multiple events.
- Test null-value/tombstone handling where applicable.
- Validate state recovery after restart.
- Check ordering and duplicate assumptions.

---

## Q72. What types of windowing does Kafka Streams support?

**Answer:**

Windowing groups records into time-bounded or activity-bounded collections for operations such as aggregations and joins.

- **Tumbling windows:** Fixed-size, non-overlapping windows.
- **Hopping windows:** Fixed-size windows that overlap because the advance interval is smaller than the window size.
- **Sliding windows:** Windows based on time relationships between events, commonly used in stream joins.
- **Session windows:** Dynamically sized windows separated by an inactivity gap.

### Example

A fraud-monitoring system may count transactions per account over a rolling time window. A session-based analytics system may group user events until a period of inactivity ends the session.

### SDET test cases

- Event exactly at a window boundary.
- Events arriving out of order.
- Events arriving after a window has closed.
- Empty windows.
- Duplicate events.
- Incorrect timestamps or clock skew.
- State restoration after restart.

Always verify the exact semantics of the topology and configured grace periods.

---

## Q73. What is a Kafka Streams state store?

**Answer:**

A state store holds local processing state required by operations such as joins, aggregations, and windowing. Kafka Streams can use local storage implementations such as RocksDB or in-memory stores.

Stateful stores are commonly backed by Kafka changelog topics so state can be restored after a failure or task migration.

### SDET perspective

- Validate state after multiple updates.
- Restart the application and verify restoration.
- Test recovery from a deleted or corrupted local state directory in a controlled environment.
- Verify changelog configuration.
- Measure restoration time and processing lag.

---

# Part 9: Transactions and Delivery Guarantees

## Q74. What is the Kafka Transactions API?

**Answer:**

The Transactions API supports atomic writes across multiple Kafka partitions and topics. It is important for Kafka workflows that consume records, process them, and publish output records while coordinating consumed offsets with produced results.

A transaction can commit all its Kafka writes together or abort them together.

### Important limitation

A Kafka transaction does not automatically include arbitrary database updates, HTTP calls, or other external side effects.

### SDET test

Simulate a failure before transaction commit and verify that downstream consumers using `read_committed` do not process aborted output records as committed business results.

---

## Q75. How does Kafka achieve end-to-end exactly-once semantics?

**Answer:**

Exactly-once semantics (EOS) requires more than a single configuration property. Within a Kafka read-process-write workflow, the main mechanisms include:

1. **Idempotent production:** Prevents duplicate Kafka log entries caused by protocol-level retries.
2. **Transactions:** Groups output records and consumed offsets into one atomic Kafka transaction.
3. **Transactional producer configuration:** Uses an appropriate `transactional.id`.
4. **`read_committed` consumers:** Read only committed transactional records.
5. **Correct processing logic:** Ensures the application follows the intended transaction and offset lifecycle.

Kafka Streams provides supported exactly-once processing modes for Kafka-based input/output workflows.

### Critical distinction

Exactly-once processing inside Kafka does not automatically guarantee exactly-once effects in an external database or payment API. External systems require compatible transactional integration, idempotency, or a design such as the transactional outbox pattern.

### SDET Lead scenario

A consumer reads a payment event, updates a database, and then crashes before committing its offset. On restart, it may receive the event again.

If the database update is not idempotent, a duplicate business effect can occur even though Kafka itself is behaving as designed.

I would test this failure window explicitly and verify that the application uses an idempotency key, uniqueness constraint, transaction strategy, or equivalent protection.

---

## Q76. What is the difference between Kafka Streams and Apache Flink?

**Answer:**

Kafka Streams is a Java library tightly integrated with Kafka. It is commonly used for stream-processing applications whose inputs and outputs are Kafka topics.

Apache Flink is a distributed processing engine that supports a wider range of data sources, sinks, and complex processing workloads.

| Area | Kafka Streams | Apache Flink |
|---|---|---|
| Deployment | Embedded library in application instances | Separate distributed processing runtime |
| Kafka integration | Native and direct | Supports Kafka and many other connectors |
| State management | Built-in state stores and changelogging | Distributed state and checkpointing |
| Use cases | Kafka-centric stream-processing services | Broad distributed stream and batch processing |

Neither is universally better. Selection depends on processing complexity, deployment architecture, state needs, operational expertise, and performance requirements.

### SDET relevance

Test the chosen engine's failure recovery, checkpoint/state restoration, ordering assumptions, event-time handling, and output correctness.

---

## Q77. What is ksqlDB, and how does it differ from Kafka Streams?

**Answer:**

ksqlDB provides SQL-like statements for defining stream-processing logic on Kafka data. Kafka Streams is a Java library used to build stream-processing applications programmatically.

### Typical distinction

- Use ksqlDB when declarative SQL-like stream definitions suit the use case and its operational model.
- Use Kafka Streams when custom application logic, Java integration, and direct control over processing topology are required.

### SDET checks

Validate filters, transformations, joins, aggregations, windowing, schema compatibility, and recovery behavior regardless of which interface is used.

---

## Q78. What is Kafka Connect?

**Answer:**

Kafka Connect is a framework for integrating Kafka with external systems through reusable source and sink connectors.

- **Source connector:** Imports data from an external system into Kafka.
- **Sink connector:** Exports data from Kafka to an external system.

Common use cases include database integration, search-index updates, object storage pipelines, and change-data capture.

### SDET test plan

- Validate source-to-topic data correctness.
- Validate topic-to-sink data correctness.
- Test connector restart and retry behavior.
- Verify offset or progress tracking.
- Test schema evolution.
- Confirm duplicate handling and error routing.
- Monitor connector task failures and lag.

---

## Q79. What is Kafka MirrorMaker?

**Answer:**

MirrorMaker is used to replicate Kafka topic data between Kafka clusters. MirrorMaker 2 builds on Kafka Connect and is commonly used for cross-cluster replication, migration, and disaster-recovery scenarios.

### SDET perspective

Validate:
- Source-to-target record completeness.
- Replication lag.
- Topic and configuration mapping.
- Consumer behavior during failover.
- Duplicate or replay behavior.
- Offset translation and recovery assumptions.
- Failback procedures.

Cross-cluster replication should not be assumed to provide a globally atomic transaction across both clusters.

---

## Q80. What is the Kafka AdminClient API?

**Answer:**

AdminClient provides programmatic administration of Kafka resources, including topics, configurations, consumer groups, ACLs, and cluster metadata, depending on the API and permissions available.

### SDET uses

An automation framework can use AdminClient to:
- Create test topics.
- Inspect topic partitions and replicas.
- Query consumer-group information.
- Validate configurations.
- Clean up test resources.
- Confirm the cluster is ready before test execution.

Use unique test resource names and deterministic cleanup to avoid interference between parallel test runs.

---

# Part 10: Kafka CLI and Administration

## Q81. How do you create a Kafka topic using the CLI?

**Answer:**

```bash
bin/kafka-topics.sh --create \
  --topic my-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 2
```

The broker cluster must have enough brokers to satisfy the requested replication factor.

### SDET use

Create isolated test topics with controlled partition counts and retention settings. In shared environments, use unique names and remove test resources only when the cleanup policy permits it.

---

## Q82. How do you list and describe topics?

**Answer:**

List topics:

```bash
bin/kafka-topics.sh --list \
  --bootstrap-server localhost:9092
```

Describe a topic:

```bash
bin/kafka-topics.sh --describe \
  --topic my-topic \
  --bootstrap-server localhost:9092
```

The description can show partitions, leaders, replicas, and ISR information.

### Troubleshooting

If the command fails, check bootstrap server reachability, TLS/SASL settings, permissions, and whether the topic exists.

---

## Q83. How do you increase the number of partitions?

**Answer:**

```bash
bin/kafka-topics.sh --alter \
  --topic my-topic \
  --partitions 6 \
  --bootstrap-server localhost:9092
```

This increases the partition count. It does not decrease it.

### Before changing partitions

Evaluate:
- Ordering requirements.
- Key distribution.
- Consumer capacity.
- Rebalance impact.
- Broker resources.
- Data migration and rollback plans.

---

## Q84. How do you produce and consume messages from the command line?

**Answer:**

Start a console producer:

```bash
bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic
```

Start a console consumer:

```bash
bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic \
  --from-beginning
```

### SDET use

These commands are useful for smoke testing connectivity and examining simple records. For structured events, use appropriate serializers/deserializers and avoid exposing sensitive production data in terminal output.

---

## Q85. How do you check consumer-group lag?

**Answer:**

```bash
bin/kafka-consumer-groups.sh --describe \
  --group my-consumer-group \
  --bootstrap-server localhost:9092
```

Inspect the current committed offset, log-end offset, and lag fields returned by the command.

### Interpretation

A growing lag trend is usually more important than one isolated reading. Correlate lag with producer throughput, processing duration, consumer errors, rebalances, and dependency latency.

---

## Q86. How do you delete a topic?

**Answer:**

```bash
bin/kafka-topics.sh --delete \
  --topic my-topic \
  --bootstrap-server localhost:9092
```

Topic deletion requires suitable permissions and cluster configuration. Data removal may be asynchronous.

### SDET warning

Never use topic deletion as a generic cleanup step against shared or production resources. Test cleanup must verify that the target is an isolated test topic.

---

# Part 11: Monitoring, Security, and Production Support

## Q87. How do you monitor Kafka in production?

**Answer:**

Kafka monitoring should cover brokers, replication, producers, consumers, and downstream processing.

### Important metrics

**Broker health**
- Request latency.
- Request queue time.
- Network and disk utilization.
- CPU and memory.
- Offline or under-replicated partitions.
- Controller and quorum health.

**Replication**
- ISR shrink/expand activity.
- Under-replicated partitions.
- Leader-election frequency.
- Replica fetch lag.

**Consumers**
- Consumer-group lag.
- Rebalance frequency and duration.
- Records consumed per second.
- Processing latency.
- Failed records and retries.

**Producers**
- Record-send rate.
- Error and retry rate.
- Request latency.
- Buffer exhaustion and delivery timeouts.

### Common tooling

- JMX metrics.
- Prometheus and Grafana.
- Kafka command-line tools.
- Kafka AdminClient.
- Centralized logs and alerting.
- Vendor or platform-specific monitoring solutions.

### SDET Lead response

I would define actionable alerts tied to business SLAs, such as event-processing delay, sustained consumer lag, under-replicated partitions, or failure rates. I would also test the alerts and recovery procedures, not only the dashboard.

---

## Q88. What is the Kafka Quota API?

**Answer:**

Kafka quotas limit client resource usage, such as produce or fetch bandwidth and request rates, depending on the quota type and configuration.

Quotas help prevent a single client from monopolizing broker resources and can improve fairness across tenants.

### SDET validation

- Generate traffic above the configured quota in a controlled environment.
- Observe throttling behavior.
- Confirm that one client's workload does not unexpectedly starve others.
- Verify monitoring and alerts.
- Check recovery when traffic falls below the limit.

---

## Q89. What security mechanisms does Kafka provide?

**Answer:**

Kafka supports several security layers:

1. **Encryption in transit:** TLS protects network communication.
2. **Authentication:** SASL mechanisms can verify client identity.
3. **Authorization:** ACLs or configured authorization mechanisms control access to topics, groups, and administrative resources.
4. **Quotas:** Limit resource use by clients or users.
5. **Operational controls:** Secure secret storage, certificate rotation, network isolation, and auditing complement Kafka's protocol-level controls.

### SDET security test cases

- Unauthenticated client is rejected.
- Incorrect credentials are rejected.
- Authorized producer can publish to the permitted topic.
- Unauthorized producer cannot publish to a protected topic.
- Consumer cannot access a topic without permission.
- Consumer-group access is restricted appropriately.
- TLS certificate and trust configuration behave as expected.
- Credentials are not exposed in logs.

Do not place real credentials in source code, test reports, or version control.

---

## Q90. How does Kafka handle message ordering?

**Answer:**

Kafka guarantees ordering within a partition. It does not provide a global ordering guarantee across all partitions of a topic.

### Example

If all events for `account-123` use the account ID as the key, they will normally route to the same partition under a stable partitioning configuration, preserving their partition order.

### Important caveats

- Ordering depends on the partitioning strategy.
- Increasing partition count can change key mapping.
- Concurrent producer requests and retries require careful configuration.
- Multiple consumers cannot process the same partition concurrently within one group, but application-level asynchronous processing can still reorder completion unless controlled.

### SDET test

Publish a sequence of events for the same entity, such as:

`ACCOUNT_CREATED → ACCOUNT_VERIFIED → ACCOUNT_ACTIVATED`

Verify the consumer observes and applies them in the intended order. Check both Kafka record order and business-processing completion order.

---

## Q91. What are Kafka's message delivery guarantees?

**Answer:**

The common delivery guarantees are **at-most-once**, **at-least-once**, and **exactly-once**.

### At-most-once

A message is processed zero or one time from the application's perspective. It can be lost if progress is committed before successful processing.

### At-least-once

A message is processed one or more times. It should not be lost under the intended retry and offset-handling design, but duplicate processing can occur.

### Exactly-once

The intended result is applied exactly once within a defined processing boundary. Kafka transactions can provide exactly-once semantics for supported Kafka-to-Kafka processing flows. External effects require their own transactional or idempotent design.

### SDET Lead answer

I would never claim exactly-once behavior without defining the boundary. I would clarify whether the guarantee applies to Kafka records, database updates, payment APIs, notifications, or the entire business workflow.

I would then test crash windows, retries, replays, duplicate event IDs, and offset commits to demonstrate the guarantee.

---

# Part 12: SDET Lead Interview Scenarios

## Q92. How would you test an API → Kafka → database workflow end to end?

**Answer:**

I would test the full business flow rather than relying on an API response alone.

### Test flow

1. Submit a valid API request.
2. Verify the API response and correlation ID.
3. Confirm that the expected event is published.
4. Validate the event key, schema, headers, and business fields.
5. Wait for the consumer to process the event.
6. Query the database or authoritative service to verify the resulting state.
7. Confirm no duplicate business effects occurred.
8. Capture logs and processing latency for diagnosis.

### Failure scenarios

- API validation failure.
- Producer timeout.
- Consumer unavailable.
- Database timeout.
- Duplicate event.
- Malformed event.
- Event replay.
- Consumer restart before or after offset commit.

### Automation design

Use unique test data and correlation IDs. Prefer bounded polling with a clear timeout over arbitrary fixed sleeps. Ensure the test can distinguish a real product failure from infrastructure setup failure.

---

## Q93. How would you test Kafka in a banking DBT onboarding workflow?

**Answer:**

Consider a workflow where a customer completes DBT onboarding, including validation checks, duplicate-account checks, beneficiary creation, and event publication.

I would validate the following:

### Positive flow

- A valid onboarding request completes.
- The expected registration event is published.
- The event contains the correct business identifier and status.
- The downstream consumer processes the event.
- The database reflects the expected beneficiary state.
- A notification or other downstream action occurs as designed.

### Negative flow

- Invalid Aadhaar or NPCI verification result.
- Duplicate account detected.
- Beneficiary creation fails.
- Database is unavailable.
- Event publication fails.
- Consumer fails after receiving the event.

### Consistency checks

For each test, compare the authoritative onboarding result with the published event and final downstream state. A success event must not be emitted for a failed business transition unless the system contract explicitly defines a different event.

### Reliability checks

- Retry behavior after temporary failure.
- Duplicate event handling.
- Consumer recovery.
- Lag recovery after dependency outage.
- Event replay and idempotency.
- Correlation ID propagation through logs and messages.

---

## Q94. How would you test a Kafka consumer that updates a database?

**Answer:**

The main risk is the failure window between processing the Kafka record, updating the database, and committing the offset.

### Example

1. Consumer receives event `E1`.
2. Consumer updates the database.
3. Consumer crashes before committing the offset.
4. Consumer restarts and receives `E1` again.

If the database operation is not idempotent, the second attempt may create duplicate effects.

### Test strategy

- Verify successful processing.
- Force a crash before the database update.
- Force a crash after the update but before offset commit.
- Redeliver the same event.
- Verify the final business state.
- Confirm retry and error-handling behavior.
- Test database uniqueness constraints or processed-event tracking.

### Lead-level recommendation

Define the business idempotency contract explicitly. A Kafka offset commit by itself does not make the database update exactly once.

---

## Q95. How would you test Kafka retries and dead-letter topics?

**Answer:**

A retry strategy should distinguish transient failures from permanent failures and avoid infinite retry loops.

### Test cases

- Temporary database failure eventually recovers.
- Invalid schema is rejected or routed to the designed error path.
- Retry count and delay match the configuration.
- Poison messages do not block unrelated work indefinitely.
- Dead-letter records preserve useful context.
- Original event ID, topic, partition, offset, and failure reason are traceable where the design supports them.
- Replaying a dead-letter record does not create duplicate business effects.

### Important design concern

Retry topics and dead-letter topics are application patterns, not automatic behavior for every Kafka consumer. The application or framework must implement the intended routing.

---

## Q96. How would you test ordering and duplicate handling together?

**Answer:**

I would publish multiple events for the same entity with monotonically increasing business sequence numbers or versions.

Example:

```text
ACCOUNT_CREATED  version=1
ACCOUNT_VERIFIED version=2
ACCOUNT_ACTIVE   version=3
```

Then I would:
1. Publish the sequence with the same partition key.
2. Verify the records arrive in the intended partition order.
3. Introduce retry and consumer restart scenarios.
4. Replay a previous event.
5. Verify the consumer does not apply an older state over a newer state.
6. Confirm duplicate business effects are prevented.

A stable event ID handles duplicate detection, while a version or sequence number can help enforce business ordering rules.

---

## Q97. How would you diagnose increasing consumer lag?

**Answer:**

I would follow a structured investigation.

### Step 1: Confirm the symptom

Inspect consumer-group lag by partition and determine whether lag is growing, stable, or recovering.

### Step 2: Check production rate

Compare producer throughput with consumer throughput. Lag naturally grows if incoming traffic exceeds processing capacity.

### Step 3: Inspect consumers

Review:
- Consumer health.
- Processing latency.
- Errors and retries.
- Rebalance frequency.
- Poll interval violations.
- CPU, memory, and garbage collection.

### Step 4: Inspect dependencies

Check database latency, API response times, connection pools, rate limits, and any external dependency blocking the consumer.

### Step 5: Check Kafka health

Inspect broker request latency, partition leaders, replication health, network utilization, and throttling.

### Step 6: Validate the fix

After tuning or scaling, verify:
- Lag decreases.
- Processing latency meets the SLA.
- Error rates remain acceptable.
- No duplicate or missing business effects occur.

Adding more consumers is not always the solution. If the topic has too few partitions or the database is saturated, scaling consumers alone may not improve throughput.

---

## Q98. How would you design a scalable Kafka test automation framework?

**Answer:**

I would separate infrastructure setup, event production, consumption, assertions, and reporting.

### Suggested components

- `KafkaTestConfig`: Bootstrap servers, security, and topic configuration.
- `KafkaProducerClient`: Publishes test records.
- `KafkaConsumerClient`: Consumes and filters records.
- `EventSchemaValidator`: Validates payloads and schema expectations.
- `CorrelationIdManager`: Connects API requests with Kafka events.
- `DatabaseValidator`: Verifies authoritative state.
- `KafkaAdminHelper`: Creates and inspects test resources.
- `TestDataFactory`: Generates unique business identifiers.
- `KafkaTestCleanup`: Cleans up isolated test resources.
- `TestReporter`: Captures diagnostics and test results.

### Design principles

1. Reuse producer and consumer clients safely.
2. Avoid sharing a non-thread-safe Kafka consumer across test threads.
3. Use unique test data and event IDs.
4. Avoid fixed sleeps when bounded polling can be used.
5. Isolate parallel tests with unique resources or correlation filters.
6. Capture topic, partition, offset, key, and correlation ID in failure reports.
7. Keep credentials outside source control.
8. Make cleanup deterministic and safe.

### Parallel execution concern

A consumer instance is not generally safe for concurrent access by multiple test threads. Each concurrently running test should have an appropriate consumer ownership model, or use a carefully designed shared event collector that routes records by correlation ID.

---

## Q99. How would you make Kafka automation reliable in CI/CD?

**Answer:**

I would make the tests deterministic, observable, and isolated.

### CI/CD strategy

- Use a controlled Kafka test environment.
- Check broker readiness before running tests.
- Create isolated topics or namespaces where appropriate.
- Use unique test data and correlation IDs.
- Run fast contract and smoke tests on pull requests.
- Run broader integration and failure-injection tests in suitable environments.
- Publish test reports and preserve diagnostic logs.
- Track test duration, failure rate, and flaky-test rate.
- Clean up test resources safely.

### Failure classification

Separate:
- Product defect.
- Test automation defect.
- Environment or infrastructure failure.
- Dependency outage.
- Data setup issue.

Do not automatically retry every failure. Retries can hide real defects and create duplicate side effects. Use retries only where their purpose is explicit and measurable.

---

## Q100. How would you test Kafka performance?

**Answer:**

I would define a workload model and measurable service-level objectives before running the test.

### Workload dimensions

- Messages per second.
- Average and maximum event size.
- Number of partitions.
- Replication factor.
- Producer concurrency.
- Consumer concurrency.
- Compression.
- Consumer processing cost.
- Dependency response time.
- Burst duration and recovery period.

### Measurements

- Throughput.
- End-to-end p50, p95, and p99 latency.
- Producer error and retry rates.
- Consumer lag.
- Broker CPU, memory, disk, and network utilization.
- Under-replicated partitions.
- Time to drain backlog after a burst or outage.

### Test types

- Baseline.
- Load.
- Stress.
- Spike.
- Soak.
- Recovery.

### SDET Lead outcome

The result should include the bottleneck, capacity estimate, configuration changes, trade-offs, and evidence that data integrity remains correct under load.

---

## Q101. How would you distinguish a Kafka infrastructure issue from an application defect?

**Answer:**

I would correlate application logs, Kafka metrics, consumer-group state, and downstream system behavior using timestamps and correlation identifiers.

### Infrastructure indicators

- Broker unavailable.
- Partition leader unavailable.
- Under-replicated partitions.
- Network or TLS failures.
- Authentication or authorization errors.
- Cluster request latency spikes.

### Application indicators

- Deserialization exceptions.
- Invalid event schema.
- Business validation failures.
- Database update errors.
- Consumer processing exceptions.
- Incorrect offset-commit order.
- Application deadlocks or excessive processing time.

### Diagnostic approach

1. Confirm the exact failing stage.
2. Check producer acknowledgement and error details.
3. Inspect the relevant topic and partition.
4. Verify consumer-group assignment and offsets.
5. Check application and dependency logs.
6. Reproduce with a minimal controlled test.
7. Verify recovery after the underlying problem is fixed.

A broker timeout and a consumer business-validation exception require different fixes.

---

## Q102. How would you test a schema change during a rolling deployment?

**Answer:**

I would test compatibility across all producer and consumer versions that can coexist during deployment.

### Test sequence

1. Record the current schema and compatibility policy.
2. Validate the proposed schema against the registry.
3. Deploy a new consumer while the old producer remains active.
4. Publish old-version events.
5. Deploy the new producer.
6. Publish new-version events.
7. Verify both consumer versions behave as intended during the overlap.
8. Verify required fields, defaults, and unknown-field handling.
9. Check failure routing for unsupported schemas.
10. Confirm rollback compatibility.

### SDET Lead point

Testing only the newest producer against the newest consumer is insufficient. The risky condition is often a mixed-version deployment.

---

## Q103. How would you test replaying historical Kafka events?

**Answer:**

Replay tests should prove that historical processing is safe and produces the intended state.

### Test strategy

- Record a controlled set of input event IDs.
- Capture the expected final business state.
- Reset a test consumer group's offsets or use a dedicated replay group.
- Replay the retained events.
- Compare the resulting state with the expected state.
- Verify external side effects are not repeated unexpectedly.
- Check processing rate and downstream capacity.
- Verify that replay can be stopped and resumed safely.

### Important caution

Resetting offsets can cause records to be processed again. Use an isolated test group and test data. Do not reset production offsets without an approved recovery plan.

---

## Q104. How would you design Kafka testing for a payment or transaction system?

**Answer:**

I would prioritize correctness, traceability, idempotency, and failure recovery.

### Critical test areas

1. **Business correctness:** Every valid transaction produces the intended event.
2. **Event contract:** Required fields and identifiers are accurate.
3. **Ordering:** Events for the same business entity follow the required sequence.
4. **Idempotency:** Duplicate delivery does not repeat the financial effect.
5. **Failure recovery:** Broker, consumer, database, and network failures are handled safely.
6. **Reconciliation:** Authoritative transaction records match downstream outcomes.
7. **Security:** Sensitive fields and credentials are protected.
8. **Observability:** Every transaction can be traced across API, Kafka, and database layers.
9. **Performance:** Peak throughput and latency meet the business SLA.
10. **Replay:** Recovery procedures do not corrupt business state.

### Lead-level principle

A green Kafka producer test is not sufficient evidence that a payment workflow is correct. The test must validate the business outcome and reconciliation path.

---

## Q105. How should an SDET Lead explain Kafka expertise in an interview?

**Answer:**

A strong answer should connect Kafka fundamentals with practical testing ownership and measurable outcomes, without claiming guarantees that the architecture does not provide.

### Suggested answer structure

**Architecture:** Explain how producers publish keyed events to partitioned topics and how consumer groups process those events.

**Reliability:** Explain replication, acknowledgements, idempotent producers, offset commits, retries, and recovery.

**Testing:** Describe how you validate event contracts, API-to-Kafka-to-database workflows, ordering, duplicate handling, and consumer recovery.

**Automation:** Explain your Java framework design, correlation-ID strategy, consumer ownership, parallel execution, and reporting.

**Observability:** Describe how you use consumer lag, application logs, broker metrics, and correlation IDs to diagnose failures.

**Leadership:** Explain how you define quality gates, choose critical scenarios, prevent flaky tests, set performance targets, and coordinate with developers and platform teams.

### Example interview response

“I approach Kafka testing as an end-to-end data-integrity problem, not just a message-publication check. I validate the event contract, partition key, consumer processing, database state, and downstream business outcome. I design failure scenarios around retries, offset commits, duplicate delivery, consumer restarts, and dependency outages. For scalability, I measure throughput, p95/p99 latency, and consumer lag under realistic load. At the SDET Lead level, I also define reusable automation components, CI quality gates, observability standards, and release criteria based on business risk.”

---

# Part 13: Rapid-Fire Revision

## Quick concepts to remember

| Topic | Key point |
|---|---|
| Topic | Named stream of records |
| Partition | Ordered log within a topic |
| Offset | Record position within a partition |
| Broker | Kafka server |
| Replication factor | Configured number of replicas |
| ISR | Replicas considered in sync |
| `acks=all` | Wait for current ISR acknowledgement under configured rules |
| `min.insync.replicas` | Minimum ISR size required for `acks=all` writes |
| Idempotent producer | Prevents duplicates caused by supported producer retries |
| Consumer group | Consumers sharing partition assignments |
| Consumer lag | Difference between available log position and group progress |
| `auto.offset.reset` | Behavior when a usable committed offset is absent |
| Log compaction | Retains the latest relevant value per key over time |
| KRaft | Kafka's built-in metadata consensus architecture |
| Kafka Connect | Integration framework for sources and sinks |
| Kafka Streams | Library for building stream-processing applications |
| Schema Registry | Manages schemas and compatibility rules |
| Transaction | Atomic Kafka writes across supported partitions/topics |
| `read_committed` | Consumer isolation level that hides aborted transactional records |
| At-least-once | Duplicates may occur |
| Exactly-once | Must be defined within a specific processing boundary |

---

# Part 14: SDET Lead Preparation Checklist

Before the interview, ensure you can confidently explain and demonstrate:

- [ ] Kafka architecture and the role of each component.
- [ ] Topics, partitions, keys, and offsets.
- [ ] Replication factor, ISR, acknowledgements, and durability.
- [ ] KRaft and the historical role of ZooKeeper.
- [ ] Producer retries, idempotence, batching, and compression.
- [ ] Consumer groups, rebalancing, offset commits, and lag.
- [ ] At-most-once, at-least-once, and exactly-once semantics.
- [ ] Schema evolution and contract testing.
- [ ] Kafka Streams, KStream, KTable, and windowing.
- [ ] Kafka Connect and cross-cluster replication.
- [ ] CLI operations and production monitoring.
- [ ] TLS, authentication, authorization, and safe secret handling.
- [ ] API → Kafka → database end-to-end automation.
- [ ] Parallel-test isolation and safe consumer ownership.
- [ ] Failure injection, replay, retry, and dead-letter scenarios.
- [ ] Performance testing and bottleneck analysis.
- [ ] A concrete banking or payment workflow example.
- [ ] A clear explanation of what Kafka guarantees and what the application must guarantee.

## Final interview advice

For every scenario, structure your response as:

1. **Expected behavior:** What should the system do?
2. **Failure risk:** What can go wrong?
3. **Test design:** How will you reproduce and validate it?
4. **Observability:** Which logs, offsets, metrics, and identifiers will you inspect?
5. **Business integrity:** How will you prove there are no missing or duplicate business effects?
6. **Release decision:** What evidence is required before approving the release?

This approach demonstrates both Kafka knowledge and the risk-based engineering mindset expected of an SDET Lead.
