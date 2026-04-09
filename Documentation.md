# Complete Technical Documentation (Backend Engineer Perspective)

# Transaction Analytics & Financial Insights Platform

---

## 1. System Overview (What I Built)

I implemented a **read-only analytics platform** consisting of:

- Event-driven ingestion services  
- Aggregation & computation modules  
- Read-optimized REST APIs  
- PostgreSQL analytical storage  
- RabbitMQ-based ingestion pipeline  

The system **observes transactions** but **never affects money movement**.

---

## 2. Microservice Breakdown

Although deployed as a **single logical bounded context**, the system follows **modular microservice principles**.

### Logical Components

1. Transaction Event Ingestion Service  
2. Aggregation & Computation Module  
3. Analytics API Service  
4. Rebuild / Replay Job  
5. Infrastructure Components (RabbitMQ, PostgreSQL)  

---

## 3. RabbitMQ Event Ingestion Flow

### 3.1 Why RabbitMQ Was Used

- At-least-once delivery  
- Back-pressure handling  
- Decoupling from transaction systems  
- Retry + DLQ support  

### 3.2 Exchange & Queue Design

```
Exchange: transaction.events.exchange (topic)

Routing Keys:
- transaction.initiated
- transaction.completed
- transaction.failed

Queue:
- analytics.transaction.events.queue
- analytics.transaction.events.dlq
```

### 3.3 Producer → Consumer Flow

1. Upstream service publishes event  
2. Event routed via topic exchange  
3. Analytics queue receives event  
4. Consumer processes & ACKs  
5. Failure → retry → DLQ  

---

## 4. Event Ingestion Microservice

### 4.1 Responsibilities

- Consume transaction lifecycle events  
- Enforce schema validation  
- Ensure idempotency  
- Persist immutable transaction facts  
- Trigger aggregation updates  

### 4.2 Event Payload

```json
{
  "eventType": "TransactionCompleted",
  "transactionId": "txn_872342",
  "amount": 1500.75,
  "currency": "INR",
  "status": "SUCCESS",
  "merchantId": "mer_129",
  "region": "IN",
  "eventTime": "2026-03-21T10:40:12Z"
}
```

### 4.3 Consumer Implementation

```java
@RabbitListener(queues = "analytics.transaction.events.queue")
public void consume(TransactionEvent event) {
    validate(event);

    if (idempotencyCheck(event)) {
        persistRawMetric(event);
        updateAggregates(event);
    }
}
```

### 4.4 Idempotency Strategy

**Why required:** RabbitMQ guarantees *at-least-once*, not *exactly-once*

**Technique Used:**
- DB-level unique constraint  
- `(transaction_id, status)`

```sql
ALTER TABLE transaction_metrics
ADD CONSTRAINT uq_tx_event UNIQUE (transaction_id, status);
```

Duplicate events:
- Safely ignored  
- Enables replay  

---

## 5. Database Design (PostgreSQL)

### 5.1 Raw Metrics Table

```sql
CREATE TABLE transaction_metrics (
  id BIGSERIAL PRIMARY KEY,
  transaction_id VARCHAR(64) NOT NULL,
  amount NUMERIC(18,2),
  currency VARCHAR(3),
  status VARCHAR(20),
  merchant_id VARCHAR(50),
  region VARCHAR(20),
  event_time TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT now()
);
```

### 5.2 Indexing Strategy

```sql
CREATE INDEX idx_tx_event_time ON transaction_metrics(event_time);
CREATE INDEX idx_tx_merchant ON transaction_metrics(merchant_id);
CREATE INDEX idx_tx_status ON transaction_metrics(status);
```

Optimized for:
- Date-range analytics  
- Merchant dashboards  
- Status distribution  

Append-only writes → high throughput  

---

## 6. Aggregation & Computation Module

### Purpose

- Convert raw transaction facts into **query-ready analytics**
- Avoid heavy scans during dashboard queries  

### Why Aggregation is Needed

Raw data:
```
txn_1, 100 INR, SUCCESS
txn_2, 50 INR, FAILED
```

Business questions:
- Total volume today  
- Failure rate  
- Merchant success rate  

### Problem Without Aggregation

```sql
SELECT
  SUM(amount),
  COUNT(*),
  SUM(CASE WHEN status='SUCCESS' THEN 1 END),
  SUM(CASE WHEN status='FAILED' THEN 1 END)
FROM transaction_metrics
WHERE event_time BETWEEN '2026-03-21 00:00:00'
AND '2026-03-21 23:59:59';
```

Issues:
- Full table scans  
- High CPU  
- Slow APIs  

---

### Incremental Aggregation Strategy

Each event updates aggregates immediately.

Example:
```
daily_summary[date].total_volume += amount
daily_summary[date].success_count += 1
```

Benefits:
- O(1) queries  
- Fast APIs  
- Predictable latency  

---

### Aggregation Algorithm

```java
LocalDate date = event.getEventTime().toLocalDate();

DailyFinancialSummary summary =
    repo.findById(date)
        .orElse(new DailyFinancialSummary(date));

summary.totalTransactions++;

if (event.isSuccess()) {
    summary.totalVolume += event.amount;
    summary.successfulTxCount++;
} else {
    summary.failedTxCount++;
}

repo.save(summary);
```

---

### Aggregated Table

```sql
CREATE TABLE daily_financial_summary (
  summary_date DATE PRIMARY KEY,
  total_volume NUMERIC(20,2),
  successful_tx_count INT,
  failed_tx_count INT,
  total_transactions INT,
  last_updated TIMESTAMP
);
```

---

### Rebuild Strategy

- Truncate aggregated tables  
- Replay raw data  
- Recompute metrics  

---

## 7. Analytics API Microservice

### Principles

- Read-only  
- Stateless  
- No side effects  
- Dashboard optimized  

---

## 8. Authentication

JWT-based authentication

```
Authorization: Bearer eyJhbGciOi...
```

---

## 9. API Documentation

### Daily Summary

```
GET /analytics/transactions/daily
```

Response:
```json
{
  "totalVolume": 92000000,
  "totalTransactions": 510234,
  "successfulTransactions": 493210,
  "failedTransactions": 17024
}
```

---

### Merchant Analytics

```
GET /analytics/transactions/merchant/{merchantId}
```

---

### Failure Analytics

```
GET /analytics/transactions/failures
```

---

### Status Distribution

```
GET /analytics/transactions/status-distribution
```

---

### Health Check

```
GET /analytics/health
```

---

## 10. Performance Optimizations

Application:
- Stateless services  
- Batch processing  
- Concurrency control  

Database:
- Indexed queries  
- Pre-aggregated tables  

Result:
**40%+ latency reduction**

---

## 11. Failure Handling

| Scenario | Handling |
|--------|--------|
| Duplicate event | Idempotent insert |
| Poison message | DLQ |
| DB lag | Back-pressure |

---

## 12. Ownership Summary

### Implemented

- Event ingestion  
- Aggregation  
- API design  
- DB tuning  
- RabbitMQ setup  

### Not Implemented

- Payment processing  
- Ledger correctness  

---

## 13. Final Mental Model

“I built a **read-only, event-driven analytics platform** that ingests events via RabbitMQ, stores immutable data, performs incremental aggregation, and exposes high-performance APIs without impacting transaction systems.”
