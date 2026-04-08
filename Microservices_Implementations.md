# Project Overview

Service name: transaction-analytics-service
Type: Event‑driven, read‑only analytics microservice
Stack: Java 17, Spring Boot, Spring Data JPA, RabbitMQ.

## 📁 Directory Structure (As Requested)

transaction-analytics-service
│
├── api
│   └── AnalyticsController.java
│
├── consumer
│   └── TransactionEventConsumer.java
│
├── service
│   ├── EventProcessingService.java
│   ├── AggregationService.java
│   └── MetricsQueryService.java
│
├── repository
│   ├── TransactionMetricRepository.java
│   └── DailySummaryRepository.java
│
├── domain
│   ├── TransactionMetric.java
│   └── DailyFinancialSummary.java
│
└── config
    ├── RabbitMQConfig.java
    └── DatabaseConfig.java

## DOMAIN LAYER

domain/TransactionMetric.java

package domain;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
 * Represents a single immutable transaction event fact.
 * This table is append-only and replay-safe.
 */
@Entity
@Table(
    name = "transaction_metrics",
    uniqueConstraints = @UniqueConstraint(
        columnNames = {"transaction_id", "status"}
    )
)
public class TransactionMetric {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "transaction_id", nullable = false)
    private String transactionId;

    private BigDecimal amount;
    private String currency;
    private String status;
    private String merchantId;
    private String region;
    private LocalDateTime eventTime;

    protected TransactionMetric() {}

    public TransactionMetric(
            String transactionId,
            BigDecimal amount,
            String currency,
            String status,
            String merchantId,
            String region,
            LocalDateTime eventTime) {

        this.transactionId = transactionId;
        this.amount = amount;
        this.currency = currency;
        this.status = status;
        this.merchantId = merchantId;
        this.region = region;
        this.eventTime = eventTime;
    }

    // Getters only (immutable design)
}

🗄️ REPOSITORY LAYER




repository/DailySummaryRepository.java



⚙️ CONFIGURATION LAYER



📨 EVENT CONSUMER



# SERVICE LAYER

service/EventProcessingService.java



service/AggregationService.java



service/MetricsQueryService.java


# 🌐 API LAYER

api/AnalyticsController.java



lnk : https://hypernotepad.com/n/5fb3b2b9ecafa788
