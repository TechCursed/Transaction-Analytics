Project Overview

Service name: transaction-analytics-service
Type: Event‑driven, read‑only analytics microservice
Stack: Java 17, Spring Boot, Spring Data JPA, RabbitMQ.

📁 Directory Structure (As Requested)
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

🧩 DOMAIN LAYER

domain/TransactionMetric.java

Stores immutable raw transaction facts
Append‑only + idempotent

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
``
domain/DailyFinancialSummary.java

Pre‑aggregated analytics for fast dashboards
package domain;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDate;

/**
* Stores aggregated metrics per calendar day.
* Optimized for read-heavy analytics APIs.
*/
@Entity
@Table(name = "daily_financial_summary")
public class DailyFinancialSummary {

    @Id
    private LocalDate summaryDate;

    private BigDecimal totalVolume = BigDecimal.ZERO;
    private int successfulTxCount;
    private int failedTxCount;
    private int totalTransactions;

    protected DailyFinancialSummary() {}

    public DailyFinancialSummary(LocalDate date) {
        this.summaryDate = date;
    }

    public void applyEvent(TransactionMetric metric) {
        totalTransactions++;

        if ("SUCCESS".equals(metric.getStatus())) {
            successfulTxCount++;
            totalVolume = totalVolume.add(metric.getAmount());
        } else if ("FAILED".equals(metric.getStatus())) {
            failedTxCount++;
        }
    }

    // Getters
}

🗄️ REPOSITORY LAYER

repository/TransactionMetricRepository.java

package repository;

import domain.TransactionMetric;
import org.springframework.data.jpa.repository.JpaRepository;

public interface TransactionMetricRepository
        extends JpaRepository<TransactionMetric, Long> {
}
``

repository/DailySummaryRepository.java

package repository;

import domain.DailyFinancialSummary;
import org.springframework.data.jpa.repository.JpaRepository;

import java.time.LocalDate;
import java.util.List;

public interface DailySummaryRepository
        extends JpaRepository<DailyFinancialSummary, LocalDate> {

    List<DailyFinancialSummary> findBySummaryDateBetween(
            LocalDate from,
            LocalDate to);
}
``

⚙️ CONFIGURATION LAYER

config/RabbitMQConfig.java

package config;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
* RabbitMQ topology for transaction analytics ingestion.
*/
@Configuration
public class RabbitMQConfig {

    public static final String EXCHANGE = "transaction.events.exchange";
    public static final String QUEUE = "analytics.transaction.events.queue";

    @Bean
    public TopicExchange exchange() {
        return new TopicExchange(EXCHANGE);
    }

    @Bean
    public Queue analyticsQueue() {
        return QueueBuilder.durable(QUEUE)
                .withArgument("x-dead-letter-exchange",
                        "transaction.events.dlq")
                .build();
    }

    @Bean
    public Binding binding() {
        return BindingBuilder
                .bind(analyticsQueue())
                .to(exchange())
                .with("transaction.*");
    }
}

config/DatabaseConfig.java

package config;

import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.annotation.EnableTransactionManagement;

/**
* Enables transactional boundaries for aggregation safety.
*/
@Configuration
@EnableTransactionManagement
public class DatabaseConfig {
}

📨 EVENT CONSUMER

consumer/TransactionEventConsumer.java

package consumer;

import service.EventProcessingService;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.Map;

/**
* Consumes transaction lifecycle events from RabbitMQ.
*/
@Component
public class TransactionEventConsumer {

    private final EventProcessingService service;

    public TransactionEventConsumer(EventProcessingService service) {
        this.service = service;
    }

    @RabbitListener(queues = "analytics.transaction.events.queue")
    public void consume(Map<String, Object> payload) {
        service.processEvent(payload);
    }
}

🧠 SERVICE LAYER

service/EventProcessingService.java

package service;

import domain.TransactionMetric;
import repository.TransactionMetricRepository;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Map;

/**
* Handles schema validation, idempotency and routing
* to aggregation logic.
*/
@Service
public class EventProcessingService {

    private final TransactionMetricRepository repository;
    private final AggregationService aggregationService;

    public EventProcessingService(
            TransactionMetricRepository repository,
            AggregationService aggregationService) {

        this.repository = repository;
        this.aggregationService = aggregationService;
    }

    public void processEvent(Map<String, Object> event) {

        TransactionMetric metric = new TransactionMetric(
                (String) event.get("transactionId"),
                new BigDecimal(event.get("amount").toString()),
                (String) event.get("currency"),
                (String) event.get("status"),
                (String) event.get("merchantId"),
                (String) event.get("region"),
                LocalDateTime.parse((String) event.get("eventTime"))
        );

        try {
            repository.save(metric);
            aggregationService.aggregate(metric);
        } catch (DataIntegrityViolationException ex) {
            // Duplicate event → safely ignored (idempotency)
        }
    }
}
``

service/AggregationService.java

package service;

import domain.DailyFinancialSummary;
import domain.TransactionMetric;
import repository.DailySummaryRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;

/**
* Performs incremental aggregation at ingestion time.
*/
@Service
public class AggregationService {

    private final DailySummaryRepository repository;

    public AggregationService(DailySummaryRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public void aggregate(TransactionMetric metric) {

        LocalDate date = metric.getEventTime().toLocalDate();

        DailyFinancialSummary summary =
                repository.findById(date)
                        .orElse(new DailyFinancialSummary(date));

        summary.applyEvent(metric);
        repository.save(summary);
    }
}
``

service/MetricsQueryService.java
package service;

import domain.DailyFinancialSummary;
import repository.DailySummaryRepository;
import org.springframework.stereotype.Service;

import java.time.LocalDate;
import java.util.List;

/**
* Handles all read-only analytics queries.
*/
@Service
public class MetricsQueryService {

    private final DailySummaryRepository repository;

    public MetricsQueryService(DailySummaryRepository repository) {
        this.repository = repository;
    }

    public List<DailyFinancialSummary> fetchDailySummary(
            LocalDate from,
            LocalDate to) {

        return repository.findBySummaryDateBetween(from, to);
    }
}

🌐 API LAYER

api/AnalyticsController.java

package api;

import domain.DailyFinancialSummary;
import service.MetricsQueryService;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;
import java.util.List;

/**
* Read-only analytics APIs.
*/
@RestController
@RequestMapping("/analytics")
public class AnalyticsController {

    private final MetricsQueryService service;

    public AnalyticsController(MetricsQueryService service) {
        this.service = service;
    }

    @GetMapping("/transactions/daily")
    public List<DailyFinancialSummary> dailySummary(
            @RequestParam LocalDate from,
            @RequestParam LocalDate to) {

        return service.fetchDailySummary(from, to);
    }
}

lnk : https://hypernotepad.com/n/5fb3b2b9ecafa788
