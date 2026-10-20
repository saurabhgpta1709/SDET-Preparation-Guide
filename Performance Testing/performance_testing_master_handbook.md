# Performance Testing Master Handbook
## Beginner to Advanced | SDET Lead | Senior QA Automation Engineer | Software Engineer

**Purpose:** A practical learning guide and interview-preparation reference for designing, scripting, executing, analyzing, and reporting performance tests.

**Primary stack:** Apache JMeter, HTTP/REST APIs, monitoring and observability, databases, Kafka/event-driven services, and CI/CD.

**Important:** Examples use fictional endpoints and illustrative thresholds. Run load, stress, spike, or resilience tests only in systems you own or are explicitly authorized to test. Never point an uncontrolled test at production or a third-party service.

---

## Contents

1. Learning path and outcomes
2. Performance testing fundamentals
3. Non-functional requirements, SLI/SLO/SLA
4. Metrics, percentiles, and formulas
5. Workload modeling and capacity planning
6. Performance test types
7. Test lifecycle and strategy
8. Apache JMeter: setup to advanced
9. JMeter correlation, parameterization, assertions, and scripting
10. Distributed load generation
11. Practical API test patterns
12. Banking API performance-testing capstone
13. Monitoring and observability
14. Bottleneck analysis
15. Databases, caching, Kafka, and infrastructure
16. CI/CD integration and performance gates
17. Reporting and stakeholder communication
18. 50 interview questions with answers
19. 20 scenario-based interview questions
20. Hands-on exercises and solutions
21. Readiness checklist and quick reference
22. Glossary and further learning

---

# 1. Learning path and outcomes

By the end of this guide, you should be able to:

- Translate vague expectations into measurable performance requirements.
- Choose a realistic workload model and explain the trade-offs.
- Build maintainable JMeter test plans for APIs and multi-step business flows.
- Handle authentication, dynamic values, test data, assertions, and cleanup.
- Execute load, stress, spike, soak, scalability, and capacity tests safely.
- Interpret latency percentiles, throughput, error rate, concurrency, and resource use.
- Distinguish an application bottleneck from a load-generator bottleneck.
- Investigate API, database, cache, Kafka, JVM, and infrastructure issues.
- Integrate smoke and regression tests into CI/CD.
- Produce a defensible report and communicate release risk.
- Answer technical and leadership interview questions with evidence.

## Suggested learning sequence

| Stage | Focus | Deliverable |
|---|---|---|
| 1 | Fundamentals and metrics | Explain latency, throughput, percentiles, and SLOs |
| 2 | JMeter basics | A working single-API test plan |
| 3 | Script quality | Assertions, correlation, CSV data, configuration |
| 4 | Workload modeling | A realistic scenario mix and arrival profile |
| 5 | Execution and analysis | A baseline/load report with server telemetry |
| 6 | Troubleshooting | A root-cause hypothesis supported by evidence |
| 7 | Advanced testing | Capacity, stress, spike, soak, and resilience plans |
| 8 | Engineering practice | CI/CD gate, reusable assets, leadership report |
| 9 | Interview readiness | Explain design choices and 20 scenarios clearly |

---

# 2. Performance testing fundamentals

## 2.1 What is performance testing?

Performance testing evaluates how a system behaves under a defined workload. It measures response time, throughput, errors, stability, resource consumption, and capacity.

It answers questions such as:

- Does the service meet its latency objective at expected peak traffic?
- What is the maximum sustainable workload?
- Which component limits throughput?
- Does the system degrade gracefully during a burst or dependency failure?
- Does performance deteriorate over several hours?
- Can the system scale as traffic and data grow?

A response that returns HTTP 200 can still be functionally wrong or too slow. Performance checks must be paired with functional and business correctness checks.

## 2.2 Performance testing versus performance engineering

**Performance testing** applies a workload and measures results. **Performance engineering** embeds performance thinking across requirements, architecture, implementation, profiling, capacity planning, observability, release decisions, and production feedback.

A strong SDET Lead influences performance early, not just at the end of a release.

## 2.3 Key terms

| Term | Definition |
|---|---|
| Workload | Requests, transactions, users, payloads, and timing applied to a system |
| Latency | Time taken for an operation, measured from a defined start to end point |
| Throughput | Completed work per unit of time |
| Concurrency | Number of operations in progress simultaneously |
| Capacity | Highest sustainable workload while meeting defined objectives and constraints |
| Saturation | A resource or queue approaches its effective limit |
| Bottleneck | Component limiting end-to-end performance |
| Baseline | Reference measurement from a known workload and environment |
| Load generator | Client-side tool and infrastructure that produces traffic |
| Correlation | Extracting a dynamic response value for use in a later request |
| Parameterization | Supplying varied inputs, users, IDs, or payloads to a test |
| Think time | Simulated pause between user actions |
| Ramp-up | Period of increasing workload |
| Steady state | Intended stable measurement period |
| Warm-up | Period allowing caches, pools, JIT, and autoscaling to reach representative behavior |

## 2.4 Functional versus non-functional testing

- Functional test: “Does the transfer endpoint create the correct transfer?”
- Performance test: “At the target workload, does the transfer endpoint meet latency and throughput objectives?”
- Reliability test: “Does it continue to work correctly over time and during failure?”
- Scalability test: “How does performance change when capacity is added?”

For banking and payment workflows, speed is not enough. Validate that no duplicate, missing, or inconsistent transactions are produced.

---

# 3. Non-functional requirements, SLI/SLO/SLA

## 3.1 Turn vague requirements into testable requirements

Weak: “The payment API should be fast.”

Stronger example:

> At 250 completed authorization requests per second for 30 minutes, using a representative request mix and dataset, p95 server-observed latency must remain below 300 ms, p99 below 700 ms, technical failure rate below 0.1%, and duplicate authorizations must remain at zero. The test is valid only when generator health, environment configuration, and dependency behavior are representative.

Thresholds here are illustrative. Agree on actual objectives with product, architecture, engineering, and operations.

## 3.2 SLI, SLO, and SLA

- **SLI (Service Level Indicator):** A measured signal, such as successful-request latency.
- **SLO (Service Level Objective):** A target for an SLI, such as 99% of eligible requests below 300 ms.
- **SLA (Service Level Agreement):** A formal agreement with users/customers and possibly contractual consequences.

A performance test should define:
- Which requests count
- Whether latency is client-observed or server-observed
- Which failures count
- Which time window applies
- Whether the objective applies to each endpoint or a transaction
- How excluded traffic is treated

## 3.3 Example acceptance criteria

| Measure | Example objective | Why it matters |
|---|---:|---|
| Offered load | 250 RPS | Defines intended incoming work |
| Completed throughput | At least 245 RPS | Detects inability to keep up |
| p95 latency | Under 300 ms | Captures common tail experience |
| p99 latency | Under 700 ms | Captures slower tail behavior |
| Technical failure rate | Under 0.1% | Measures service failures |
| Business correctness | No duplicate transfers | Protects data integrity |
| Queue lag | Bounded and recovering | Prevents delayed processing |
| Resource headroom | Agreed safe margin | Helps absorb bursts and failures |

Do not label a run “passed” just because the mean latency is low. All required criteria must be considered.

---

# 4. Metrics, percentiles, and formulas

## 4.1 Latency and percentiles

Report the distribution, not only the average.

- **p50:** median; 50% of observations are at or below this value.
- **p90:** 90% are at or below this value.
- **p95:** common tail-latency objective.
- **p99:** slower tail; important for critical services.
- **Maximum:** useful for investigation but highly sensitive to sample size.
- **Mean:** useful for aggregate calculations but can hide a poor tail.

Example:

| Metric | Value |
|---|---:|
| p50 | 90 ms |
| p95 | 260 ms |
| p99 | 900 ms |

Most calls are fast, but the slowest portion may be unacceptable. Investigate queue waits, downstream tails, lock contention, retries, GC pauses, and database waits.

Percentiles are **not additive**. If Service A p95 is 100 ms and Service B p95 is 150 ms, end-to-end p95 is not necessarily 250 ms. Use end-to-end measurements and distributed traces.

## 4.2 Throughput

Common units:
- Requests per second (RPS)
- Transactions per second (TPS)
- Messages per second
- Bytes per second

Separate **offered rate** from **completed throughput**. A generator may schedule 1,000 RPS while the service completes only 700 RPS.

Define a business transaction precisely. One checkout may call several endpoints but count as one checkout.

## 4.3 Error rate

\[
\text{Error rate} = \frac{\text{failed operations}}{\text{attempted operations}} \times 100\%
\]

Define technical failures and business failures separately. Examples include:
- Network errors and timeouts
- HTTP 5xx responses
- Unexpected status codes
- Assertion failures
- Invalid business response
- Duplicate or missing side effects
- Generator-side errors

A business rejection may be expected for a negative test and should not automatically be counted as a technical failure.

## 4.4 Little's Law

\[
L = \lambda W
\]

Where:
- \(L\): average number of items in the system
- \(\lambda\): average arrival/completion rate in a stable system
- \(W\): average time in the system

Example: at 100 requests/sec and an average response time of 0.2 seconds, average in-flight requests are approximately \(100 \times 0.2 = 20\).

This formula relates averages under stable conditions. Do not substitute p95 latency and treat the result as an exact concurrency requirement.

## 4.5 Approximate closed-model throughput

A rough estimate is:

\[
\text{Throughput} \approx \frac{\text{requests per iteration} \times \text{concurrent users}}{\text{iteration response time + think time}}
\]

This is an approximation. Parallel requests, pacing, connection behavior, and tool scheduling affect the result.

## 4.6 Relative change

\[
\%\Delta = \frac{\text{new value} - \text{baseline value}}{\text{baseline value}} \times 100\%
\]

If p95 changes from 200 ms to 250 ms, latency increased by 25%.

## 4.7 Resource metrics

**Application/host**
- CPU utilization and throttling
- Heap, native memory, allocation rate, GC
- Thread count, blocked threads, queue depth
- File descriptors and sockets
- Network throughput, retransmissions, packet drops
- Disk latency, IOPS, queue depth

**Database**
- Query latency and execution plans
- Active sessions, locks, deadlocks, wait events
- Connection-pool utilization and acquisition wait
- Cache/buffer hit rate and storage pressure

**Distributed services**
- Dependency latency and error rate
- Queue depth and consumer lag
- Retry and timeout rates
- Circuit-breaker state
- Cache hit/miss and eviction rate

High CPU may indicate saturation; low CPU does not prove spare capacity. A thread waiting for a database connection consumes little CPU but may create high latency.

## 4.8 Sample size and confidence

A p99 calculated from only 100 requests is unstable: it depends on roughly one observation near the tail. Collect enough samples, repeat runs, and report variation. Zero observed failures in a small sample does not prove the true failure probability is zero.

---

# 5. Workload modeling and capacity planning

## 5.1 Build a workload model

Record:
- Critical business journeys and endpoints
- Scenario mix and request count per transaction
- Arrival rate by endpoint
- User population and session behavior
- Payload size and data distribution
- Read/write ratio
- Think time and pacing
- Authentication and token-refresh behavior
- Cache hit/miss patterns
- Peak, burst, and off-peak traffic
- Duration, ramp, and steady-state window
- Downstream traffic amplification
- Geographic and network assumptions where relevant

Example fictional commerce mix at 500 total RPS:

| Scenario | Share | Approximate rate |
|---|---:|---:|
| Browse | 55% | 275 RPS |
| Search | 20% | 100 RPS |
| Add to cart | 10% | 50 RPS |
| Checkout | 10% | 50 RPS |
| Order status | 5% | 25 RPS |

Shares total 100%. Model the full business journey where appropriate rather than assuming each request is an independent user.

## 5.2 Closed versus open workload models

**Closed model:** a fixed number of virtual users sends work, waits for the response, then continues. When latency increases, the users generate fewer requests.

**Open model:** arrivals are scheduled independently of response completion. This often represents externally driven API traffic better because incoming requests can continue while the service slows.

A closed model can understate overload and hide queueing. Choose the model that resembles real traffic; do not assume virtual users equal requests per second.

## 5.3 Concurrency estimate

At 300 RPS and average response time of 0.4 seconds:

\[
L \approx 300 \times 0.4 = 120
\]

This estimates about 120 average in-flight requests under stable conditions. It is a starting estimate, not a substitute for validating generator capacity and workload scheduling.

## 5.4 Capacity-testing method

1. Define SLOs and safety stop conditions.
2. Prepare representative data and traffic mix.
3. Establish a low-load baseline.
4. Increase the offered rate in controlled increments.
5. Hold each step long enough to stabilize and collect sufficient samples.
6. Record completed throughput, p95/p99, errors, and resource saturation.
7. Find the first objective that fails.
8. Identify the limiting resource with telemetry and traces.
9. Repeat around the boundary to refine the estimate.
10. Test headroom, autoscaling delay, and relevant failure scenarios.
11. Report the highest **sustainable** rate that meets all objectives.

A service briefly reaching 1,000 RPS with exploding p99 and errors does not have a sustainable capacity of 1,000 RPS.

## 5.5 Headroom

If projected peak demand is 600 RPS, a team might target demonstrated capacity of 900 RPS for 50% headroom. The appropriate margin depends on burstiness, growth, scaling delay, failover, and cost. State the rationale.

---

# 6. Performance test types

| Type | Goal | Example |
|---|---|---|
| Load | Validate expected/peak workload | Sustain expected peak for 30 minutes |
| Stress | Find limits and failure behavior | Increase load beyond expected demand |
| Spike | Observe sudden burst behavior | Jump from 50 to 800 RPS quickly |
| Soak/endurance | Find long-term degradation | Sustain representative load for 8 hours |
| Capacity | Find highest sustainable workload | Refine rate until an SLO fails |
| Scalability | Measure response to added capacity | Compare 2, 4, and 8 service instances |
| Volume | Test large data sizes | Query a large production-like dataset |
| Resilience/failover | Observe dependency or node failure | Remove an instance in a controlled environment |
| Benchmark/regression | Compare versions or configurations | Same workload on build A and build B |
| Concurrency | Expose contention and race conditions | Simultaneous updates to shared inventory |

## Typical run stages

1. Script validation / smoke
2. Warm-up
3. Ramp-up
4. Steady state
5. Ramp-down
6. Recovery observation

Separate ramp-up from steady-state reporting when the objective requires stable behavior. Document why any warm-up data is excluded.

---

# 7. Test lifecycle and strategy

## Phase 1: Discovery

- Identify critical journeys and architecture.
- Review dependencies and traffic patterns.
- Gather expected, peak, and burst rates.
- Define SLIs/SLOs and business correctness invariants.
- Agree scope, exclusions, risk, and environment.

## Phase 2: Design

Document:
- Objective and hypothesis
- Workload and scenario mix
- Open/closed model
- Test data and cleanup
- Environment topology
- Duration and ramp
- Metrics, dashboards, traces
- Acceptance criteria
- Stop conditions and recovery steps

## Phase 3: Preparation

- Validate accounts, tokens, and test data.
- Confirm rate limits and third-party constraints.
- Synchronize clocks.
- Verify load-generator capacity.
- Confirm dashboards and logs.
- Decide whether cache should be warm or cold.
- Ensure the target environment is authorized and safe.

## Phase 4: Script validation

At low load:
- Assert status and business response.
- Verify dynamic values and correlation.
- Confirm data setup/cleanup.
- Verify request counts and scenario mix.
- Ensure failures are visible.
- Compare client-side counts with server-side counts.

## Phase 5: Execution

- Run smoke, baseline, and planned test types.
- Annotate deployment/configuration changes.
- Avoid unrelated jobs during the measurement window.
- Monitor stop conditions continuously.
- Repeat important runs.

## Phase 6: Analysis

Correlate load-generator output with:
- Client latency/errors
- Gateway and service metrics
- Database/cache metrics
- Queue lag and consumer throughput
- Distributed traces and logs
- Deployment/configuration events

A metric changing after latency rises may be a consequence rather than the cause. Use a timeline and controlled experiments.

## Phase 7: Report and retest

- State tested scope and exclusions.
- Compare results with each criterion.
- Identify the limiting component and evidence.
- Recommend prioritized actions.
- Repeat the same workload after a change.
- Preserve scripts, data assumptions, environment, and raw results.

---

# 8. Apache JMeter: setup to advanced

## 8.1 What is JMeter?

Apache JMeter is an open-source load-testing tool that can exercise HTTP APIs and many other protocols. It supports GUI test-plan authoring, command-line execution, parameterization, assertions, timers, extractors, plugins, and distributed load generation.

Use the GUI to build and debug. Use non-GUI mode for serious load runs to reduce rendering and listener overhead.

## 8.2 Installation and first steps

1. Install a supported Java runtime required by the JMeter release you choose.
2. Download JMeter from the official Apache JMeter site.
3. Extract it to a local directory.
4. Start the GUI using the platform's JMeter launcher.
5. Create a Test Plan.
6. Add a Thread Group.
7. Add HTTP Request Defaults and HTTP Header Manager.
8. Add an HTTP Request sampler.
9. Add assertions and a results listener for low-load debugging.
10. Run with one user and inspect the response.
11. Save the `.jmx` file.
12. Move load execution to non-GUI mode.

Verify the current JMeter and Java compatibility requirements in official documentation before installation.

## 8.3 JMeter building blocks

| Component | Purpose |
|---|---|
| Test Plan | Overall test definition |
| Thread Group | User threads and execution schedule |
| Sampler | Sends a request or performs an action |
| HTTP Request Defaults | Shared protocol, host, port, or defaults |
| HTTP Header Manager | Sets request headers |
| Config element | Shared configuration |
| Timer | Delays execution / models pacing |
| Assertion | Validates response conditions |
| Extractor / post-processor | Extracts dynamic values |
| Controller | Organizes, repeats, or branches scenarios |
| CSV Data Set Config | Reads parameterized data |
| Listener | Displays or writes results |
| Backend listener | Sends metrics to a monitoring backend when configured |

## 8.4 Build a first HTTP API test

Example target (replace with your authorized test endpoint):
- Method: `GET`
- Path: `/health`
- Expected status: `200`

Steps:
1. Create a Thread Group.
2. Set one thread, one loop, and no meaningful ramp-up.
3. Add HTTP Request Defaults with protocol/host.
4. Add an HTTP Request sampler with the method and path.
5. Add a Response Assertion for status code `200`.
6. Add a lightweight result listener for debugging.
7. Run once and confirm the response.
8. Add server-side verification if available.

Do not load-test a public health endpoint unless you have permission.

## 8.5 Thread Group settings

Common fields:
- Number of Threads (users)
- Ramp-up period
- Loop count
- Scheduler / duration settings, depending on Thread Group type

Example: 100 threads with a 50-second ramp-up means JMeter attempts to start threads across that ramp period. It does **not** guarantee 100 RPS. Actual throughput depends on response time, timers, iteration structure, and thread scheduling.

## 8.6 Timers and pacing

Timers add delay before samplers. Use them to model user think time or request pacing when appropriate. Be careful: a fixed timer is not the same as a guaranteed open arrival rate. Select an appropriate scheduling model for the workload.

## 8.7 Assertions

Useful assertion targets:
- HTTP status
- Required JSON field
- Business result
- Schema or response structure
- Response-time limit (for a narrow script-level check; use aggregate latency distributions for SLOs)
- Absence of an error field

A fast 500 response must not be counted as a successful fast operation. Validate business semantics as well as transport status.

## 8.8 Listeners and results

Useful outputs:
- JTL result file
- Summary/aggregate metrics
- HTML dashboard report
- Backend time-series metrics

Avoid heavy listeners such as detailed response trees during serious load. They consume memory and CPU and can distort results. Keep raw result files and correlate them with server metrics.

## 8.9 Non-GUI execution

```bash
jmeter -n -t performance-test.jmx -l results.jtl
```

Generate an HTML report where appropriate:

```bash
jmeter -n -t performance-test.jmx -l results.jtl -e -o report
```

Use a fresh output directory where required. Confirm the command and options for your installed JMeter version.

Externalize configuration where supported:

```bash
jmeter -n -t performance-test.jmx \
  -l results.jtl \
  -JtargetHost=test.example.internal \
  -Jthreads=50
```

Your test plan must reference these properties for them to have an effect. Do not place credentials in command history, source control, or reports.

## 8.10 JMeter best practices

- Debug in GUI at low load; execute serious tests in non-GUI mode.
- Use assertions and correlation deliberately.
- Use representative data and payloads.
- Reuse connections if representative of production.
- Externalize target, rate, duration, and test data.
- Keep scripts version-controlled.
- Monitor generator CPU, memory, network, and sockets.
- Check request counts on both client and server.
- Do not infer target RPS from thread count.
- Avoid excessive logging and heavy listeners.

---

# 9. JMeter correlation, parameterization, assertions, and scripting

## 9.1 Correlation

Correlation extracts a dynamic value from one response and uses it in a subsequent request.

Example flow:
1. `POST /sessions` returns `sessionId`.
2. Extract `sessionId` from JSON.
3. Use it in `GET /sessions/${sessionId}`.

In JMeter:
- Add a JSON Extractor as a post-processor under the login/session sampler.
- Configure a variable name such as `sessionId`.
- Use the appropriate JSONPath, for example `$.sessionId`.
- Reference it later as `${sessionId}`.
- Add a check that extraction succeeded before proceeding.

Dynamic values can include tokens, order IDs, transfer IDs, CSRF tokens, and pagination cursors. A fixed recorded value may expire or be unique to one user.

## 9.2 Parameterization with CSV Data Set Config

Use a CSV file for unique accounts or inputs.

Example `test-users.csv`:

```csv
username,password,accountId
perf_user_001,REPLACE_WITH_TEST_SECRET,acct_001
perf_user_002,REPLACE_WITH_TEST_SECRET,acct_002
perf_user_003,REPLACE_WITH_TEST_SECRET,acct_003
```

Configure the CSV Data Set Config with:
- Filename
- Variable names such as `username,password,accountId`
- Delimiter
- Recycle on EOF behavior
- Stop thread on EOF behavior
- Sharing mode appropriate for the thread/data design

Avoid committing secrets. Prefer synthetic test identities and a secret store. Decide whether users share rows or receive distinct rows; this changes contention and workload realism.

## 9.3 Authentication

Common patterns:
- Static test token for simple isolated tests
- Login request plus extracted access token
- Token refresh flow
- Client credentials in an authorized test environment

Token acquisition can become a workload of its own. Decide whether authentication should be included in every user journey or performed in setup, and document the choice. Do not accidentally send all users with one production identity if that is not representative.

## 9.4 Assertions and business validation

For each critical request, validate:
- Expected status
- Expected content type
- Required fields
- Valid business state
- No unexpected duplicate side effects

Do not over-assert every response field if doing so creates fragile scripts. Focus on correctness conditions relevant to the scenario.

## 9.5 Test data and cleanup

- Generate unique identifiers or partition data by worker.
- Avoid all users updating one row unless testing contention intentionally.
- Reset or clean disposable data after a run.
- Ensure data volume resembles production.
- Validate idempotency for retried writes.
- Track data created per iteration to prevent uncontrolled growth.

## 9.6 Dynamic properties and maintainability

Use named properties for host, path fragments, user count, and test duration. Keep environment settings out of duplicated test plans. Use clear names and comments for controllers and samplers.

## 9.7 Distributed JMeter execution

A single machine may not generate the intended load. Distributed execution uses multiple load generators coordinated according to the chosen JMeter setup.

Before scaling out:
1. Confirm one generator is insufficient.
2. Check CPU, heap, network, sockets, and GC on the generator.
3. Ensure all generators use compatible JMeter and plugin versions.
4. Distribute test data to avoid collisions.
5. Coordinate environment variables and target settings.
6. Confirm clocks and logs can be correlated.
7. Verify aggregate scheduled rate against server-side request counts.
8. Check network placement and load balancer distribution.
9. Avoid one central controller becoming a bottleneck.
10. Rehearse safe stop and cleanup behavior.

Adding generators does not automatically multiply valid throughput. Network limits, target-side rate limits, data collisions, and orchestration overhead can still constrain the test.

---

# 10. Practical API test patterns

## 10.1 API test plan template

For each scenario, document:

| Field | Example |
|---|---|
| Business journey | Create a transfer |
| Endpoint | `POST /api/transfers` |
| Method | POST |
| Authentication | Test token |
| Input data | Synthetic source/destination account |
| Expected status | 201 or agreed response |
| Functional assertion | Transfer ID and state are valid |
| Workload | 100 arrivals/sec for this scenario |
| Latency criteria | p95/p99 objectives |
| Error definition | Transport plus business correctness |
| Cleanup | Reconcile or reset synthetic test records |

## 10.2 Example k6 script

This optional companion example illustrates an arrival-rate workload and thresholds. It is not required for the JMeter curriculum.

```javascript
import http from 'k6/http';
import { check } from 'k6';

const BASE_URL = __ENV.BASE_URL;

export const options = {
  scenarios: {
    api_load: {
      executor: 'constant-arrival-rate',
      rate: 50,
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 30,
      maxVUs: 150,
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<300', 'p(99)<700'],
    checks: ['rate>0.99'],
  },
};

export default function () {
  const response = http.get(`${BASE_URL}/api/catalog/items`);

  check(response, {
    'status is 200': (r) => r.status === 200,
    'body is not empty': (r) => r.body !== '',
  });
}
```

Run against an authorized test endpoint:

```bash
k6 run -e BASE_URL=https://your-test-host performance-test.js
```

Key points:
- 50 iterations/sec does not necessarily equal 50 HTTP requests/sec if each iteration makes multiple requests.
- A check records pass/fail; the threshold makes the run fail if the aggregate check rate violates the target.
- Inspect dropped iterations in arrival-rate tests.
- Ensure the generator has enough VUs and compute capacity.
- Use realistic data and business checks for write operations.
- Keep secrets outside source control.

---

# 11. Banking API performance-testing capstone

## 11.1 Objective

Design a safe, repeatable performance test for a fictional banking payments platform. The goal is to validate latency, throughput, correctness, and stability across a set of APIs and an asynchronous event flow.

**Use only a dedicated test environment and synthetic accounts.** Do not use real customer accounts or live payment rails.

## 11.2 Fictional architecture

`Load Generator → API Gateway → Authentication → Payment Service → Database`

The service publishes a payment event to a Kafka-like event stream. Consumers process the event and update downstream status/ledger views.

Potential bottlenecks:
- Gateway rate limits
- Authentication/token service
- Payment-service CPU or thread pools
- Database connection pool, locks, or query plans
- Producer retries
- Partition skew
- Consumer lag
- Downstream ledger/status service
- Load-generator saturation

## 11.3 Example endpoints

These are illustrative names, not real service contracts.

| Endpoint | Method | Purpose |
|---|---|---|
| `/auth/token` | POST | Acquire a test token |
| `/api/accounts/{id}/balance` | GET | Read synthetic account balance |
| `/api/payments` | POST | Submit a synthetic payment |
| `/api/payments/{id}` | GET | Read payment status |
| `/api/payments/{id}/events` | GET | Inspect test-event status, if supported |

## 11.4 Example workload mix

Illustrative workload at 200 API requests/sec:

| Scenario | Traffic share | Approximate rate |
|---|---:|---:|
| Balance read | 50% | 100 RPS |
| Payment submission | 20% | 40 RPS |
| Payment status read | 25% | 50 RPS |
| Event/status verification | 5% | 10 RPS |

This is a request mix, not necessarily a business-transaction mix. If one payment transaction triggers several API calls, calculate the resulting total traffic.

## 11.5 Test data design

Prepare:
- Synthetic source accounts with adequate test balances
- Synthetic destination accounts
- Unique client request IDs
- Idempotency keys for payment submissions
- Separate datasets per generator or worker where needed
- Cleanup/reconciliation procedures
- Test accounts isolated from real financial systems

Avoid repeated transfers between a tiny set of accounts unless contention is the scenario under test. Define how balances and ledger states will be verified after the run.

## 11.6 Example test matrix

| Test | Workload | Duration | Primary question |
|---|---|---|---|
| Smoke | 1–5 users / low rate | 2–5 min | Are scripts and data valid? |
| Baseline | Low stable rate | 10–15 min | What is normal behavior? |
| Load | Expected peak | 30 min | Are SLOs met? |
| Stress | Stepwise increase | Controlled | Where does an objective fail? |
| Spike | Abrupt increase | Short, controlled | Does the system recover safely? |
| Soak | Representative sustained rate | Several hours | Is there degradation or leakage? |
| Event lag | Vary producer rate | Controlled | Can consumers keep up? |
| Recovery | Return to normal rate | 10–20 min | Do queues and resources recover? |

Duration and load are examples. Choose them based on system risk and authorized environment capacity.

## 11.7 Example acceptance criteria

- Payment submission p95 below the agreed threshold.
- Payment submission p99 below the agreed threshold.
- Technical failure rate below the agreed limit.
- Completed throughput close to the intended offered rate.
- No duplicate payments for repeated idempotency keys.
- No missing or inconsistent ledger updates in the tested dataset.
- Queue lag remains within an agreed bound and drains after the run.
- Database and service resources retain safe headroom.
- The test generator does not saturate.

## 11.8 Execution procedure

1. Confirm authorization, scope, rate limits, and stop conditions.
2. Verify synthetic accounts and balances.
3. Run one payment end-to-end and reconcile its state.
4. Validate extraction of payment IDs and idempotency keys.
5. Run a low-load smoke test.
6. Compare client request counts with gateway and service counts.
7. Capture baseline latency, throughput, errors, DB metrics, and queue lag.
8. Ramp to target workload.
9. Hold steady state and annotate the timeline.
10. Watch for duplicate operations, unexpected business outcomes, or unsafe saturation.
11. Stop if agreed safety conditions are reached.
12. Reconcile every created payment record and event.
13. Analyze the first failing objective and supporting telemetry.
14. Retest after one controlled change.
15. Publish results, limitations, and follow-up actions.

## 11.9 Example findings and analysis

**Observation:** p95 is acceptable, but p99 rises sharply as payment submission traffic increases.

Investigate:
- Payment-service traces
- DB connection acquisition wait
- Query duration and lock waits
- Producer latency and retries
- Consumer lag and partition skew
- Thread pools and GC
- Idempotency lookup latency

**Possible conclusion format:**

> At the target workload, p95 met the objective but p99 failed. The slowest requests correlated with increased database connection-acquisition wait and longer transaction queries. This supports a database contention hypothesis, but it does not prove that pool size is the root cause. Review query plans and database waits, change one factor, then repeat the same workload and validate both latency and database headroom.

## 11.10 Capstone deliverables

- Workload model and assumptions
- JMeter `.jmx` test plan
- Synthetic test-data specification
- Environment/configuration record
- Run commands and tool versions
- Dashboard screenshots or exported charts
- Latency/throughput/error summary
- Queue-lag and database analysis
- Correctness reconciliation results
- Findings with evidence and confidence
- Prioritized tuning recommendations
- Retest comparison and residual risk

---

# 12. Monitoring and observability

## 12.1 What to monitor

At minimum, align these time series on one timeline:

- Offered request rate
- Completed request rate
- p50/p95/p99 latency
- Technical and business failure rates
- Generator CPU, memory, network, and dropped work
- Service CPU, memory, threads, pools, and GC
- Database query latency, connections, locks, and waits
- Cache hit rate, misses, evictions
- Queue depth, consumer throughput, and lag
- Dependency latency and retry rates

## 12.2 Metrics, logs, and traces

- **Metrics:** aggregated time-series measurements; useful for trends and saturation.
- **Logs:** event details and errors; useful for explaining individual failures.
- **Traces:** a request's path through services and dependencies; useful for locating latency.

Use correlation/request IDs so test traffic can be found in telemetry. Avoid logging secrets or sensitive account data.

## 12.3 Tracing questions

For a slow request:
- Which span dominates end-to-end latency?
- Is the time spent executing or waiting?
- Are downstream calls sequential?
- Is there an N+1 query pattern?
- Are retries hidden inside a client library?
- Do slow requests share a tenant, key, partition, or payload?
- Is the trace sample representative of the slow tail?

Trace sampling may omit rare slow requests; use aggregate metrics alongside traces.

## 12.4 Dashboard design

A useful dashboard should show:
1. Workload and throughput
2. Latency percentiles
3. Error classification
4. Generator health
5. Service resource saturation
6. Database/cache/dependency metrics
7. Queue lag and processing rate
8. Test phase and deployment annotations

A dashboard is not a diagnosis by itself. Correlate events and test hypotheses.

---

# 13. Bottleneck analysis

## 13.1 Repeatable method

1. Reproduce the issue at a controlled workload.
2. Verify that the generator is healthy.
3. Compare offered rate, completed throughput, latency, and errors.
4. Identify the first metric that changes as load increases.
5. Find growing queues or saturated resources.
6. Inspect traces, profiles, query plans, and logs.
7. Form a specific hypothesis.
8. Change one factor where practical.
9. Repeat the same workload.
10. Verify correctness and report trade-offs.

## 13.2 Symptom map

| Symptom | Possible causes | Evidence |
|---|---|---|
| Throughput plateaus, latency rises | Saturation or queue growth | Resource metrics, queues, traces |
| Low CPU, high latency | I/O wait, pool exhaustion, locks, downstream delay | Wait events, thread dumps, traces |
| CPU near 100% | Expensive code, serialization, GC, inefficient algorithm | CPU profile, GC logs |
| Periodic p99 spikes | GC, scheduled jobs, cache expiry, batch processing | Timeline and runtime metrics |
| Errors only at high concurrency | Pool limits, timeouts, rate limits, races | Error types and pool metrics |
| DB latency rises | Poor query plan, locks, missing index, excess queries | Query plan, slow query stats |
| Consumer lag grows | Consumers slower than producers, skew, retries | Per-partition lag and processing time |
| Degrades over hours | Leaks, unbounded queues, resource growth | Memory, handles, queue trends |
| Generator misses target | Generator CPU/network/VU shortage | Dropped iterations, generator metrics |
| New build regresses | Code/config/query/library changes | Same-workload comparison, traces |
| First request is slow | Cold start, TLS, DNS, JIT, cache miss | Connection spans and first-request data |

Treat these as hypotheses. Confirm them with evidence.

## 13.3 Database investigation

1. Identify slow queries from traces or database statistics.
2. Inspect execution plans with representative parameters.
3. Check indexes, row estimates, sorting, and data distribution.
4. Measure locks, waits, and transaction duration.
5. Check query count per business transaction.
6. Compare connection-pool wait with query execution time.
7. Validate index changes against write cost and storage.
8. Retest with representative data and concurrency.

Adding indexes blindly can speed reads but slow writes and increase storage and maintenance cost.

## 13.4 Java/JVM investigation

For Java services, understand:
- Heap versus native memory
- Allocation rate and GC pauses
- Thread dumps and blocked/waiting threads
- CPU profiles and flame graphs
- JIT warm-up
- Thread pools and queue policy
- JDBC pool behavior
- GC logs and container memory limits

Do not prescribe heap sizes or a GC algorithm without workload and runtime evidence.

---

# 14. Databases, caching, Kafka, and infrastructure

## 14.1 Connection pools

If a service has 50 DB connections but 200 requests need a connection concurrently, requests may queue while waiting. CPU can remain low even as latency becomes unacceptable.

Measure active/idle connections, acquisition wait, query duration, DB saturation, and pool exhaustion. Increasing pool size without checking database capacity may worsen the bottleneck.

## 14.2 Kafka and event-driven processing

Monitor:
- Producer rate and send latency
- Consumer throughput
- Consumer lag
- Processing latency
- Partition distribution and hot keys
- Rebalances
- Retries and dead-letter records
- Duplicate processing and idempotency

If producer rate stays above consumer processing rate, backlog grows. A queue buffers bursts but does not create unlimited processing capacity.

Adding consumers may not help when:
- There are too few partitions
- Keys are unevenly distributed
- A downstream database is saturated
- Processing is CPU- or I/O-bound
- Retries dominate work
- Consumer-group rebalances interrupt processing

## 14.3 Retries and timeouts

Retries can amplify load during an incident. Use:
- End-to-end timeout budgets
- Bounded retries
- Exponential backoff and jitter
- Retry only when safe
- Idempotency keys for sensitive writes
- Circuit breakers and bulkheads where appropriate

A client timeout does not prove that the server did not complete the operation. This is especially important for payments.

## 14.4 Caching

Test realistic hit/miss ratios, key cardinality, hot-key distribution, TTL, expiration, invalidation, and cache stampedes. Warm-cache and cold-cache tests answer different questions.

## 14.5 Autoscaling

Measure trigger delay, instance startup/readiness, warm-up, scale-in behavior, cost, and capacity during the transition. Autoscaling does not remove the need for headroom while new instances start.

## 14.6 Infrastructure

Investigate CPU throttling, memory pressure, network saturation, retransmissions, disk latency, file descriptors, container limits, load balancers, and rate limits. Do not assume application code is always the source of a bottleneck.

---

# 15. CI/CD integration and performance-test strategy

## 15.1 Tiered pipeline

**Pull request**
- Fast microbenchmark where meaningful
- API smoke test
- Catch obvious latency or correctness regressions

**Nightly**
- Representative performance regression test
- Stable data and environment
- Compare with a trusted baseline

**Pre-release**
- Full load/capacity test
- Soak or resilience tests according to risk
- Validate headroom and SLOs

**Before major events**
- Peak simulations, failover, autoscaling, and capacity tests

Avoid running uncontrolled stress tests on every pull request.

## 15.2 Example quality gates

- p95 and p99 under agreed limits
- Error rate below target
- Completed throughput reaches the intended rate
- No unexpected dropped iterations
- Business correctness checks pass
- No major resource regression
- Generator remains healthy

Use both absolute thresholds and relative regression checks. A relative-only gate can preserve an already-slow baseline; an absolute-only gate may be too sensitive to test-environment noise.

## 15.3 Illustrative pipeline pseudocode

```yaml
# Illustrative only: adapt to your CI platform and secret manager.
steps:
  - name: Run API performance smoke
    run: |
      jmeter -n \
        -t tests/api-smoke.jmx \
        -l artifacts/results.jtl

  - name: Archive results
    if: always()
    run: upload-artifacts
```

Adapt artifact upload, environment variables, and failure handling to the CI platform. Keep secrets out of the repository and logs.

## 15.4 Reproducibility

Store:
- Script and data version
- Build/commit identifier
- Tool version and configuration
- Environment size and deployment version
- Workload profile and test duration
- Dashboard and trace references
- Raw results and summary
- Known deviations and anomalies

---

# 16. Reporting and stakeholder communication

## 16.1 Report template

1. Executive summary
2. Scope and exclusions
3. Environment and build
4. Workload and data assumptions
5. Acceptance criteria
6. Results table
7. Evidence and dashboards
8. Bottleneck analysis
9. Recommendations and owners
10. Limitations and residual risks
11. Retest comparison

## 16.2 Example results

Fictional run at 250 offered RPS:

| Metric | Objective | Observed | Status |
|---|---:|---:|---|
| Offered rate | 250 RPS | 250 RPS scheduled | Confirm server count |
| Completed throughput | ≥245 RPS | 248 RPS | Met |
| p95 | <300 ms | 275 ms | Met |
| p99 | <700 ms | 820 ms | Failed |
| Technical error rate | <0.1% | 0.05% | Met |
| Duplicate operations | 0 | 0 observed | Met for tested sample |

The test fails the p99 objective even though other measures pass.

## 16.3 Communicate with evidence

Weak: “The API is slow.”

Stronger:

> At 250 RPS, p95 met the 300 ms objective but p99 reached 820 ms against a 700 ms limit. Slow traces correlated with increased database connection-acquisition wait. This suggests pool contention or long-running queries; it does not yet prove that pool size is the root cause. Inspect query plans and database waits, then rerun the same workload after a controlled change.

## 16.4 SDET Lead responsibilities

- Negotiate measurable NFRs and acceptance criteria.
- Prioritize journeys by business risk.
- Review workload realism and test design.
- Ensure generators and environments are adequate.
- Coach the team on causal analysis and evidence.
- Coordinate with developers, SRE, DBAs, platform, and product.
- Communicate release risk without overstating certainty.
- Maintain reusable scripts, dashboards, and CI gates.
- Track recurring performance debt.
- Ensure all testing is authorized and safe.

---

# 17. 50 interview questions with answers

## Fundamentals (1–10)

### 1. What is performance testing?
It evaluates response time, throughput, stability, capacity, and resource behavior under a defined workload. It validates non-functional requirements and helps identify bottlenecks.

### 2. What is the difference between load and stress testing?
Load testing validates expected or peak workload against objectives. Stress testing exceeds expected workload to find limits and understand failure and recovery behavior.

### 3. What is spike testing?
A test that applies a sudden workload increase to evaluate burst handling, queueing, scaling delay, errors, and recovery.

### 4. What is soak testing?
A long-duration test used to detect leaks, resource exhaustion, queue growth, and gradual degradation.

### 5. What is capacity testing?
Finding the highest sustainable workload while meeting all defined SLOs and constraints.

### 6. What is scalability testing?
Measuring how performance changes as resources or instances are added or workload grows.

### 7. What is the difference between latency and throughput?
Latency measures time per operation; throughput measures completed work per unit time. A system can have high throughput and still have poor tail latency.

### 8. What is concurrency?
The number of operations in progress simultaneously. It is not the same as user count or RPS.

### 9. Why is average response time insufficient?
The mean can hide a slow tail. Report percentiles and error distributions alongside the mean.

### 10. Explain p95 and p99.
p95 is a value at or below which approximately 95% of observations fall; p99 similarly covers approximately 99%. They describe distribution tails, not averages.

## Workload and metrics (11–20)

### 11. What is Little's Law?
For a stable system, average work in the system equals average arrival rate multiplied by average time in the system: \(L=\lambda W\).

### 12. What is the difference between RPS and TPS?
RPS counts requests; TPS counts defined business transactions. One transaction may generate multiple requests.

### 13. What is an open workload model?
An arrival model where new work is scheduled independently of response completion. It can better represent externally driven traffic during slowdowns.

### 14. What is a closed workload model?
A fixed user population performs work and waits for each response before continuing. As latency rises, the generated request rate often falls.

### 15. What is coordinated omission?
A measurement bias where the generator fails to schedule or record work during stalls, underrepresenting the latency that independent arrivals would experience.

### 16. How do you estimate concurrency?
Use Little's Law with target throughput and average response time as an initial estimate, then validate against the actual workload model and tool behavior.

### 17. What is the error rate formula?
Failed operations divided by attempted operations, multiplied by 100%. Define technical and business failures explicitly.

### 18. How many samples do you need for p99?
There is no universal count, but very small samples make p99 unstable. Collect enough observations for the tail, repeat runs, and report variability.

### 19. Why can a load test pass but production be slow?
Different workload mix, data size, cache state, dependency latency, network, multi-tenant contention, autoscaling, retries, or environment configuration.

### 20. What is a performance baseline?
A reference run with documented workload, environment, data, and tool configuration that future runs can be compared against.

## JMeter (21–30)

### 21. What is Apache JMeter?
An open-source load-testing tool with test plans, samplers, controllers, timers, assertions, extractors, data configuration, listeners, and command-line execution.

### 22. Why use non-GUI mode?
The GUI and heavy listeners consume resources. Non-GUI execution is more suitable for controlled load generation and CI.

### 23. What is a Thread Group?
A component that defines virtual-user threads and execution behavior such as ramp-up, loop count, or duration depending on the group type.

### 24. Does 100 threads mean 100 RPS?
No. Throughput depends on response time, request count per iteration, timers, iteration structure, and scheduling.

### 25. What is a sampler?
A component that performs an action, such as sending an HTTP request.

### 26. What is an assertion?
A validation that checks a response or result, such as status code or required business content.

### 27. What is correlation?
Extracting a dynamic value from one response and using it in a later request, such as a token or generated payment ID.

### 28. What is parameterization?
Using varied data—often from CSV—to model distinct users, accounts, products, or inputs rather than repeating a single value.

### 29. Why use timers?
To model think time or pacing. Timers delay sampler execution, but they do not automatically guarantee an open arrival rate.

### 30. How do you run JMeter from a terminal?
For example: `jmeter -n -t test.jmx -l results.jtl`. Check the installed version's options and configure the test plan to use any external properties.

## Diagnosis and distributed systems (31–40)

### 31. Throughput plateaus while latency rises. What do you suspect?
Saturation and queue growth. Inspect the generator, service, database, pools, locks, dependencies, and queues to find the first limiting resource.

### 32. CPU is low but latency is high. Why?
The application may be waiting on I/O, a connection pool, a lock, a downstream service, or a queue.

### 33. CPU is high. What do you do?
Profile the process, inspect hot methods, allocation and GC, serialization, logging, algorithms, and concurrency. Optimize evidence-backed hot paths.

### 34. p50 is fine but p99 is poor. What do you investigate?
Slow dependency tails, queueing, GC pauses, locks, retries, hot partitions, cache misses, and noisy neighbors. Correlate slow traces with metrics.

### 35. How do you diagnose a database bottleneck?
Inspect query latency, query plans, indexes, lock/wait events, pool acquisition wait, query count per transaction, and data distribution.

### 36. Why might increasing a DB pool make performance worse?
It may overwhelm the database and increase contention. Tune only after correlating pool wait with DB saturation and query behavior.

### 37. Why does Kafka consumer lag increase?
Consumers may process slower than producers, partitions may be skewed, downstream dependencies may be slow, or retries/rebalances may reduce throughput.

### 38. Why might adding consumers not improve throughput?
Insufficient partitions, hot keys, downstream saturation, expensive processing, retries, or rebalances may limit parallelism.

### 39. How can retries cause a traffic storm?
Retries add more requests during failure and can multiply downstream load. Use bounded retries, backoff, jitter, timeouts, and idempotency.

### 40. What is the role of distributed tracing?
It breaks an end-to-end request into spans, helping identify where time is spent across services and dependencies. Correlate traces with metrics and logs.

## Strategy and leadership (41–50)

### 41. How do you define acceptance criteria?
Translate user/business needs into measurable workload, latency percentiles, throughput, errors, duration, environment, and correctness requirements.

### 42. How do you distinguish regression from noise?
Repeat controlled runs, compare distributions and historical variation, verify environment consistency, and use meaningful absolute and relative thresholds.

### 43. How do you test autoscaling?
Use realistic ramps/spikes and measure trigger delay, readiness, capacity during scaling, recovery, scale-in, and cost.

### 44. How do you integrate performance testing into CI/CD?
Use fast smoke tests for PRs, representative regression runs nightly, and full capacity/soak/resilience tests before releases or major events.

### 45. What should a performance report include?
Scope, environment, workload, acceptance criteria, latency percentiles, throughput, errors, resource metrics, evidence, findings, limitations, and actions.

### 46. What if the load generator is saturated?
Treat the result as invalid for target capacity. Optimize or distribute the generator, then verify intended traffic using server-side counts.

### 47. How do you test a payment API safely?
Use a dedicated test environment, synthetic accounts, idempotency keys, controlled rates, cleanup/reconciliation, explicit stop conditions, and correctness checks.

### 48. What is the difference between monitoring and performance testing?
Testing applies a planned workload to answer a specific question; monitoring continuously observes a running system. Monitoring data should inform test models.

### 49. How do you report an inconclusive run?
State what could not be established and why—for example, generator saturation or unrepresentative data—then specify the correction and rerun. Do not call an invalid test a pass.

### 50. How would you build a performance-testing practice as an SDET Lead?
Define NFRs, prioritize critical journeys, create reusable scripts and dashboards, standardize workload models, add tiered CI gates, train engineers, and maintain reproducible baselines.

---

# 18. 20 scenario-based interview questions

For each scenario, explain **observations → hypotheses → evidence → controlled experiment → result**. Do not jump straight to a fix.

## Scenario 1: p95 passes but p99 fails
**Situation:** p95 is 250 ms, but p99 is 1.4 seconds against a 700 ms limit.

**Approach:** Verify sample size and metric definition. Compare slow traces with normal requests; correlate with DB waits, GC, cache misses, retries, and dependency tails. Change one variable and rerun. Keep the SLO failed until the criterion is met or formally revised.

## Scenario 2: Test reaches 1,000 RPS, production fails at 700 RPS
Compare workload mix, payloads, data cardinality, cache state, dependency latency, network, retries, multi-tenant contention, and RPS definitions. Validate both client and server counts.

## Scenario 3: Consumer lag grows after adding consumers
Check partition count, consumer assignment, hot keys, per-partition lag, processing time, downstream capacity, retries, and rebalances. More consumers only help if work can be parallelized and downstream capacity exists.

## Scenario 4: New release improves latency but raises database CPU
Compare throughput, DB CPU per transaction, query count, error rate, and headroom. Confirm peak and failover capacity. It may be a trade-off, not an unconditional win.

## Scenario 5: Test passes but generator CPU is 98%
The result is not trustworthy for capacity. Reduce generator overhead or distribute load, then verify intended traffic at the server.

## Scenario 6: Payment request times out, then client retries
The original may still have completed. Use idempotency keys, inspect server state, reconcile records, and validate duplicate protection. A timeout alone does not prove failure.

## Scenario 7: Design a 10,000-RPS test
Clarify offered versus completed RPS and endpoint mix. Define workload, SLOs, safety limits, generator capacity, and environment. Scale generators, ramp gradually, compare server counts, stop safely, and document limitations.

## Scenario 8: Latency rises but CPU and memory look normal
Inspect connection-pool waits, DB locks, network, downstream calls, queue depth, rate limiting, and thread dumps. Check telemetry coverage before concluding no resource is saturated.

## Scenario 9: Only the first request is slow
Investigate DNS/TLS, connection establishment, cold starts, JIT warm-up, and cache misses. Decide whether cold-start latency belongs in the objective.

## Scenario 10: Soak test degrades after three hours
Graph memory, GC, handles, connections, queues, cache size, and scheduled jobs over time. Reproduce, isolate the growing resource, and compare after a fix.

## Scenario 11: Errors rise only above 500 concurrent users
Classify exact errors. Check pools, thread limits, rate limits, file descriptors, database contention, race conditions, and test-data collisions. Build a focused test around the failure threshold.

## Scenario 12: p95 suddenly spikes every 15 minutes
Compare the spikes with GC, scheduled tasks, cache expiry, token refresh, backups, and batch jobs. Use synchronized timelines and traces to test the hypothesis.

## Scenario 13: Database query is fast alone but slow under load
Inspect lock contention, concurrent execution plans, pool wait, I/O saturation, transaction duration, and data skew. A single-user query benchmark does not represent concurrent behavior.

## Scenario 14: Autoscaling occurs, but users still see errors
Measure scaling trigger delay, readiness, warm-up, load-balancer registration, connection establishment, and queue growth during the gap. Ensure baseline capacity survives until new instances are ready.

## Scenario 15: Cache dramatically improves the test result
Check hit rate, key cardinality, TTL, hot keys, cold-start behavior, and production distribution. Report warm-cache and cold-cache results separately when both matter.

## Scenario 16: Adding an index improves reads but slows writes
Measure read latency, write latency, DB CPU, storage, and index-maintenance cost. Decide based on the complete workload and business SLOs, not a single query.

## Scenario 17: Error rate is low but business records are missing
Technical success does not guarantee correctness. Reconcile submitted operations with persisted records and events; validate eventual consistency windows, retries, idempotency, and consumer failures.

## Scenario 18: New build is 8% slower than baseline
Repeat runs under identical conditions, check environment noise, compare distributions and endpoint-level metrics, and inspect changed traces/profiles. Determine whether 8% exceeds known variability and the agreed gate.

## Scenario 19: Load test and server logs show different request counts
Check retries, redirects, health checks, dropped iterations, sampling, timeouts, client-side failures, gateway transformations, and log ingestion loss. Define the counting point and reconcile the numbers.

## Scenario 20: Leadership asks whether release is safe
Summarize tested workload and objectives, pass/fail results, critical risks, evidence, environment gaps, headroom, and untested scenarios. Give a recommendation with confidence and residual risk; do not overstate certainty.

---

# 19. Hands-on exercises and solutions

## Exercise 1: Calculate approximate concurrency

**Question:** A service completes 200 RPS at an average latency of 150 ms. Estimate average in-flight requests.

**Solution:**

\[
L \approx 200 \times 0.15 = 30
\]

Approximately 30 in-flight requests on average, assuming stable conditions.

## Exercise 2: Calculate error rate

**Question:** Out of 80,000 attempted operations, 64 fail.

**Solution:**

\[
\frac{64}{80000}\times100\%=0.08\%
\]

Compare this with the actual agreed threshold and failure definition.

## Exercise 3: Calculate regression

**Question:** p95 increases from 240 ms to 300 ms.

**Solution:**

\[
\frac{300-240}{240}\times100\%=25\%
\]

The p95 latency increased by 25%.

## Exercise 4: Identify the bottleneck hypothesis

**Observation:** Completed throughput stops growing; p99 rises; DB connection acquisition wait increases; service CPU remains moderate.

**Strong answer:** Database-pool contention or slow queries are plausible. Inspect pool wait, active connections, DB CPU, lock waits, and query duration. Do not immediately increase pool size.

## Exercise 5: Interpret Kafka lag

**Observation:** Producers send 4,000 messages/sec; consumers process 3,200 messages/sec for a sustained interval.

**Solution:** Backlog grows at approximately 800 messages/sec while those rates persist. Estimate time to clear only after processing capacity exceeds new arrivals and other effects are accounted for.

## Exercise 6: Plan a load test

Write a one-page plan containing:
- Goal
- Workload model
- Traffic mix
- Data
- Environment
- Duration and ramp
- SLOs
- Stop conditions
- Telemetry
- Cleanup
- Report template

## Exercise 7: Improve a weak interview answer

Weak answer: “I will increase threads if response time is high.”

Better answer: “First I will verify the workload and generator, then compare offered and completed throughput, latency percentiles, errors, and resource metrics. I will identify the first constrained component and validate a hypothesis with traces or profiling before changing concurrency or pool sizes.”

## Exercise 8: Design a regression gate

Define an absolute p95/p99 objective, a relative regression tolerance, minimum sample/run requirements, an error threshold, and rules for rerunning noisy or invalid tests. Explain how you will avoid hiding a consistently poor baseline.

---

# 20. Readiness checklist and quick reference

## Before a test

- [ ] Objective and acceptance criteria are explicit.
- [ ] Workload model matches the intended traffic.
- [ ] Environment is authorized and appropriately isolated.
- [ ] Data is representative and cleanup is planned.
- [ ] Script passes low-load functional validation.
- [ ] Authentication and dynamic correlation work.
- [ ] Generator capacity has been checked.
- [ ] Metrics, traces, and logs are available.
- [ ] Stop conditions and owners are agreed.
- [ ] Build, tool version, configuration, and data version are recorded.

## During a test

- [ ] Offered load and completed throughput are visible.
- [ ] p50/p95/p99 and errors are monitored.
- [ ] Generator CPU, memory, network, and dropped work are healthy.
- [ ] Service, DB, cache, queue, and dependency metrics are correlated.
- [ ] Test phases and deployments are annotated.
- [ ] Safety stop conditions are enforced.
- [ ] Business side effects are checked.

## After a test

- [ ] Validate run integrity before interpreting results.
- [ ] Compare every objective with observed values.
- [ ] Identify the first failing criterion.
- [ ] Correlate results with telemetry.
- [ ] Separate facts from hypotheses.
- [ ] Document limitations and anomalies.
- [ ] Save scripts, raw data, configuration, and reports.
- [ ] Assign prioritized actions and retest with the same workload.

## Quick formula sheet

| Concept | Formula / rule |
|---|---|
| Error rate | Failed ÷ attempted × 100% |
| Relative change | (New − baseline) ÷ baseline × 100% |
| Little's Law | Average in-system work = arrival rate × average time in system |
| Approx. concurrency | Throughput × average response time |
| Capacity | Highest sustainable rate meeting all defined objectives |
| Regression decision | Consider absolute SLO, relative change, sample size, and run variability |

## Strong 60-second interview answer

> “I start by translating performance expectations into measurable criteria: workload, latency percentiles, throughput, error rate, duration, and correctness. I validate the script at low load, confirm that the generator and environment are representative, and establish a baseline. I then ramp to target load while correlating client metrics with service, database, cache, queue, and dependency telemetry. If an objective fails, I identify the first constrained component and test a specific hypothesis using traces, profiling, or query plans. After tuning, I rerun the same workload, verify business correctness, and report the evidence, limitations, and remaining risk.”

## Glossary

- **Arrival rate:** rate at which new work is offered.
- **Backpressure:** mechanism that slows or rejects upstream work when downstream capacity is constrained.
- **Baseline:** controlled reference run.
- **Capacity:** sustainable workload within objectives.
- **Coordinated omission:** bias caused by failing to record delays while the generator is stalled.
- **Headroom:** spare capacity above expected demand.
- **Idempotency:** repeated operation has the intended same effect rather than duplicate side effects.
- **NFR:** non-functional requirement.
- **Queue depth:** amount of work waiting to be processed.
- **Saturation:** resource or queue near its effective limit.
- **SLI/SLO/SLA:** indicator / objective / agreement for service behavior.
- **Think time:** simulated user pause between actions.
- **VU:** virtual user in a load-testing tool.

## Further learning

Use official documentation as the source of truth for version-specific behavior:

- Apache JMeter: https://jmeter.apache.org/usermanual/
- Grafana k6: https://grafana.com/docs/k6/latest/
- Google SRE Book: https://sre.google/sre-book/table-of-contents/
- Google SRE Workbook: https://sre.google/workbook/table-of-contents/
- OpenTelemetry: https://opentelemetry.io/docs/
- Prometheus: https://prometheus.io/docs/
- Apache Kafka: https://kafka.apache.org/documentation/
- Web Vitals: https://web.dev/articles/vitals

---

## Final principle

A performance test is trustworthy only when the workload is realistic, the generator is healthy, measurements are correctly defined, the environment is understood, business correctness is verified, and conclusions are supported by evidence. The strongest performance engineer is not the person who generates the most traffic; it is the person who turns performance data into a defensible engineering decision.
