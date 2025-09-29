# StreamView – Technical & Architectural Overview

**Project goal**
StreamView builds **materialized views** on top of Apache Kafka event data to expose fast and predictable reads over REST (ASP.NET Core **Controllers**). The source of truth lives in the stream; SQL databases are used for secondary indexing and querying.

---

## Concrete use case

The source topic is `orders.events` (e.g., `OrderCreated`, `OrderItemAdded`, `OrderStatusChanged`). StreamView consumes events in key order (`orderId`) and maintains the `orders_view` table that powers the API:

* `GET /orders?status=&from=&to=&page=` – paginated & filtered lists,
* `GET /orders/{id}` – order details (denormalized fields),
* `GET /lags` – consumer health and per‑partition lag,
* `POST /views/rebuild` – view rebuild (optional, controlled).

---

## High‑level architecture

* **Ingest:** .NET BackgroundService consumer (Confluent.Kafka) with group `streamview-orders`.
* **Materialization:** upserts into Postgres (`orders_view`, `order_items_view`).
* **REST:** ASP.NET Core MVC Controllers + FluentValidation + Swagger.
* **Observability:** OpenTelemetry (traces, metrics), Prometheus, Jaeger.
* **Idempotency:** de‑duplication based on `eventId`/`sequence` (Kafka header) + SQL constraints.
* **Rebuild:** pause consumer and replay from offset 0 (dev/test environments).

---

## Event contracts (schemas)

Recommended JSON Schema / Avro in a registry (compatibility **BACKWARD**):

* `OrderCreated { eventId, occurredAt, orderId, customerId, currency, items[] }`
* `OrderItemAdded { eventId, occurredAt, orderId, itemId, sku, qty, price }`
* `OrderStatusChanged { eventId, occurredAt, orderId, status }`
  Kafka headers:
* `x-sequence` (monotonic counter per `orderId`),
* `x-trace-id`, `x-event-version`.

---

## View model (SQL)

Table **`orders_view`** (denormalized, read‑optimized):

* `order_id` PK, `customer_id`, `status`,
* `total_net`, `total_gross`, `currency`,
* `item_count`, `last_event_sequence`, `last_event_id`, `updated_at`,
* `dedup_event_id` UNIQUE (fast de‑dup if a row represents the latest event),
* composite indexes for queries (`status, updated_at DESC`).

Table **`order_items_view`** (optional drill‑down):

* `order_id`, `line_no`, `sku`, `qty`, `price`, `net_line_total`, `gross_line_total` (index: `order_id`).

> Rule: **idempotent upsert** – each event is applied at most once; `x-sequence` < `last_event_sequence` → **skip** (out‑of‑order), `==` → **dedup** (duplicate), `== last+1` → **apply**, `> last+1` → **stash** (optional delayed queue).

---

## API (Controllers)

**OrdersController** (`/orders`):

* `GET /orders?status=&from=&to=&page=&pageSize=&sort=`
  Returns `Page<OrderDto>` with `Link: rel="next|prev"` headers.
* `GET /orders/{id}`
  `200` or `404`. Optionally returns `ETag` derived from `updated_at`.

**OpsController** (`/ops`):

* `GET /lags` – per‑partition lag snapshot and consumer state.
* `POST /views/rebuild` – (dev/test) stop consumer, clear view, reset offsets to `0`, and replay.

> Authorization: `Ops` endpoints behind `[Authorize(Policy="OpsOnly")]`.

---

## Consumer processing strategy

1. Subscribe to `orders.events` with key = `orderId`.
2. In the consume loop: validate schema, parse event type.
3. Within a DB transaction (or single upsert):

    * Load `last_event_sequence` for `orderId`.
    * Compare with `x-sequence`:

        * `<` → ignore (late/out‑of‑order); increment metric.
        * `==` → dedup; increment `duplicates_total`.
        * `== last+1` → apply the projection, update `last_event_*`.
        * `>` → optionally stash (side‑queue); count the gap.
4. Commit the Kafka offset **after** a successful DB write (at‑least‑once with idempotency).

**DLQ**

* Unparseable messages or schema violations → publish to `orders.events.dlq` with reason.

---

## Docker Compose (local)

* **Redpanda/Kafka** (single broker) + **Schema Registry** (e.g., Redpanda/Apicurio).
* **Postgres 16**.
* **Jaeger** (tracing), **Prometheus + Grafana** (metrics) – optional.
* **StreamView.API** (ASP.NET Core) with Controllers.

Example ports:

* API: `8080`, Postgres: `5432`, Kafka: `9092`, Registry: `8081`, Jaeger UI: `16686`.

---

## Observability (important)

Metrics (Prometheus):

* `kafka_consumer_lag{partition}`
* `streamview_events_total{type}`
* `streamview_duplicates_total`
* `streamview_out_of_order_total`
* `streamview_upserts_duration_seconds` (histogram)
* `http_requests_duration_seconds` (controllers)

Traces (OpenTelemetry): span from poll → parse → upsert → commit; propagate `traceparent` from Kafka headers.

Logs: Serilog in JSON with `orderId`, `eventId`, `sequence`, `partition`, `offset`.

---

## Tests

**Unit** (Domain/Projection):

* Applying events in order and out of order.
* Idempotency – repeated `x-sequence`.

**Integration** (Testcontainers):

* Start Postgres + Redpanda.
* Publish a sequence of events → verify `orders_view` via REST.
* Publish a duplicate → state unchanged; `duplicates_total`++.
* Publish out‑of‑order (larger `x-sequence`) → state unchanged; `out_of_order_total`++.

**Contract tests** (schemas):

* Validate messages against the Registry (BACKWARD compatibility).

---

## Backlog – implementation steps

1. API skeleton (Controllers), Swagger, Health, Serilog JSON, Versioning (optional).
2. Compose: Redpanda + Registry + Postgres + API + Jaeger.
3. Event schemas (JSON Schema/Avro), header helper for `x-sequence`.
4. Postgres: `orders_view` + indexes, EF migrations.
5. Kafka consumer (BackgroundService) + upsert loop (idempotency, out‑of‑order, DLQ).
6. `OrdersController` – `GET /orders`, `GET /orders/{id}` (pagination, sorting, filters, `Link` headers).
7. `OpsController` – `GET /lags` + consumer health; (opt.) `POST /views/rebuild`.
8. Observability: OTel + Prometheus + dashboard (lag, throughput, error rate).
9. Testcontainers: integration scenarios + event seeders.

---

## Non‑functional decisions

* **Exactly‑once:** achieved in practice as *at‑least‑once + idempotent upsert*.
* **Partition key:** `orderId` guarantees ordering within an order.
* **Retention:** `orders.events` with long `retention.ms`; optionally `compact` for state logs.
* **Security:** dev without TLS; prod with mTLS/SASL, dedicated consumer principal.
* **Scaling:** consumer instances ≤ partition count; API scales horizontally with Postgres (indexes matter).

---

## Where this fits your projects

* **LekPing/Billing:** materialized views for subscriptions & invoices – fast reporting.
* **ServoSense/FleetStream:** telemetry views (latest device state, fast alert lists).
* **Storefront:** fast order/customer listings without touching the source systems.

Practical, production‑shaped training: Controllers, Kafka consumer, idempotency, Testcontainers, observability, and ops (lag, rebuild).
