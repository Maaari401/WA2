# WA2 Project - Milestone 2: Data Ownership Table

**Group:** 6  
**Project Domain:** Distributed Ticketing Platform  


---

## 1. Executive Summary

In accordance with microservice architectural principles, no service reads or writes the tables of another one. Inter-service interaction occurs exclusively through:
1. **Synchronous HTTP APIs** (one OpenAPI 3.1 specification per service, see `openapi/`).
2. **Asynchronous Event-Driven Messaging** via RabbitMQ using the Transactional Outbox Pattern.

Key modelling decisions:
- An **event** has one or more **sessions** (dates); every session has its own seat inventory.
- The **seat layout is fixed**: each venue has predefined sectors (rows × seats per row), loaded as seed data.
- **Price tiers** are bound to venue sectors and apply to all sessions of the event.
- Seats are **generated and owned by Inventory & Orders** from the `SessionPublished` message; each seat keeps a copy of its tier and price.
- Money is always stored as **integer cents + ISO 4217 currency**.
- Every consumer deduplicates messages by `messageId` in a `processed_message` table (at-least-once delivery).

---

## 2. Data Ownership Matrix

| Microservice | Stack | Schema / Tables Owned | Consistency & Access Rules | Inter-Service Interaction (Sync / Async) |
| :--- | :--- | :--- | :--- | :--- |
| **API Gateway / BFF** | Kotlin / Spring Boot | *None (Stateless)* | Single entry point for client requests (SPA). Validates OIDC JWT tokens from Keycloak and routes requests per path to downstream services (each service verifies the JWT again). Does not persist domain data. Does not expose the Payments endpoints. | **Sync:** Proxies incoming HTTP requests to Catalogue, Inventory & Orders, Ticketing. |
| **Catalogue Service** | Kotlin / Spring Boot | `venue`<br>`sector`<br>`event`<br>`session`<br>`price_tier`<br>`tier_sector`<br>`session_sales`<br>`processed_message`<br>`outbox` | **Catalogue Domain Owner.** Manages venues and their fixed sector layout, events, sessions (dates and sales opening time) and price tiers bound to sectors.<br><br>`tier_sector` guarantees that every sector has exactly one tier per event.<br><br>`session_sales` is an **event-driven read model** for the organiser dashboard (per session and per tier), updated strictly via `OrderPaid`. | **Publishes:** `SessionPublished` (one per session), `SessionCancelled` *(optional)*<br>**Consumes:** `OrderPaid` (updates `session_sales`) |
| **Inventory & Orders Service** | Rust / Axum + SQLx | `session`(another table)<br>`seat_state`<br>`hold`<br>`order`<br>`processed_message`<br>`outbox` | **Inventory & Purchase Boundary Owner.** Generates the seats of a session from `SessionPublished` and owns their state transitions (`FREE` $\rightarrow$ `HELD` $\rightarrow$ `SOLD`) together with the order lifecycle, to prevent race conditions and overbooking (conditional update `WHERE status = 'FREE'`).<br><br>`session` is a minimal local copy (dates, sales opening, status) built from messages. The order amount comes from the price copied on `seat_state`.<br><br>Manages 10-minute hold TTLs via scheduled background jobs. | **Publishes:** `OrderPaid`, `OrderRejected`<br>**Consumes:** `SessionPublished`, `SessionCancelled` *(optional)*, `PaymentCaptured`, `PaymentFailed`<br>**Sync Calls:** Calls `Payments` (`POST /payments`, 2s timeout) |
| **Payments Service** | Kotlin / Spring Boot | `payment`<br>`webhook_inbox`<br>`processed_message`<br>`outbox` | **Financial Domain Owner.** Interfaces with the external Payment Provider (Sandbox). Tracks payment states (`PENDING`, `CAPTURED`, `DECLINED`). One payment per order (idempotent on `orderId`).<br><br>Uses `webhook_inbox` to guarantee strict deduplication of incoming provider webhooks (verified with HMAC signature). | **Publishes:** `PaymentCaptured`, `PaymentFailed`<br>**Consumes:** `OrderRejected` (triggers payment reversal/refund compensation)<br>**Sync Endpoints:** Exposes `POST /payments` for internal synchronous payment initialization |
| **Ticketing Service** | Rust / Axum + SQLx | `ticket`<br>`scan`<br>`processed_message`<br>`outbox` | **Access Control Owner.** Issues one digital ticket per paid seat upon order confirmation.<br><br>Stores gate validation records in `scan` and enforces strict **idempotency** to prevent ticket reuse or double entry. | **Publishes:** `TicketIssued`<br>**Consumes:** `OrderPaid` (triggers ticket issuance) |

---

## 3. Messages (RabbitMQ)

Every message has the envelope `{ messageId, type, occurredAt, data }`.

| Message | Producer | Consumers | Main `data` fields |
| :--- | :--- | :--- | :--- |
| `SessionPublished` | Catalogue | Inventory & Orders | `sessionId`, `eventId`, `startsAt`, `salesOpenAt`, `sectors[] { code, rows, seatsPerRow, tierCode, priceCents, currency }` |
| `SessionCancelled` *(optional)* | Catalogue | Inventory & Orders | `sessionId`, `reason` |
| `OrderPaid` | Inventory & Orders | Catalogue, Ticketing | `orderId`, `sessionId`, `seat`, `tierCode`, `amountCents`, `currency`, `customerId` |
| `OrderRejected` | Inventory & Orders | Payments | `orderId`, `reason` |
| `PaymentCaptured` | Payments | Inventory & Orders | `orderId`, `paymentId`, `amountCents`, `currency` |
| `PaymentFailed` | Payments | Inventory & Orders | `orderId`, `paymentId`, `reason` |
| `TicketIssued` | Ticketing | *(Notifications, optional)* | `ticketId`, `orderId`, `sessionId`, `seat`, `customerId` |
