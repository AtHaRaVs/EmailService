# EmailService

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Design Patterns & Principles](#design-patterns--principles)
   - Circuit Breaker
   - Exponential Backoff
   - Queueing & Concurrency Control
   - Idempotency
   - Rate Limiting
4. [Configuration & Extensibility](#configuration--extensibility)
5. [Error Handling & Retry Strategy](#error-handling--retry-strategy)
6. [Metrics, Monitoring & Observability](#metrics-monitoring--observability)
7. [Runbook & Operational Guide](#runbook--operational-guide)
8. [Testing Strategy](#testing-strategy)
9. [CI/CD & Release Management](#cicd--release-management)
10. [Contributing](#contributing)
11. [License](#license)

---

## Overview

`EmailService` is a production-grade, fault-tolerant Node.js micro-library for multi-provider transactional email delivery. It encapsulates provider failover, idempotency, rate limiting, circuit breaker, and exponential backoff to ensure reliable email dispatch at scale.

## Architecture

- **Provider Abstraction**: Pluggable provider interface; supports multiple SMTP/HTTP-based providers.
- **Core Components**:
  - **Queue Manager**: In-memory task queue with consumption loop.
  - **Rate Limiter**: Token bucket algorithm resets every minute.
  - **Circuit Breaker**: Tracks failure count; trips on threshold; auto-resets after cooldown.
  - **Idempotency Cache**: In-memory LRU cache for request deduplication.
  - **Status Tracker**: Maps tracking IDs to lifecycle states (`queued` → `sending` → `sent`/`failed`).

## Design Patterns & Principles

### Circuit Breaker

- Prevents cascading failures when provider endpoints are down.
- **Threshold**: 3 consecutive failures → opens the circuit.
- **Cooldown**: Default 30s, configurable for different SLAs.

### Exponential Backoff

- On provider error, retries up to 3 attempts with backoff factor:  
  `delay = base * 2^attempts` (configurable base).

### Queueing & Concurrency Control

- Single-worker model; can be extended to a worker pool.
- Ensures ordered dispatch and backpressure handling.

### Idempotency

- Deduplicates requests with unique keys.
- Prevents double-sends in retry and network failure scenarios.

### Rate Limiting

- Configurable max emails/minute.
- Excess tasks remain queued until quota refresh.

## Configuration & Extensibility

```js
const svc = new EmailService({
  rateLimit: 100, // emails per minute
  backoffBase: 500, // ms
  circuitBreaker: {
    threshold: 5,
    coolDown: 60000, // ms
  },
  providers: [
    /* custom providers */
  ],
});
```

- Override default providers and behaviors via constructor.

## Error Handling & Retry Strategy

1. **Attempt Send**:
   - If circuit is open → immediate failure.
   - Attempt up to 3 times across providers.
2. **Failover**:
   - Round-robin provider switch on each retry.
3. **Final Failure**:
   - Cache status; provide detailed error via `getStatus(trackingId)`.

## Metrics, Monitoring & Observability

- Expose Prometheus-compatible metrics:
  - `email_sent_total`
  - `email_failed_total`
  - `circuit_breaker_state`
  - `queue_length`
  - `rate_limit_remaining`
- Instrumentation hooks for custom logging and alerting.

## Runbook & Operational Guide

- **Deployment**:
  - Embed service in your microservice or serverless function.
- **Alerts**:
  - Trigger on high failure rate or prolonged open circuit.
- **SLA Considerations**:
  - Maintain error budget; configure thresholds per provider SLA.
- **Scaling**:
  - Deploy multiple worker instances behind a distributed queue (e.g., Redis).

## Testing Strategy

- **Unit Tests**: Jest mocks for providers and time-based behaviors.
- **Integration Tests**: End-to-end envelope testing with real SMTP sandbox.
- **Chaos Testing**: Simulate provider downtimes and network spikes.
- **Load Testing**: Validate rate limiter under bursts.

## CI/CD & Release Management

- GitHub Actions pipeline:
  - Lint, unit tests, coverage enforcement.
  - Docker image build and push.
  - Semantic versioning via Conventional Commits.

## Contributing

1. Fork the repo.
2. Create feature branch.
3. Write tests before code.
4. Run `npm test` and ensure coverage.
5. Submit PR with detailed description and changelog entry.

## License

MIT © [Your Company]
