# Payment Service

Idempotent payment processor for the [Order Management Platform](https://github.com/soltyDude/order-management-infra). Handles charges, refunds, and multiple payment methods via Strategy pattern. Protected by circuit breaker on external gateway calls.

## Status: In Development

**MVP planned: Phase 2 of the [project roadmap](https://github.com/soltyDude/order-management-infra#roadmap).**

Architecture, database schema, and API contract are fully designed and documented. Implementation begins after [order-service](https://github.com/soltyDude/order-service) Phase 1 completion.

## Responsibility

- Process payments idempotently (no duplicate charges on retries)
- Support multiple payment methods through Strategy pattern (CARD, PayPal, bank transfer)
- Call external payment gateway with circuit breaker protection
- Handle refunds — full (saga compensation) and partial (admin REST)
- Publish payment results as Kafka events for saga orchestration

## Patterns

| Pattern | Why |
|---------|-----|
| **Strategy** | `PaymentProcessor` interface with per-method implementations (Card, PayPal, BankTransfer). Adding a new method = new class, zero changes to existing code (OCP). Runtime selection via `supports()`. |
| **Idempotency** | `Idempotency-Key` header (UUID) + request body hash. Same key + same body = cached response. Same key + different body = 409. 24h TTL. Prevents duplicate charges on network retries. |
| **Circuit Breaker** | External payment gateway may be slow or down. Resilience4j CB: 50% failure rate / 10 calls → open 30s → half-open (3 test calls). Prevents cascading failures. |
| **Outbox** | Payment result events saved in the same transaction as payment record. Poller publishes to Kafka. No dual-write risk. |

## Planned API

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/payments` | Process payment (idempotent via `Idempotency-Key` header) |
| GET | `/api/v1/payments/{paymentId}` | Payment details with refund info |
| GET | `/api/v1/payments` | List payments (filtered by orderId, userId, status) |
| POST | `/api/v1/payments/{paymentId}/refund` | Manual refund (admin only) |

### Example: Process Payment

```bash
curl -X POST http://localhost:8081/api/v1/payments \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <jwt>" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "orderId": "d290f1ee-6c54-4b01-90e6-d701748f0851",
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 109.97,
    "currency": "USD",
    "method": "CARD"
  }'
```

**Response (201):**
```json
{
  "id": "pay_abc123def456",
  "orderId": "d290f1ee-6c54-4b01-90e6-d701748f0851",
  "status": "PROCESSING",
  "amount": 109.97,
  "currency": "USD",
  "method": "CARD",
  "createdAt": "2025-01-15T10:30:01Z"
}
```

> Primary payment flow goes through Kafka (`payment.charge.requested` from order-service), not REST. REST endpoints exist for admin operations, direct consumers, and testing.

## Planned Database Schema

Five tables in `payments_db` (port 5433):

| Table | Purpose |
|-------|---------|
| `payments` | Payment records. ID format: `pay_` + nanoid. Tracks status, gateway reference, failure reason |
| `refunds` | Refund records. ID format: `ref_` + nanoid. Supports full (saga) and partial (REST) refunds |
| `idempotency_keys` | Request dedup. Maps UUID key → request hash → cached response. 24h TTL |
| `outbox_events` | Transactional outbox for Kafka publishing |
| `processed_events` | Consumer idempotency |

```
payments (1) ──── (N) refunds              [FK: payment_id]
payments (1) ──── (0..1) idempotency_keys  [FK: payment_id]
```

## Planned Package Structure

```
com.example.paymentservice/
├── api/
│   ├── controller/          PaymentController
│   ├── dto/
│   │   ├── request/         ProcessPaymentRequest, RefundRequest
│   │   └── response/        PaymentDto, PaymentSummaryDto, RefundDto
│   └── exception/           GlobalExceptionHandler, InsufficientFundsException
├── domain/
│   ├── model/               Payment, Refund, PaymentStatus, PaymentMethod
│   ├── event/               PaymentProcessedEvent, PaymentFailedEvent, PaymentRefundedEvent
│   ├── service/             PaymentService
│   └── processor/
│       ├── PaymentProcessor.java           (interface)
│       ├── CardPaymentProcessor.java       (calls gateway + circuit breaker)
│       ├── BankTransferProcessor.java
│       └── PayPalPaymentProcessor.java
├── infrastructure/
│   ├── kafka/
│   │   ├── producer/        OutboxPoller
│   │   └── consumer/        PaymentRequestConsumer, RefundRequestConsumer
│   ├── persistence/
│   │   ├── entity/          IdempotencyKey, OutboxEvent, ProcessedEvent
│   │   ├── repository/      PaymentRepository, RefundRepository, IdempotencyKeyRepository
│   │   └── mapper/          PaymentMapper
│   ├── gateway/             PaymentGatewayClient (WebClient + @CircuitBreaker + @Retry)
│   └── config/              KafkaConfig, SecurityConfig, ResilienceConfig
└── PaymentServiceApplication.java
```

## Kafka Topics

**Consumes (commands from order-service):**
| Topic | Action |
|-------|--------|
| `payment.charge.requested` | Process payment via Strategy, publish result |
| `payment.refund.requested` | Process full refund (saga compensation) |

**Produces (results back to order-service):**
| Topic | When |
|-------|------|
| `payment.processed` | Payment completed successfully |
| `payment.failed` | Payment declined (insufficient funds, gateway error) |
| `payment.refunded` | Refund completed |

## Tech Stack

Java 17 · Spring Boot 3.x · PostgreSQL 16 · Flyway · Resilience4j · Maven

## Related

- [order-management-infra](https://github.com/soltyDude/order-management-infra) — project overview, Docker Compose, documentation
- [order-service](https://github.com/soltyDude/order-service) — saga orchestrator (produces commands for this service)
- [Full API Contract](https://github.com/soltyDude/order-management-infra/blob/main/docs/api-contract-full.md)
