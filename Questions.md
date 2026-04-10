## MICROSERVICES Q&As

### Q1 : What do you mean by read‑only microservice in this context?

✅ Ideal Answer:
By read‑only, I mean the service does not modify or influence transaction data or money movement. It only consumes transaction events, stores immutable records for analytics, and exposes REST APIs to read aggregated insights for dashboards, reporting, and compliance. This ensures analytics logic never interferes with critical transaction processing.

### Q2 : What problem was this service solving?

✅ Ideal Answer:
The problem was to provide near‑real‑time financial insights—such as transaction volumes, success/failure rates, and merchant performance—without impacting core payment systems. Business, operations, and compliance teams needed fast and reliable analytics, and this service enabled that through an event‑driven, decoupled design.

### Q3 : What do you mean by event‑driven here?

✅ Ideal Answer:
Event‑driven means the service reacts to transaction lifecycle events asynchronously instead of making synchronous calls. Core systems emit events like transaction initiated or completed, and this service consumes those events via a message queue to process analytics independently.

### Q4 : Why did you choose an asynchronous event‑driven approach instead of API calls?

✅ Ideal Answer:
Using async events avoids tight coupling with transaction systems and ensures analytics processing never adds latency to payments. It also improves scalability, fault isolation, and reliability, since failures in analytics don’t affect transaction execution.


### Q5 : How did you ensure the metrics were near‑real‑time?

✅ Ideal Answer:
Events were consumed as soon as they were published, and metrics were incrementally aggregated during ingestion rather than computed on demand. This allowed dashboards to reflect updates within seconds of a transaction completing, while still remaining eventually consistent.

### Q6 : What kind of data did this microservice ingest and store?

✅ Ideal Answer:
It ingested transaction lifecycle events containing transaction ID, amount, currency, status, merchant ID, region, and timestamp. The service stored immutable transaction facts in a raw metrics table and maintained pre‑aggregated summaries for faster reads.


### Q7 : How do you prevent duplicate events from corrupting analytics metrics?

✅ Ideal Answer:
We implemented idempotency at the consumer level using a unique constraint on transaction ID and event type. If the same event is received more than once, the duplicate insert fails gracefully, ensuring aggregates are not double‑counted.

### Q8 : Why not calculate analytics directly from the raw transactions table at query time?

✅ Ideal Answer:
Query‑time aggregation would require frequent full‑table scans or large joins, which doesn’t scale under high transaction volumes. Pre‑aggregating metrics during ingestion allows O(1) reads for dashboards and keeps database load predictable and low.

### Q9 : How does this design support compliance and auditing needs?

✅ Ideal Answer:
By storing immutable, append‑only transaction facts, the system maintains a complete historical record that can be audited or replayed. Aggregates can always be regenerated from raw data, ensuring traceability and data consistency for compliance reporting.

### Q10 : What happens if this analytics service goes down during peak transaction hours?

✅ Ideal Answer:
Nothing impacts transactions. Core systems continue processing payments normally. Events remain queued in the message broker and are processed once the analytics service recovers. This failure isolation is a key benefit of the event‑driven design.

### Q11. What do you mean by asynchronous ingestion?

✅ Answer:
Asynchronous ingestion means the analytics service consumes transaction events independently without blocking or waiting for the core transaction flow. Events are published to RabbitMQ, and the analytics service processes them later at its own pace, ensuring no impact on transaction latency.

### Q12. Why did you use events instead of direct API calls?

✅ Answer:
Direct API calls would tightly couple analytics to transaction systems, introducing latency and failure dependency. Events allow loose coupling, fault isolation, and scalability—analytics failures don’t affect transaction execution.

### Q13. What are transaction lifecycle events?

✅ Answer:
They represent different states of a transaction, such as TransactionInitiated, TransactionCompleted, and TransactionFailed, allowing downstream systems like analytics to understand how a transaction progressed without querying core systems.

### Q14. Why were only initiated, completed, and failed events used?

✅ Answer:
These events capture the essential business lifecycle: when a transaction starts, when it succeeds, and when it fails. They are sufficient to derive volume, success rates, failure analysis, and financial summaries without overloading the system.

### Q15. Why RabbitMQ specifically?

✅ Answer:
RabbitMQ is well‑suited for reliable event delivery, flexible routing, and moderate‑to‑high throughput use cases. It integrates easily with Spring, supports acknowledgments and retries, and fits analytics ingestion patterns well.

### Q16. How does this design protect core transaction systems?

✅ Answer:
The transaction system only publishes an event and moves on. Even if analytics is slow or down, RabbitMQ buffers messages and retries delivery later, ensuring transaction processing remains unaffected.

### Q17. What happens if analytics service goes down?

✅ Answer:
Transactions continue normally. Events stay in RabbitMQ and are processed once the service recovers. No transaction data is lost, and no upstream systems are blocked.

### Q18. How does RabbitMQ fit into the decoupling?

✅ Answer:
RabbitMQ acts as an intermediary. Producers don’t know who consumes the events, and consumers don’t know who produced them. This removes direct dependencies and allows independent scaling and deployment.

### Q19. How do you ensure messages are not lost?

✅ Answer:
We use durable exchanges and queues, persistent messages, and manual acknowledgments. Messages are acknowledged only after successful database persistence.

### Q20. How do you handle duplicate events?

✅ Answer:
We enforce idempotency at the database level using a unique constraint on transaction_id and event_type. Duplicate messages are safely ignored without affecting aggregates.

### Q21. How do you handle message failures?

✅ Answer:
On processing failures, the message is retried. After repeated failures, it’s routed to a Dead Letter Queue (DLQ) for inspection and correction without blocking the pipeline.

### Q22. How do events flow from core systems to analytics?

✅ Answer:
Core systems publish events to a RabbitMQ exchange, which routes them to analytics queues based on routing keys. The analytics consumer processes and persists them asynchronously.

### Q23. Why not expose analytics directly from transaction DBs?

✅ Answer:
Querying transactional databases for analytics would introduce load, slow down transactions, and risk data contention. Dedicated analytics storage keeps operational systems fast and stable.

### Q24. How does eventual consistency apply here?

✅ Answer:
Analytics data may lag slightly behind real‑time transactions, but it eventually reflects the correct state once all events are processed. This trade‑off enables scalability and safety.

### Q25. How do you scale this ingestion pipeline?

✅ Answer:
The analytics service is stateless, so we scale horizontally by adding more consumers. RabbitMQ distributes messages across consumers, allowing parallel processing during peak traffic.

### Q26. How do you maintain ordering guarantees?

✅ Answer:
Ordering is not strictly required globally, but per‑transaction consistency is maintained using transaction IDs. The final state is derived correctly even if events arrive slightly out of order.

### Q27. Why not Kafka instead of RabbitMQ?

✅ Answer:
RabbitMQ suits moderate throughput and routing‑based patterns with simpler operational overhead. Kafka would be preferred for very high throughput, long‑term retention, or stream processing needs.

### Q28. How do you rebuild analytics data if logic changes?

✅ Answer:
Because raw events are stored as immutable facts, we can replay them through the consumer to recompute aggregates without touching transaction systems.

### Q29. How do you ensure analytics never slows down payments?

✅ Answer:
There is no synchronous dependency. The transaction flow ends once an event is published. Analytics processing happens independently and cannot impact transaction latency or success.

### Q30. What would break if this decoupling didn’t exist?

✅ Answer:
Failures or slowness in analytics could block transactions, cause cascading outages, increase latency, and violate financial SLAs—making the system unsafe for fintech workloads.


### Q31 : What do you mean by an idempotent consumer?

✅ Ideal Answer:
An idempotent consumer ensures that processing the same event multiple times produces the same outcome as processing it once. Even if duplicate messages are delivered, the system state remains correct and consistent.

### Q32 : Why do you need idempotency in message‑driven systems?

✅ Ideal Answer:
Because message brokers can deliver duplicate events due to retries, network failures, or consumer crashes. Without idempotency, duplicates can corrupt data, especially in financial systems.

### Q33 : What is meant by duplicate‑event handling?

✅ Ideal Answer:
Duplicate‑event handling refers to detecting and safely ignoring already processed events, ensuring they don’t cause double inserts or double aggregation in downstream systems.

### Q34 : What kind of problems can duplicates cause in analytics systems?

✅ Ideal Answer:
Duplicates can inflate transaction counts, overstate financial volume, skew success/failure ratios, and cause incorrect business or compliance reports.

### Q35 : What does schema validation mean here?

✅ Ideal Answer:
Schema validation ensures incoming events adhere to a predefined structure and data contract, such as expected fields, data types, and allowed values, before processing them.


### Q36 : How did you implement idempotency in your consumer?

✅ Ideal Answer:
Using a unique business key—typically a combination of transaction ID and event type—enforced through a database unique constraint. If a duplicate event is processed, the insert fails gracefully and the event is ignored.

### Q37 : Why is database‑level idempotency preferred over in‑memory checks?

✅ Ideal Answer:
Database‑level idempotency is durable and works across restarts and multiple consumer instances. In‑memory checks fail in distributed systems or when consumers restart.

### Q38 : What schema validation strategies did you use?

✅ Ideal Answer:
We validated required fields, data types, and enum values at the consumer boundary—often using JSON schema or DTO validation—before business processing or database writes.

### Q39 : What happens if an event fails schema validation?

✅ Ideal Answer:
Invalid events are rejected and sent to a Dead Letter Queue for inspection, preventing corrupt or malformed data from entering the analytics pipeline.

### Q40 : How do retries interact with idempotency?

✅ Ideal Answer:
Retries may resend the same event multiple times, but because the consumer is idempotent, retries do not cause duplicate state changes, making retries safe.


### Q41 : You mentioned exactly‑once processing semantics. How did you achieve that?

✅ Ideal Answer:
True exactly‑once isn’t guaranteed by most brokers, so we achieved effectively‑once semantics by combining at‑least‑once delivery with idempotent consumers and transactional database writes.

### Q42 : Why is exactly‑once hard to achieve in distributed systems?

✅ Ideal Answer:
Because partial failures, network partitions, and crashes can occur after message delivery but before acknowledgment. Coordinating state changes across distributed components without duplication is inherently complex.

### Q43 : How does your design support safe event replays?

✅ Ideal Answer:
Since consumers are idempotent and raw data is append‑only, events can be replayed from the broker or event store without risking double counting or data corruption.

### Q44 : What changes, if any, are required before replaying events?

✅ Ideal Answer:
Typically, aggregated tables may be truncated or recomputed, but raw transaction tables remain unchanged, ensuring deterministic rebuilds.

### Q45 : How do you handle high throughput while maintaining idempotency?

✅ Ideal Answer:
By using efficient database constraints, indexed unique keys, batch processing, and tuned consumer concurrency so idempotency checks remain O(1) and don’t become a bottleneck.

### Q46 : Can idempotency alone guarantee data correctness?

✅ Ideal Answer:
No. Idempotency prevents duplicates, but correctness also depends on schema validation, correct event ordering assumptions, and consistent aggregation logic.

### Q47 : How do you deal with out‑of‑order events?

✅ Ideal Answer:
Analytics systems are designed to be eventually consistent. Aggregations rely on event timestamps, and since multiple lifecycle events are allowed per transaction, ordering isn’t strictly enforced at ingestion time.

### Q48 : What are the risks if idempotency is implemented incorrectly?

✅ Ideal Answer:
Incorrect idempotency can silently corrupt metrics, cause financial misreporting, and break compliance requirements—often without immediate visibility.

### Q49 : Why is this design especially important in fintech systems?

✅ Ideal Answer:
Fintech systems require high data accuracy, auditability, and fault tolerance. Duplicate or inconsistent analytics can lead to wrong financial decisions or regulatory violations.

### Q50 : How would this design differ if this were a transaction‑processing service instead of analytics?

✅ Ideal Answer:
For transaction processing, stricter guarantees, distributed transactions, and stronger consistency controls would be required. For analytics, idempotent event consumption with eventual consistency is the safer and more scalable choice.

### Q51 : What do you mean by “immutable transaction facts”?

✅ Answer:
Immutable transaction facts are records that, once written, are never updated or deleted. Each record represents a factual event that occurred at a specific time, ensuring the data always reflects what actually happened.

### Q52 : Why is immutability important for transaction data?

✅ Answer:
Immutability ensures data integrity, auditability, and trust. In financial systems, historical transaction data must not change, as it may be used for audits, dispute resolution, and compliance checks.

### Q53 : What does “append‑only” mean in database terms?

✅ Answer:
Append‑only means that data is only inserted into the table. Existing rows are never updated or deleted, making write operations predictable and avoiding accidental data loss or corruption.

### Q54: Why did you choose PostgreSQL for this storage?

✅ Answer:
PostgreSQL provides strong ACID guarantees, excellent indexing support, mature tooling, and reliability, making it well‑suited for storing critical financial data with consistency and durability.

### Q55 : What kind of data is stored in this raw metrics table?

✅ Answer:
It stores transaction identifiers, amounts, currencies, statuses, merchant information, region, and event timestamps—essentially the core transactional facts required for analytics and reporting.

### Q56. Why not update the record when transaction status changes?

✅ Answer:
Updating records would overwrite historical states. By storing each lifecycle event as a separate immutable record, we preserve the full transaction history and make downstream analytics more reliable.

### Q57. How does this help historical analysis?

✅ Answer:
Because all events are retained permanently, we can analyze trends over time, recompute metrics for past periods, and answer questions that weren’t originally anticipated when the data was stored.

### Q58 : How does this design support auditability?

✅ Answer:
Auditors can trace every transaction’s full lifecycle by reviewing immutable records. Since data isn’t modified, audits can trust the accuracy and completeness of historical records.

### Q59 : How do you prevent duplicate transaction records?

✅ Answer:
We use a unique constraint on key fields like transaction_id and event_type. If a duplicate event arrives, the insert fails safely without impacting the dataset.

### Q60 : How does this design enable replayability?

✅ Answer:
Since raw transaction events are stored permanently, we can reprocess them to recompute aggregates if business logic changes or analytics tables need rebuilding.

### Q61 : What performance benefits does append‑only storage provide?

✅ Answer:
Append‑only writes are fast because they avoid row locking and in‑place updates. This makes the system efficient under high write throughput.

### Q62 :  How do you query large append‑only tables efficiently?

✅ Answer:
We use time‑based indexes and merchant‑based indexes, allowing efficient filtering by date ranges and identifiers without scanning the entire table.

### Q63 : Why not store only aggregated data instead of raw facts?

✅ Answer:
Aggregates alone lose detail. Raw facts allow rebuilding aggregates, validating metrics, and answering new analytical questions without data loss.

### Q64 : How do you handle schema evolution over time?

✅ Answer:
We version events and make schema changes backward‑compatible by adding nullable fields or new tables, ensuring old data remains valid and usable.

### Q65 : How does immutability simplify system design?

✅ Answer:
Immutability reduces complexity by eliminating update conflicts, rollback scenarios, and race conditions. Systems become easier to reason about and debug.

### Q66 : How does this design behave under very high transaction volume?

✅ Answer:
Append‑only inserts scale well under load. Combined with partitioning and indexing, PostgreSQL can efficiently handle large write volumes without degrading performance.

### Q67 : Why not use NoSQL or event stores instead?

✅ Answer:
PostgreSQL provides strong consistency, relational querying, and mature tooling. Given the need for structured queries and reporting, it was a practical and reliable choice.

### Q68 : How do you manage storage growth for immutable data?

✅ Answer:
We use table partitioning by date and retention policies. Older partitions can be archived, compressed, or moved to cold storage without impacting recent queries.

### Q69 : How does this enable compliance and legal investigations?

✅ Answer:
Because every transaction event is preserved, investigators can reconstruct exact transaction timelines, verify outcomes, and prove data integrity during disputes or audits.

### Q70 : What problems would arise if the data were mutable instead?

✅ Answer:
Mutable data could hide historical states, introduce inconsistencies, complicate audits, and make it impossible to confidently replay or verify past transaction behavior.

### Q71 : What do you mean by “incremental aggregation”?

✅ Answer:
Incremental aggregation means updating aggregate metrics continuously as new events arrive, instead of recomputing totals by scanning the entire dataset each time.

### Q72 : What are “pre‑aggregated daily financial summaries”?

✅ Answer:
They are daily snapshots of key metrics like total transaction volume, total transactions, and success/failure counts, stored in advance to support fast queries.

### Q73 : Why aggregate data daily instead of per request?

✅ Answer:
Aggregating per request would require expensive database scans, increasing latency and load. Pre‑aggregating daily data allows constant‑time reads and fast dashboard responses.

### Q74 : What metrics were you aggregating?

✅ Answer:
Total transaction volume, number of successful transactions, number of failed transactions, and total transaction count per day.

### Q75 : How does this enable millisecond‑level queries?

✅ Answer:
Dashboards query a small, pre‑computed summary table instead of large raw datasets, drastically reducing disk scans and computation time.

### Q76 : Where does the input for aggregation come from?

✅ Answer:
The input comes from transaction lifecycle events consumed asynchronously from RabbitMQ and persisted as raw immutable transaction facts.

### Q77 : What happens if no aggregation pipeline exists?

✅ Answer:
Dashboards would run slow queries directly on raw transaction data, leading to poor performance and potential impact on the analytics database.

### Q78 : How does incremental aggregation work at a high level?

✅ Answer:
Each incoming transaction event updates the corresponding daily summary row by increasing counters and totals, ensuring aggregates stay current without full recomputation.

### Q79 : How do you ensure aggregates stay correct with duplicate events?

✅ Answer:
We enforce idempotency at the raw data layer using unique constraints. Aggregation is only performed after a successful raw insert, preventing double counting.

### Q80 Why not aggregate directly from raw data on a schedule?

✅ Answer:
Scheduled aggregation introduces delays and heavy database scans. Incremental aggregation spreads computation over time and keeps summaries near real‑time.

### Q81 : How do you handle late‑arriving events?

✅ Answer:
Late events are still processed based on their event date and update the appropriate daily summary, ensuring historical correctness.

### Q82 : What database operations are used during aggregation?

✅ Answer:
We use atomic upserts (INSERT … ON CONFLICT DO UPDATE) to safely update daily summaries in a single transaction.

### Q83 : How do you prevent race conditions during aggregation updates?

✅ Answer:
Database‑level atomic operations and row‑level locking ensure concurrent updates to the same daily summary are handled safely.

### Q84 : How does this design benefit dashboard teams?

✅ Answer:
It provides fast, predictable APIs that return results instantly, allowing dashboards to remain responsive even under heavy user load.

### Q85 : How does incremental aggregation scale with increasing traffic?

✅ Answer:
Aggregation cost is distributed across individual events instead of centralized batch jobs, allowing horizontal scaling with increased event volume.

### Q86 : How does this design behave under traffic spikes?

✅ Answer:
RabbitMQ buffers incoming events, and consumers process them at a controlled rate. Aggregation keeps up without overwhelming the database.

### Q87 : What happens if aggregation logic has a bug?

✅ Answer:
Since raw events are stored immutably, we can fix the logic and recompute aggregates by replaying historical data.

### Q88 : Why not use real‑time stream processing instead?

✅ Answer:
Stream processing adds complexity. Incremental DB‑based aggregation was sufficient for near real‑time needs with lower operational overhead.

### Q89 : How does this design support compliance and audits?

✅ Answer:
Aggregates are derived from immutable raw data, so auditors can always verify summary numbers by tracing back to original events.

### Q90 : What are the risks if aggregation logic is incorrect?

✅ Answer:
Incorrect financial metrics could mislead business decisions or reports. That’s why immutability, replayability, and validation checks are critical safeguar

### Q91 : What do you mean by “read‑only REST APIs”?

✅ Answer:
Read‑only REST APIs only expose data retrieval endpoints like GET and do not allow data creation, updates, or deletion, ensuring analytics consumers cannot modify financial data.

### Q92 Why are analytics APIs exposed as read‑only?

✅ Answer:
Analytics data represents derived financial insights and must remain protected from external modification to preserve correctness, auditability, and consistency.

### Q93 : What does it mean for an API to be stateless?

✅ Answer:
Stateless means each request contains all the information required to process it, and the server does not store any client session state between requests.

### Q94 : Why is statelessness important for dashboards?

✅ Answer:
Stateless APIs can scale horizontally, allowing multiple instances behind a load balancer to serve high dashboard traffic without session affinity.

### Q95 : What kind of clients consume these APIs?

✅ Answer:
Operational dashboards, business intelligence tools, finance teams, reporting services, and compliance systems consume these APIs.

### Q96 : What is meant by “daily summaries”?

✅ Answer:
Daily summaries are aggregated metrics per day such as total transaction volume, transaction counts, and success/failure ratios.

### Q97 : What are “merchant‑level insights”?

✅ Answer:
Merchant‑level insights provide performance metrics per merchant, including transaction volume, success rate, regional distribution, and historical trends.

### Q98 : Why expose separate APIs for daily summaries and merchant insights?

✅ Answer:
They serve different use cases—daily summaries power operational dashboards, while merchant insights support performance analysis and risk evaluations. Separation keeps APIs focused and efficient.

### Q99 : How are these APIs optimized for high traffic?

✅ Answer:
They query pre‑aggregated tables, return compact responses, avoid joins on large datasets, and use indexed access paths for predictable low latency.

### Q100 How do you ensure consistent response times under load?

✅ Answer:
By querying small summary tables, tuning database indexes, using connection pooling, and keeping APIs stateless for horizontal scaling.

### Q101 Why not expose raw transaction data directly?

✅ Answer:
Raw data is large and sensitive. Exposing it would increase latency, load, and risk, while dashboards typically require aggregated insights, not granular records.

### Q102 How do clients filter data in these APIs?

✅ Answer:
Filters are passed as query parameters like date ranges, merchant IDs, or regions, allowing flexible yet controlled data access.

### Q103 : How do you validate incoming API requests?

✅ Answer:
We validate query parameters for format, range, and business rules, returning appropriate error responses for invalid inputs.

### Q104 : How do you version these APIs?

✅ Answer:
We use URL‑based or header‑based versioning to introduce changes without breaking existing dashboard consumers.

### Q105 : How does stateless API design enable horizontal scaling?

✅ Answer:
Because no session state is stored on the server, any request can be handled by any instance, making scaling out straightforward using load balancers.

### Q106 : How do you prevent APIs from becoming a bottleneck?

✅ Answer:
By using pre‑aggregated data, caching frequently accessed responses, and scaling API instances independently from ingestion pipelines.

### Q107 : How do you secure read‑only analytics APIs?

✅ Answer:
We use authentication, role‑based authorization, and network‑level controls to restrict access while keeping APIs read‑only.

### Q108 : How do these APIs support real‑time dashboards?

✅ Answer:
Aggregates are updated incrementally during ingestion, so dashboards receive near‑real‑time data without heavy computation during reads.

### Q109 : What happens if dashboard traffic suddenly spikes?

What they’re testing:
Burst handling.

✅ Answer:
Stateless APIs scale horizontally, and the database handles predictable loads through indexed queries, ensuring stable performance during spikes.

### Q110 : What issues would arise if APIs were stateful or write‑enabled?

✅ Answer:
Statefulness would limit scalability, and write access could corrupt analytics data, break compliance guarantees, and introduce security risks.

### Q111 : What performance problem were you trying to solve here?

✅ Ideal Answer:
The system needed to handle very high write volumes and still support fast read queries for analytics dashboards. Without optimization, inserts would be slow and analytical queries over large datasets would degrade significantly.

### Q112: What do you mean by time‑based indexing?

✅ Ideal Answer:
Time‑based indexing means creating indexes on timestamp or date columns—like event_time—so that queries filtering by date range can quickly locate relevant rows without scanning the entire table.

### Q113 : Why is time‑based indexing especially important in analytics systems?

✅ Ideal Answer:
Most analytics queries are time‑bounded, such as daily, weekly, or monthly reports. Indexing on time ensures these queries remain fast as historical data grows.

### Q114 : What is a merchant‑based index?

✅ Ideal Answer:
A merchant‑based index is an index on merchant_id, which allows fast retrieval of transactions related to a specific merchant—commonly required for merchant performance dashboards or risk analysis.

### Q115 : What do you mean by batch inserts?

✅ Ideal Answer:
Batch inserts mean inserting multiple records in a single database operation instead of inserting one row at a time, significantly reducing network overhead and transaction costs.

### Q116 : Why not rely on a single composite index instead of multiple indexes?

✅ Ideal Answer:
Different query patterns require different indexes. Time‑range queries, merchant‑specific queries, and status‑based queries benefit from separate or targeted composite indexes. A single index rarely serves all access patterns efficiently.

### Q117 : How did you decide which fields to index?

✅ Ideal Answer:
By analyzing query patterns—fields frequently used in WHERE clauses, GROUP BY clauses, and joins. Time, merchant, and status fields were consistently used in analytics queries, making them strong candidates.

### Q118 : What is the downside of adding too many indexes?

✅ Ideal Answer:
Indexes increase write overhead because every insert or update must also update the index. They also consume storage, so we only added indexes that delivered clear query‑performance benefits.

### Q119 : How do batch inserts help under high throughput?

✅ Ideal Answer:
They reduce the number of database round trips and transaction commits, allowing the system to process thousands of records efficiently while keeping latency low.

### Q120 : Did batch inserts affect idempotency or correctness?

✅ Ideal Answer:
No, because idempotency was enforced at the database level using unique constraints. Even in batch inserts, duplicate rows would be rejected safely.

### Q121 : How do indexes behave as the table grows to millions of records?

✅ Ideal Answer:
Indexes grow in size, but proper indexing ensures logarithmic lookup time. Combined with partitioning and selective indexes, query performance remains stable even with large datasets.

### Q122 : Why not use real‑time aggregation instead of optimizing raw tables?

✅ Ideal Answer:
Even with aggregates, raw transaction data must be stored efficiently for audit, replay, and recomputation. Optimizing raw tables ensures both rebuildability and sustained write performance.

### Q123 : How does batch insert size affect system performance?

✅ Ideal Answer:
Very small batches don’t reduce overhead significantly, while extremely large batches increase memory usage and transaction duration. We tune batch size to balance throughput and system stability.

### Q124 : What happens if batch inserts fail midway?

✅ Ideal Answer:
Batch inserts are typically wrapped in transactions. If a failure occurs, the transaction is rolled back, ensuring partial inserts don’t corrupt the database.

### Q125 : How does indexing affect write‑heavy systems like this?

✅ Ideal Answer:
Indexes slightly slow down writes due to maintenance overhead, but carefully chosen indexes provide a net performance gain because read efficiency is critical for analytics workloads.

### Q126 : Would you ever drop indexes temporarily?

✅ Ideal Answer:
Yes, during large backfills or replays, indexes can be dropped and recreated afterward to significantly speed up bulk loading.

### Q127 : How does partitioning complement time‑based indexing?

✅ Ideal Answer:
Partitioning physically separates data—such as monthly partitions—so queries automatically scan only relevant partitions, further reducing I/O and improving performance beyond what indexes alone provide.

### Q128 : How do these optimizations help during peak traffic?

✅ Ideal Answer:
Batch inserts absorb high write bursts efficiently, while indexes keep read queries predictable and fast, preventing dashboards and reports from slowing down during peak transaction hours.

### Q129 : What metrics would you monitor to ensure database performance stays healthy?

✅ Ideal Answer:
Query latency, index hit ratio, write throughput, lock contention, slow query logs, and table growth trends.

Q20 (Advanced)

Interviewer: How would this approach differ if writes were transactional instead of analytics‑only?

✅ Ideal Answer:
Transactional systems would require stricter consistency guarantees and potentially fewer indexes to minimize write latency. Analytics systems favor read optimization and tolerate eventual consistency.

Q1 (Basic)

Interviewer: What kind of failures are you referring to here?

✅ Ideal Answer:
Failures like message processing errors, temporary database outages, malformed events, schema mismatches, or unexpected runtime exceptions in the analytics service.

Q2 (Basic)

Interviewer: Why is failure handling important in an analytics service?

✅ Ideal Answer:
Because analytics systems handle high volumes of asynchronous data, and failures are inevitable. Proper handling ensures system stability, data correctness, and prevents failures from cascading to upstream systems.

Q3 (Basic)

Interviewer: What do you mean by retry mechanisms?

✅ Ideal Answer:
Retry mechanisms automatically reattempt message processing when transient failures occur, such as temporary network or database issues, instead of immediately discarding the event.

Q4 (Basic)

Interviewer: What is a Dead Letter Queue (DLQ)?

✅ Ideal Answer:
A DLQ is a separate queue where messages are sent after repeated processing failures, allowing them to be isolated, inspected, and handled without blocking the main processing flow.

Q5 (Basic)

Interviewer: Why do you need a DLQ at all?

✅ Ideal Answer:
Because some failures are permanent, like invalid schemas or corrupt data. Retrying such messages endlessly would block the pipeline, so DLQs allow safe isolation and manual intervention.

✅ MEDIUM LEVEL (Design & Implementation)

Q6 (Medium)

Interviewer: How do retries work in your system?

✅ Ideal Answer:
When a message processing attempt fails, it is retried automatically with a limited retry count, usually using broker‑level retry or consumer‑level retry logic. Only after retries are exhausted is the message sent to the DLQ.

Q7 (Medium)

Interviewer: What kind of failures should be retried?

✅ Ideal Answer:
Transient failures like database timeouts, temporary network issues, or short‑lived service unavailability should be retried, as they often resolve on their own.

Q8 (Medium)

Interviewer: What failures should not be retried?

✅ Ideal Answer:
Permanent failures such as invalid payloads, schema violations, missing mandatory fields, or logically incorrect data should not be retried and should go directly to the DLQ.

Q9 (Medium)

Interviewer: How many retries are usually safe?

✅ Ideal Answer:
Retries should be limited—often 3 to 5 attempts—to avoid infinite retry loops. The exact number depends on system latency, throughput requirements, and failure patterns.

Q10 (Medium)

Interviewer: How do retries interact with idempotency?

✅ Ideal Answer:
Retries can cause duplicate deliveries, so idempotent processing ensures retries don’t result in duplicate data or incorrect aggregates.

✅ ADVANCED LEVEL (Reliability, Isolation & Production Thinking)

Q11 (Advanced)

Interviewer: How did you guarantee analytics failures never impacted transaction processing?

✅ Ideal Answer:
By designing the analytics service to be fully asynchronous and decoupled from transaction systems. Transactions emit events but never wait for analytics to succeed, so failures in analytics cannot affect transaction execution.

Q12 (Advanced)

Interviewer: What happens if the analytics service is completely down?

✅ Ideal Answer:
Transaction systems continue operating normally. Events remain buffered in the message broker and are processed once the analytics service comes back online.

Q13 (Advanced)

Interviewer: How do you avoid message loss during failures?

✅ Ideal Answer:
Messages are acknowledged only after successful processing and database commits. Failed attempts are retried, and persistent failures are stored in a DLQ rather than dropped.

Q14 (Advanced)

Interviewer: What metadata is useful in DLQ messages?

✅ Ideal Answer:
Original payload, error reason, stack trace, retry count, timestamp, and correlation IDs to trace the message back to its source.

Q15 (Advanced)

Interviewer: How do you handle DLQ messages operationally?

✅ Ideal Answer:
DLQ messages are monitored via alerts. Engineers inspect them to identify root causes, fix bugs or schema issues, and replay messages when appropriate.

Q16 (Advanced)

Interviewer: How is replay done safely without corrupting data?

✅ Ideal Answer:
Because consumers are idempotent and raw data is append‑only, replaying messages does not create duplicates or inconsistencies.

Q17 (Advanced)

Interviewer: What is the risk of aggressive retries?

✅ Ideal Answer:
Aggressive retries can overwhelm downstream systems, increase load during outages, and cause retry storms. Controlled retries with backoff are safer.

Q18 (Advanced)

Interviewer: Did you use exponential backoff? Why?

✅ Ideal Answer:
Yes, exponential backoff reduces pressure on failing dependencies by gradually increasing retry intervals, improving overall system stability.

Q19 (Advanced)

Interviewer: How do you monitor failure handling effectiveness?

✅ Ideal Answer:
By tracking retry counts, DLQ size, processing latency, consumer error rates, and alerting when thresholds are breached.

Q20 (Advanced)

Interviewer: Why is this pattern particularly critical in fintech systems?

✅ Ideal Answer:
Because fintech systems require high availability and strict isolation. Analytics errors must never delay or block money movement, and robust failure handling ensures regulatory safety and operational resilience.

