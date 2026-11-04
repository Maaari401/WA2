# WA2 Project - Milestone 2: Data Ownership Table

**Group:** 6  
**Project Domain:** Distributed Ticketing Platform  


---

## 1. Executive Summary

In accordance with microservice architectural principles, inter-service interaction occurs exclusively through:
1. **Synchronous HTTP APIs** (defined via OpenAPI 3.1 specifications).
2. **Asynchronous Event-Driven Messaging** via RabbitMQ using the Transactional Outbox Pattern.

---

## 2. Data Ownership Matrix

| Microservice | Stack | Schema / Tables Owned | Consistency & Access Rules | Inter-Service Interaction (Sync / Async) |
| :--- | :--- | :--- | :--- | :--- |
| **API Gateway / BFF** | Kotlin / Spring Boot | *None (Stateless)* | Acts as the single entry point for client requests (SPA). Validates OIDC JWT tokens from Keycloak and proxies requests to downstream services. Does not persist domain data. | **Sync:** Proxies incoming HTTP requests to downstream microservices. |
| **Catalogue Service** | Kotlin / Spring Boot | `venue`<br>`seat_plan`<br>`event`<br>`session`<br>`price_tier`<br>`session_sales`<br>`outbox` | **Catalogue Domain Owner.** Manages venue configurations, seating plans, event publications, and sales opening times.<br><br>The `session_sales` table is an **event-driven read model** for the organiser dashboard, updated strictly via asynchronous events. | **Publishes:** `SessionPublished`<br>**Consumes:** `OrderPaid` (updates `session_sales`) |
| **Inventory & Orders Service** | Rust / Axum + SQLx | `seat_state`<br>`hold`<br>`order`<br>`outbox` | **Inventory & Purchase Boundary Owner.** Encapsulates seat state transitions (`FREE` $\rightarrow$ `HELD` $\rightarrow$ `SOLD`) and order lifecycles to prevent race conditions and overbooking.<br><br>Manages 10-minute hold TTLs via scheduled background jobs and handles automatic hold expiration. | **Publishes:** `OrderPaid`, `OrderRejected`<br>**Consumes:** `SessionPublished`, `PaymentCaptured`, `PaymentFailed`<br>**Sync Calls:** Calls `Payments` (`POST /payments`, 2s timeout) |
| **Payments Service** | Kotlin / Spring Boot | `payment`<br>`webhook_inbox`<br>`outbox` | **Financial Domain Owner.** Interfaces with the external Payment Provider (Sandbox). Tracks payment states (`PENDING`, `CAPTURED`, `DECLINED`).<br><br>Uses `webhook_inbox` to guarantee strict deduplication of incoming provider webhooks. | **Publishes:** `PaymentCaptured`, `PaymentFailed`<br>**Consumes:** `OrderRejected` (triggers payment reversal/refund compensation)<br>**Sync Endpoints:** Exposes `POST /payments` for internal synchronous payment initialization |
| **Ticketing Service** | Rust / Axum + SQLx | `ticket`<br>`scan` | **Access Control Owner.** Issues digital tickets upon order confirmation.<br><br>Stores gate validation records in `scan` and enforces strict **idempotency** to prevent ticket reuse or double entry. | **Publishes:** `TicketIssued`<br>**Consumes:** `OrderPaid` (triggers ticket issuance) |
