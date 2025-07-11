# EmailService

A production-grade, fault-tolerant Node.js micro-library for multi-provider transactional email delivery.

## Table of Contents

- [High-Level Overview](#high-level-overview)  
- [Key Features](#key-features)  
- [Architecture](#architecture)  
- [Usage](#usage)  
  - [Installation](#installation)  
  - [Quick Start](#quick-start)  
  - [Advanced Configuration](#advanced-configuration)  
  - [Extending the Service](#extending-the-service)  
- [Operational Guide](#operational-guide)  
  - [Metrics & Observability](#metrics--observability)  
  - [Runbook](#runbook)  
- [Testing Strategy](#testing-strategy)  
- [Design Decisions](#design-decisions)  
- [Contributing](#contributing)  
- [License](#license)  

---

## High-Level Overview

**EmailService** abstracts provider fail-over, rate limiting, automatic retries with exponential back-off, circuit breaking, and idempotency behind a unified API. The goal is **five-nines reliability** for critical notification flows without burdening product teams with cross-provider logic.

---

## Key Features

- **Multi-Provider Rotation** – Round-robin fallback between any number of SMTP or HTTP email gateways.  
- **Circuit Breaker** – Opens after N consecutive failures per provider, auto-resets after a configurable cool-down.  
- **Exponential Back-Off Retries** – Configurable base delay and retry ceiling.  
- **Global Rate Limiting** – Sliding window limiter (default 10 emails per minute) to respect provider quotas.  
- **Idempotency Cache** – Guarantees exactly-once semantics for external callers.  
- **In-Memory Queue** – Decouples caller latency from provider throughput.  
- **Pluggable Observability** – Emits hookable events for metrics, tracing, and structured logs.  
- **100% Test Coverage** – Jest suite validates all edge cases, including degraded-mode scenarios.  

---

## Architecture

```mermaid
flowchart LR
  Client -->|sendEmail()| Queue[In-Memory Queue]
  Queue -->|processQueue()| EmailService
  subgraph EmailService
    Policy[Rate Limiter]
    Policy -->|allow| Dispatcher
    Dispatcher -->|attemptSend()| ProviderA & ProviderB
    Dispatcher -->|fallback| ProviderB
    Dispatcher -->|update| CircuitBreaker
    Dispatcher -->|update| IdempotencyCache
  end
```

### Component Responsibilities

| Component          | Responsibility                                    |
|-------------------|---------------------------------------------------|
| Queue             | Buffered hand-off; isolates spikes.               |
| Rate Limiter      | Throttles throughput per configured window.       |
| Dispatcher        | Orchestrates retries, provider rotation, status.  |
| Circuit Breaker   | Protects downstream providers; trips on failure.  |
| Idempotency Cache | Prevents duplicate sends across restarts.         |

---

## Usage

### Installation

```bash
npm install @your-org/emailservice
```

### Quick Start

```javascript
const EmailService = require('@your-org/emailservice');

const emailService = new EmailService({
  rateLimit: 50,               // 50 emails / min
  backoffBase: 250,            // 250 ms base for retries
  circuitBreaker: {
    threshold: 5,
    coolDown: 15_000           // 15 s
  },
  providers: [
    new SendGridAdapter(process.env.SENDGRID_TOKEN),
    new SESAdapter({ region: 'us-east-1' })
  ]
});

const email = {
  to: 'user@example.com',
  subject: 'Welcome',
  body: 'Thanks for signing up!'
};

const { trackingId, status } = await emailService.sendEmail(email, 'signup-123');
console.log({ trackingId, status });
```

---

## Advanced Configuration

| Option             | Type / Default                      | Description                                         |
|--------------------|-------------------------------------|-----------------------------------------------------|
| providers          | `Provider[]` (required)             | Ordered list of adapters implementing `send()`.     |
| rateLimit          | `number` / `10`                     | Max messages per 60s window.                        |
| backoffBase        | `number` / `1000`                   | Base delay in ms for exponential retry.             |
| circuitBreaker     | `object`                            | `{ threshold: 3, coolDown: 30_000 }`                |
| idempotencyTTL     | `number` / `3600000`                | TTL for idempotency cache in ms.                    |
| queueIntervalMs    | `number` / `100`                    | Polling interval for processing queue.              |

---

## Extending the Service

### Creating a Provider Adapter

```typescript
class PostmarkAdapter {
  constructor(apiKey) {
    this.client = new Postmark.Client(apiKey);
  }

  name = 'Postmark';

  async send(email) {
    const response = await this.client.sendEmail({
      From: 'noreply@your.org',
      To: email.to,
      Subject: email.subject,
      HtmlBody: email.body
    });
    if (response.ErrorCode !== 0) throw new Error(response.Message);
    return `Email sent via Postmark: ${email.subject}`;
  }
}
```

```javascript
emailService.addProvider(new PostmarkAdapter(process.env.POSTMARK_TOKEN));
```

### Overriding Storage

```javascript
emailService.setIdempotencyStore(new RedisCache({ ttl: 3600 }));
```

---

## Operational Guide

### Metrics & Observability

| Metric                             | Type     | Description                            |
|-----------------------------------|----------|----------------------------------------|
| `email_sent_total{provider}`      | Counter  | Successful sends per provider.         |
| `email_failure_total{provider}`   | Counter  | Failed attempts per provider.          |
| `email_queue_depth`               | Gauge    | Current queue length.                  |
| `circuit_breaker_open{provider}`  | Gauge    | 1 if open, 0 otherwise.                |
| `email_send_duration_ms`         | Histogram| End-to-end latency.                    |

```javascript
emailService.on('metrics', (metric) => pushToPrometheus(metric));
```

### Runbook

| Scenario                  | Operator Action                                                  |
|---------------------------|------------------------------------------------------------------|
| Queue backlog > 100       | Scale service instances; verify provider availability.          |
| Circuit breaker open      | Investigate provider status; check credential validity.         |
| High failure rate         | Enable verbose logging: `DEBUG=emailservice* node app.js`.      |
| Memory Leak Suspected     | Restart with `--max-old-space-size`; confirm idempotency TTL.   |

---

## Testing Strategy

- **Unit Tests** – 100% branch coverage via Jest. Simulate flake, latency, rate limits.
- **Contract Tests** – Each adapter validated against provider sandbox APIs.
- **Load Tests** – `k6` scripts sustaining 1k RPS to test queue and GC stability.
- **Chaos Suite** – Inject network errors and 5xx responses to validate resilience.

```bash
npm test
npm run load-test
```

---

## Design Decisions

- **In-Memory Queue vs Distributed** – Chose in-memory for minimal latency; users can plug in external queues.
- **Backoff Formula** – `delay = backoffBase × 2^attempt`. Predictable. Overrideable for advanced users.
- **Circuit Breaker Scope** – Global across providers to prevent cascade; can be made per-provider if needed.
- **No Logger Lock-in** – Emits structured events instead of direct logs.

---

## Contributing

1. Fork & clone this repo.  
2. Run `npm ci` to install exact dependencies.  
3. Use [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, etc.)  
4. Add/modify unit tests. Ensure `npm test` passes.  
5. Submit PR — template validates lint, types, and 95%+ coverage.  
6. We follow [DORA metrics](https://www.devops-research.com/research.html); PRs should keep MTTR and frequency healthy.

---

## License

MIT © Your Organization 2025