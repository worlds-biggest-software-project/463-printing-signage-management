# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Printing & Signage Management (Candidate #463)
> Generated: 2026-05-25

---

## Approach Summary

An event-sourced architecture with Command Query Responsibility Segregation (CQRS) for the printing and signage MIS. Every state change -- a quote being approved, a job entering production, a barcode scan at a press, a proof rejection, a delivery confirmation -- is captured as an immutable event in an append-only event store. Read models (projections) are built by replaying events into denormalized query-optimised views.

This approach is a natural fit for the print shop domain because:
- **Jobs have complex lifecycles.** A print job passes through 10-20 state transitions from estimate to invoice. Event sourcing captures every transition with full context, enabling auditing, timeline reconstruction, and undo capabilities.
- **Production floor scans are events by nature.** Every barcode scan at a machine station is an event. Storing these natively rather than as UPDATE operations preserves the full production timeline.
- **Scheduling conflicts need resolution history.** When a machine breaks down and jobs cascade, event sourcing preserves the original schedule, the disruption event, and the rescheduling decisions.
- **AI features benefit from event streams.** Anomaly detection, estimation accuracy analysis, and demand forecasting all consume event streams as their natural input format.

---

## Event Store Schema (PostgreSQL)

The event store uses PostgreSQL for its transactional guarantees and JSONB support, but the schema is minimal -- the richness lives in the event payloads.

```sql
-- Core event store table
CREATE TABLE event_store (
    id                  BIGSERIAL PRIMARY KEY,
    event_id            UUID NOT NULL DEFAULT gen_random_uuid() UNIQUE,
    -- Aggregate identification
    aggregate_type      VARCHAR(50) NOT NULL,
    aggregate_id        UUID NOT NULL,
    -- Event metadata
    event_type          VARCHAR(100) NOT NULL,
    event_version       INT NOT NULL,
    -- Payload
    data                JSONB NOT NULL,
    metadata            JSONB NOT NULL DEFAULT '{}',
    -- Ordering and causation
    sequence_number     BIGINT NOT NULL,
    causation_id        UUID,
    correlation_id      UUID,
    -- Tenant and user
    organisation_id     UUID NOT NULL,
    user_id             UUID,
    -- Timestamp
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Optimistic concurrency control
    UNIQUE (aggregate_type, aggregate_id, event_version)
);

-- Primary query pattern: load all events for an aggregate
CREATE INDEX idx_event_store_aggregate
    ON event_store(aggregate_type, aggregate_id, event_version);

-- Query pattern: all events for an organisation since a given sequence
CREATE INDEX idx_event_store_org_seq
    ON event_store(organisation_id, sequence_number);

-- Query pattern: events by type for projections
CREATE INDEX idx_event_store_type
    ON event_store(event_type, created_at);

-- Query pattern: correlation tracking
CREATE INDEX idx_event_store_correlation
    ON event_store(correlation_id);

-- Sequence generator for global ordering
CREATE SEQUENCE event_sequence_seq;

-- Snapshot store for aggregates with long event histories
CREATE TABLE aggregate_snapshots (
    aggregate_type      VARCHAR(50) NOT NULL,
    aggregate_id        UUID NOT NULL,
    version             INT NOT NULL,
    state               JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);

-- Dead letter queue for failed event processing
CREATE TABLE dead_letter_queue (
    id                  BIGSERIAL PRIMARY KEY,
    event_id            UUID NOT NULL,
    projection_name     VARCHAR(100) NOT NULL,
    error_message       TEXT,
    retry_count         INT DEFAULT 0,
    last_retry_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Projection checkpoint tracking
CREATE TABLE projection_checkpoints (
    projection_name     VARCHAR(100) PRIMARY KEY,
    last_sequence       BIGINT NOT NULL DEFAULT 0,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Aggregate Definitions and Event Types

### Aggregate: Customer

```
Aggregate: Customer
Identity: customer_id (UUID)

Events:
├── CustomerRegistered
│   { company_name, first_name, last_name, email, phone, address, payment_terms_days }
├── CustomerUpdated
│   { changed_fields: { field_name: { old, new } } }
├── CustomerContactAdded
│   { contact_id, first_name, last_name, email, role, receives_proofs, receives_invoices }
├── CustomerContactRemoved
│   { contact_id }
├── CustomerPortalAccessGranted
│   { portal_user_id }
├── CustomerCreditLimitChanged
│   { old_limit, new_limit, reason }
└── CustomerDeactivated
    { reason }
```

### Aggregate: Estimate

```
Aggregate: Estimate
Identity: estimate_id (UUID)

Events:
├── EstimateCreated
│   { estimate_number, customer_id, title, description, created_by_user_id }
├── EstimateItemAdded
│   { item_id, description, job_type, quantity, width_mm, height_mm,
│     pricing_method, material_id, machine_id }
├── EstimateItemRemoved
│   { item_id }
├── EstimateCostLineAdded
│   { item_id, cost_line_id, cost_type, description, material_id,
│     quantity, unit_cost, total_cost }
├── EstimatePriced
│   { subtotal, tax_rate, tax_amount, total, margin_percent, markup_percent,
│     item_prices: [{ item_id, unit_price, total_price }] }
├── EstimateRepriced
│   { reason, old_total, new_total, changed_items: [...] }
├── EstimateSentToCustomer
│   { sent_to_email, sent_at, valid_until }
├── EstimateApprovedByCustomer
│   { approved_by, approval_signature, approved_at }
├── EstimateRejectedByCustomer
│   { rejected_by, rejection_reason }
├── EstimateExpired
│   { expired_at }
├── EstimateConvertedToJob
│   { job_id, job_number }
└── EstimateAnomalyFlagged
    { anomaly_type, confidence_score, similar_jobs: [...], recommendation }
```

### Aggregate: Job

```
Aggregate: Job
Identity: job_id (UUID)

Events:
├── JobCreated
│   { job_number, customer_id, estimate_id, title, description, priority,
│     date_required, items: [{ item_id, description, quantity, material_id }] }
├── JobStatusChanged
│   { old_status, new_status, changed_by_user_id, reason }
├── JobItemAdded
│   { item_id, description, quantity, width_mm, height_mm, material_id }
├── JobItemRemoved
│   { item_id, reason }
├── JobPriorityChanged
│   { old_priority, new_priority, reason }
├── JobAssigned
│   { assigned_to_user_id, sales_user_id }
├── JobOutsourced
│   { supplier_id, outsource_cost, po_number, expected_return_date }
├── JobOutsourceReceived
│   { received_at, received_by_user_id, quality_notes }
│
├── ProductionStepDefined
│   { step_id, item_id, step_order, step_type, machine_id,
│     estimated_duration_minutes }
├── ProductionStepStarted
│   { step_id, started_at, operator_user_id, machine_id }
├── ProductionStepPaused
│   { step_id, paused_at, reason }
├── ProductionStepResumed
│   { step_id, resumed_at }
├── ProductionStepCompleted
│   { step_id, completed_at, actual_duration_minutes, operator_user_id }
├── ProductionStepFailed
│   { step_id, failed_at, reason, rework_required }
├── ProductionScanRecorded
│   { step_id, scan_type, scanned_by_user_id, device_info, scanned_at }
│
├── MaterialConsumed
│   { material_id, item_id, quantity_used, waste_quantity, unit_cost,
│     total_cost, recorded_by_user_id }
├── TimeRecorded
│   { user_id, item_id, step_id, start_time, end_time, duration_minutes,
│     hourly_rate, total_cost }
│
├── ArtworkUploaded
│   { file_id, item_id, file_name, file_type, file_size_bytes,
│     storage_path, width_mm, height_mm, resolution_dpi, colour_space }
├── ArtworkPreflightCompleted
│   { file_id, status, report }
├── ProofCreated
│   { proof_id, item_id, file_id, version }
├── ProofSentToCustomer
│   { proof_id, sent_to_email, sent_at }
├── ProofApproved
│   { proof_id, approved_by, approved_at }
├── ProofRejected
│   { proof_id, rejected_by, rejection_reason }
├── ProofRevisionRequested
│   { proof_id, annotations: [{ page, x, y, content }] }
├── ProofChased
│   { proof_id, chase_count, chased_at }
│
├── JobReadyForDelivery
│   { ready_at }
├── JobDeliveryAssigned
│   { delivery_id }
├── JobDelivered
│   { delivered_at, pod_signature, pod_photo_urls, received_by }
├── JobCompleted
│   { completed_at, actual_cost, margin_percent }
│
├── JobInvoiceGenerated
│   { invoice_id, invoice_number, total }
├── JobOnHold
│   { reason, placed_by_user_id }
├── JobResumedFromHold
│   { resumed_by_user_id }
└── JobCancelled
    { reason, cancelled_by_user_id, cancellation_fee }
```

### Aggregate: Schedule

```
Aggregate: Schedule
Identity: organisation_id (UUID) — single aggregate per organisation

Events:
├── ScheduleEntryCreated
│   { entry_id, job_id, item_id, step_id, machine_id, assigned_user_id,
│     scheduled_start, scheduled_end, setup_minutes, changeover_minutes }
├── ScheduleEntryMoved
│   { entry_id, old_start, old_end, new_start, new_end, old_machine_id,
│     new_machine_id, reason }
├── ScheduleEntryCancelled
│   { entry_id, reason }
├── ScheduleEntryCompleted
│   { entry_id, actual_start, actual_end }
├── MachineDowntimeRecorded
│   { machine_id, reason, start_time, end_time }
├── ScheduleRebalanced
│   { trigger_reason, affected_entries: [{ entry_id, old_start, new_start }],
│     rebalanced_by }
└── ScheduleConflictDetected
    { entry_id_1, entry_id_2, machine_id, overlap_start, overlap_end }
```

### Aggregate: Inventory

```
Aggregate: InventoryItem
Identity: material_id (UUID)

Events:
├── MaterialRegistered
│   { sku, name, material_type, unit_of_measure, width_mm, height_mm,
│     cost_per_unit, reorder_point, reorder_quantity }
├── StockReceived
│   { quantity, po_id, unit_cost, received_by_user_id }
├── StockConsumed
│   { quantity, job_id, item_id, consumed_at }
├── StockAdjusted
│   { old_quantity, new_quantity, reason, adjusted_by_user_id }
├── StockWasted
│   { quantity, job_id, reason }
├── ReorderPointReached
│   { current_stock, reorder_point, suggested_quantity }
├── PurchaseOrderTriggered
│   { po_id, supplier_id, quantity, estimated_delivery_date }
├── CostUpdated
│   { old_cost, new_cost, effective_date, reason }
└── MaterialDeactivated
    { reason }
```

### Aggregate: Invoice

```
Aggregate: Invoice
Identity: invoice_id (UUID)

Events:
├── InvoiceCreated
│   { invoice_number, customer_id, job_id, issue_date, due_date,
│     lines: [{ description, quantity, unit_price, total }],
│     subtotal, tax_rate, tax_amount, total }
├── InvoiceSent
│   { sent_to_email, sent_at }
├── InvoiceViewed
│   { viewed_at }
├── PaymentReceived
│   { payment_id, amount, payment_method, payment_reference, payment_date }
├── InvoiceFullyPaid
│   { paid_at, total_paid }
├── InvoiceOverdue
│   { days_overdue, amount_outstanding }
├── InvoiceCredited
│   { credit_note_id, credit_amount, reason }
├── InvoiceSyncedToQuickBooks
│   { quickbooks_id, synced_at }
└── InvoiceSyncedToXero
    { xero_id, synced_at }
```

### Aggregate: Delivery

```
Aggregate: Delivery
Identity: delivery_id (UUID)

Events:
├── DeliveryPlanned
│   { delivery_date, delivery_type, vehicle_id, driver_user_id,
│     delivery_address, items: [{ job_id, item_id, quantity }] }
├── DeliveryJobAdded
│   { job_id, item_id, quantity }
├── DeliveryJobRemoved
│   { job_id, reason }
├── DeliveryDispatched
│   { dispatched_at, driver_user_id }
├── DeliveryCompleted
│   { delivered_at, pod_signature, pod_photo_urls, received_by }
├── DeliveryPartiallyCompleted
│   { delivered_items: [...], undelivered_items: [...], reason }
├── DeliveryFailed
│   { failed_at, reason }
├── InstallationStarted
│   { started_at, crew_notes }
├── InstallationDayCompleted
│   { day_number, work_completed, photos }
├── InstallationCompleted
│   { completed_at, final_photos, sign_off_by }
└── DeliveryCancelled
    { reason }
```

---

## Read Model Projections (Query Side)

Read models are denormalized views rebuilt by replaying events. They live in separate PostgreSQL tables optimised for specific query patterns.

### Projection 1: Job Dashboard View

```sql
-- Denormalized view for the main job management screen
CREATE TABLE rm_jobs (
    job_id              UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    job_number          VARCHAR(50) NOT NULL,
    title               VARCHAR(255) NOT NULL,
    customer_id         UUID NOT NULL,
    customer_name       VARCHAR(255),
    status              VARCHAR(30) NOT NULL,
    priority            VARCHAR(20),
    date_received       TIMESTAMPTZ,
    date_required       TIMESTAMPTZ,
    date_promised       TIMESTAMPTZ,
    date_completed      TIMESTAMPTZ,
    assigned_to_name    VARCHAR(200),
    sales_person_name   VARCHAR(200),
    quoted_total        NUMERIC(12,2),
    actual_cost         NUMERIC(12,2),
    margin_percent      NUMERIC(5,2),
    is_outsourced       BOOLEAN DEFAULT FALSE,
    outsource_supplier  VARCHAR(255),
    item_count          INT DEFAULT 0,
    proof_status        VARCHAR(20),
    delivery_status     VARCHAR(20),
    invoice_status      VARCHAR(20),
    last_scan_at        TIMESTAMPTZ,
    last_scan_step      VARCHAR(50),
    last_updated_at     TIMESTAMPTZ,
    version             BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_rm_jobs_org_status ON rm_jobs(organisation_id, status);
CREATE INDEX idx_rm_jobs_org_date ON rm_jobs(organisation_id, date_required);
CREATE INDEX idx_rm_jobs_customer ON rm_jobs(customer_id);
```

### Projection 2: Gantt Schedule View

```sql
-- Optimised for rendering the Gantt chart with minimal joins
CREATE TABLE rm_schedule (
    entry_id            UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    machine_id          UUID NOT NULL,
    machine_name        VARCHAR(255),
    machine_type        VARCHAR(50),
    job_id              UUID NOT NULL,
    job_number          VARCHAR(50),
    job_title           VARCHAR(255),
    customer_name       VARCHAR(255),
    step_type           VARCHAR(50),
    scheduled_start     TIMESTAMPTZ NOT NULL,
    scheduled_end       TIMESTAMPTZ NOT NULL,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    setup_minutes       INT,
    status              VARCHAR(20),
    priority            INT,
    assigned_user_name  VARCHAR(200),
    is_delayed          BOOLEAN DEFAULT FALSE,
    delay_minutes       INT DEFAULT 0,
    version             BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_rm_schedule_machine ON rm_schedule(organisation_id, machine_id, scheduled_start);
CREATE INDEX idx_rm_schedule_date ON rm_schedule(organisation_id, scheduled_start, scheduled_end);

-- Machine availability view (includes downtime blocks)
CREATE TABLE rm_machine_timeline (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    machine_id          UUID NOT NULL,
    machine_name        VARCHAR(255),
    block_type          VARCHAR(20) NOT NULL, -- 'job', 'downtime', 'available'
    reference_id        UUID,                 -- job_id or downtime_id
    label               VARCHAR(255),
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ NOT NULL,
    status              VARCHAR(20)
);

CREATE INDEX idx_rm_machine_timeline ON rm_machine_timeline(organisation_id, machine_id, start_time);
```

### Projection 3: Customer Portal View

```sql
-- What the customer sees in their portal
CREATE TABLE rm_customer_jobs (
    job_id              UUID NOT NULL,
    customer_id         UUID NOT NULL,
    job_number          VARCHAR(50),
    title               VARCHAR(255),
    status              VARCHAR(30),
    status_display      VARCHAR(100),
    date_received       TIMESTAMPTZ,
    date_promised       TIMESTAMPTZ,
    current_stage       VARCHAR(50),
    proof_status        VARCHAR(20),
    proof_version       INT,
    delivery_date       DATE,
    delivery_status     VARCHAR(20),
    invoice_number      VARCHAR(50),
    invoice_total       NUMERIC(12,2),
    invoice_status      VARCHAR(20),
    last_updated_at     TIMESTAMPTZ,
    PRIMARY KEY (job_id, customer_id)
);

CREATE INDEX idx_rm_customer_jobs ON rm_customer_jobs(customer_id);
```

### Projection 4: Inventory Levels

```sql
CREATE TABLE rm_inventory (
    material_id         UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    sku                 VARCHAR(50),
    name                VARCHAR(255),
    material_type       VARCHAR(50),
    current_stock       NUMERIC(12,2),
    reserved_stock      NUMERIC(12,2) DEFAULT 0,
    available_stock     NUMERIC(12,2),
    reorder_point       NUMERIC(12,2),
    cost_per_unit       NUMERIC(12,4),
    total_value         NUMERIC(12,2),
    needs_reorder       BOOLEAN DEFAULT FALSE,
    last_received_at    TIMESTAMPTZ,
    last_consumed_at    TIMESTAMPTZ,
    -- Rolling usage stats for demand forecasting
    usage_last_30_days  NUMERIC(12,2) DEFAULT 0,
    usage_last_90_days  NUMERIC(12,2) DEFAULT 0,
    avg_daily_usage     NUMERIC(10,4) DEFAULT 0,
    estimated_days_remaining INT,
    version             BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_rm_inventory_org ON rm_inventory(organisation_id);
CREATE INDEX idx_rm_inventory_reorder ON rm_inventory(organisation_id, needs_reorder) WHERE needs_reorder = TRUE;
```

### Projection 5: Financial and Profitability

```sql
-- Per-job profitability (used by margin reports)
CREATE TABLE rm_job_profitability (
    job_id              UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    job_number          VARCHAR(50),
    customer_id         UUID NOT NULL,
    customer_name       VARCHAR(255),
    job_type            VARCHAR(50),
    estimated_material  NUMERIC(12,2) DEFAULT 0,
    estimated_labour    NUMERIC(12,2) DEFAULT 0,
    estimated_machine   NUMERIC(12,2) DEFAULT 0,
    estimated_outsource NUMERIC(12,2) DEFAULT 0,
    estimated_total     NUMERIC(12,2) DEFAULT 0,
    actual_material     NUMERIC(12,2) DEFAULT 0,
    actual_labour       NUMERIC(12,2) DEFAULT 0,
    actual_machine      NUMERIC(12,2) DEFAULT 0,
    actual_outsource    NUMERIC(12,2) DEFAULT 0,
    actual_total        NUMERIC(12,2) DEFAULT 0,
    quoted_price        NUMERIC(12,2) DEFAULT 0,
    invoiced_amount     NUMERIC(12,2) DEFAULT 0,
    gross_margin        NUMERIC(12,2) DEFAULT 0,
    margin_percent      NUMERIC(5,2) DEFAULT 0,
    estimation_accuracy NUMERIC(5,2),
    completed_at        TIMESTAMPTZ,
    version             BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_rm_profitability_org ON rm_job_profitability(organisation_id, completed_at);

-- Estimator accuracy tracking
CREATE TABLE rm_estimator_accuracy (
    user_id             UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    period_month        DATE NOT NULL,
    job_type            VARCHAR(50),
    jobs_estimated      INT DEFAULT 0,
    avg_accuracy_pct    NUMERIC(5,2),
    total_estimated     NUMERIC(12,2) DEFAULT 0,
    total_actual        NUMERIC(12,2) DEFAULT 0,
    overestimated_count INT DEFAULT 0,
    underestimated_count INT DEFAULT 0,
    PRIMARY KEY (user_id, period_month, job_type)
);
```

### Projection 6: Production Timeline (Event Replay View)

```sql
-- Full timeline of a job for audit and customer visibility
CREATE TABLE rm_job_timeline (
    id                  BIGSERIAL PRIMARY KEY,
    job_id              UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    event_type          VARCHAR(100) NOT NULL,
    timestamp           TIMESTAMPTZ NOT NULL,
    actor_name          VARCHAR(200),
    description         TEXT NOT NULL,
    details             JSONB,
    is_customer_visible BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_rm_job_timeline ON rm_job_timeline(job_id, timestamp);
```

---

## Command Handlers (Write Side)

Commands are validated, then emit events. Example command handler pseudocode:

```
Command: ApproveEstimate
├── Validate: estimate exists, status is 'sent', not expired
├── Validate: customer has authority to approve
├── Emit: EstimateApprovedByCustomer
└── Side effect: Trigger CreateJob command

Command: RecordProductionScan
├── Validate: job exists, step exists, step is in valid state for scan_type
├── Validate: user has operator role
├── Emit: ProductionScanRecorded
├── If scan_type == 'start': Emit ProductionStepStarted
├── If scan_type == 'complete': Emit ProductionStepCompleted
└── Side effect: Check if all steps complete → Emit JobReadyForDelivery

Command: RebalanceSchedule
├── Load all ScheduleEntries for affected machines
├── Calculate cascade impact of delay/downtime
├── Emit: ScheduleRebalanced (with all moves)
└── Side effect: Notify affected job owners via WebSocket

Command: RecordMaterialConsumption
├── Validate: job exists, material exists, sufficient stock
├── Emit: MaterialConsumed (on Job aggregate)
├── Emit: StockConsumed (on InventoryItem aggregate)
└── Side effect: Check reorder point → Emit ReorderPointReached
```

---

## Event Processing Pipeline

```
Event Store
    │
    ├──> Projection Rebuilder (async)
    │    ├── JobDashboardProjection
    │    ├── ScheduleProjection
    │    ├── CustomerPortalProjection
    │    ├── InventoryProjection
    │    ├── ProfitabilityProjection
    │    └── TimelineProjection
    │
    ├──> Integration Event Publisher (async)
    │    ├── QuickBooks/Xero sync (on InvoiceCreated, PaymentReceived)
    │    ├── Email notifications (on ProofSentToCustomer, InvoiceSent)
    │    ├── SMS notifications (on DeliveryDispatched, DeliveryCompleted)
    │    └── Webhook dispatch (on configurable events)
    │
    ├──> AI/ML Event Consumer (async)
    │    ├── Estimation anomaly detection (on EstimatePriced)
    │    ├── Demand forecasting (on StockConsumed, JobCreated)
    │    ├── Schedule optimisation (on MachineDowntimeRecorded)
    │    └── Estimator accuracy tracking (on JobCompleted)
    │
    └──> JDF/XJDF Bridge (async)
         ├── Generate XJDF job tickets (on JobCreated, ProductionStepDefined)
         └── Consume JMF machine status messages → emit MachineStatusChanged
```

---

## Pros and Cons

### Pros

1. **Complete audit trail for every job.** Every state change -- from estimate creation through proof approval, each barcode scan, and final delivery sign-off -- is preserved as an immutable event. This is invaluable for dispute resolution, compliance, and understanding production bottlenecks.

2. **Natural fit for barcode/QR scan workflows.** Shop floor scans are inherently event-like. Instead of mutating a status column, each scan is a first-class event that can be replayed, analysed, and correlated with machine utilisation data.

3. **Temporal queries without complexity.** "What was the schedule at 3pm yesterday before the machine broke down?" is answered by replaying events up to that timestamp. This is difficult to achieve with a traditional update-in-place model.

4. **Independent read model scaling.** The Gantt schedule projection can be served from a Redis cache or a dedicated read replica, completely independent of the write path. High-frequency Gantt polling does not contend with job creation or barcode scanning writes.

5. **AI/ML pipeline integration.** Event streams are the natural input for anomaly detection (flagging underpriced estimates), demand forecasting (predicting substrate consumption), and estimator accuracy tracking. No ETL pipeline is needed -- the events are the data.

6. **Resilient integration.** If the QuickBooks sync fails, the InvoiceCreated event remains in the store. A retry consumer can pick it up later. No data is lost, and the core system is unaffected.

7. **Schedule rebalancing with history.** When a machine goes down, the ScheduleRebalanced event captures every entry that moved, enabling comparison of original vs. adjusted schedules and root-cause analysis of delivery delays.

8. **Projection rebuild capability.** If a new reporting requirement emerges (e.g., "substrate waste per job type per month"), a new projection can be built by replaying the full event history without any schema changes to the source data.

### Cons

1. **Higher implementation complexity.** Event sourcing requires building command handlers, event handlers, projections, snapshot logic, and idempotent event processing. The development team needs experience with this pattern or significant ramp-up time.

2. **Eventual consistency in read models.** After a barcode scan, the Gantt schedule view may take milliseconds to seconds to update. For a fast-paced shop floor, this latency must be carefully managed (e.g., optimistic UI updates, WebSocket push on projection rebuild).

3. **Event schema evolution.** As the system evolves, event schemas change. Adding a field to `JobCreated` in v2 requires upcasting logic for v1 events during replay. This adds a maintenance burden that grows with the event catalogue.

4. **Snapshot management for long-lived aggregates.** A job with 200+ events (common for complex signage installation jobs spanning weeks) requires snapshotting to avoid replaying all events on every command. Snapshot frequency and invalidation must be tuned.

5. **Debugging complexity.** Understanding the current state of a job requires tracing through potentially dozens of events. Developers must build tooling (event viewers, aggregate state inspectors) that would be unnecessary with a simple row-per-job model.

6. **Storage growth.** The event store grows monotonically. A busy shop processing 50 jobs/day with 15-20 events per job generates ~300,000 events/year. While PostgreSQL handles this easily, the read model tables must also be maintained.

7. **Integration complexity with accounting systems.** QuickBooks and Xero expect CRUD-style API calls, not events. A translation layer must map invoice events to REST API calls, handle idempotency, and reconcile failures.

8. **Overkill for simple lookups.** Queries like "list all customers" or "show material catalogue" do not benefit from event sourcing. These reference data entities are better served by simple CRUD tables alongside the event-sourced aggregates.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event store | PostgreSQL 16+ with JSONB (simple, transactional) or EventStoreDB (purpose-built, projections built in) |
| Message broker | Apache Kafka or NATS JetStream for event distribution to projections and integrations |
| Read model store | PostgreSQL for structured projections; Redis for hot data (Gantt cache, inventory levels) |
| Projection framework | Custom async workers (Node.js/TypeScript) or Marten (C#/.NET) |
| Snapshot store | PostgreSQL (same instance as event store) |
| API layer | REST/GraphQL for read models; command endpoints for writes |
| Real-time updates | WebSocket push when projections rebuild (NATS or Socket.io) |
| Event schema registry | JSON Schema with version tracking; upcaster chain for backward compatibility |
| Monitoring | Event lag dashboard (projection checkpoint vs. latest sequence number) |

---

## Migration and Scaling Considerations

### Initial Deployment
- Use PostgreSQL as both event store and read model store for simplicity.
- Start with synchronous projection rebuilds (rebuild projections in the same transaction as event persistence) for strong consistency. Move to async later when scale demands it.
- Implement snapshots from the start for the Job aggregate (snapshot every 50 events).
- Build an event replay CLI tool for rebuilding projections during development.

### Growth Phase
- Move projection rebuilding to async workers consuming from a Kafka topic or NATS subject.
- Add Redis caching for the Gantt schedule read model to handle frequent polling.
- Implement event archiving: move events older than 2 years to cold storage (S3/Parquet) while keeping projection state intact.
- Add a projection lag monitoring dashboard to detect stale read models.

### Scale Phase (Multi-Tenant SaaS)
- Partition the event store by `organisation_id` for tenant isolation.
- Consider EventStoreDB for its built-in projections, persistent subscriptions, and optimised append-only storage.
- Deploy projection workers per tenant or per tenant tier (dedicated workers for high-volume tenants).
- Implement event compaction for completed jobs: replace the full event stream with a single `JobArchived` event containing the final state, keeping storage bounded.

### Migration from Existing Systems
- Build an event importer that converts existing job records into synthetic event streams (e.g., a completed job becomes `JobCreated` + `JobStatusChanged` + `JobCompleted`).
- Run the event-sourced system in parallel with the existing system during migration, comparing outputs.
- Projection rebuild from scratch should take less than 1 hour for up to 1 million events on a single-node PostgreSQL instance.
