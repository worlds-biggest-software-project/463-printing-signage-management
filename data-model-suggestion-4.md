# Data Model Suggestion 4: Graph Database (Neo4j) + PostgreSQL Hybrid

> Project: Printing & Signage Management (Candidate #463)
> Generated: 2026-05-25

---

## Approach Summary

A polyglot persistence architecture using Neo4j as the primary graph database for the production workflow core -- job routing, machine scheduling, dependency tracking, and production timeline -- alongside PostgreSQL for financial data, authentication, and inventory quantities. This approach treats the printing and signage domain as fundamentally a graph problem: jobs flow through interconnected production routes, machines have capability relationships with substrates and finishing processes, schedule entries form dependency chains, and outsourced components link external suppliers into the production timeline.

The graph model is specifically motivated by three domain characteristics that relational databases handle poorly:

1. **Production routing is a directed graph.** A signage job might split into three parallel production paths (print panels, fabricate frames, source LEDs) that converge at assembly. Representing this as a graph with dependency edges is natural; representing it in a relational model requires complex self-referencing tables with recursive queries.

2. **Machine-substrate-finishing capability matching is a bipartite graph.** "Which machines can print on 440gsm PVC with UV ink and then laminate with matte cold lamination?" is a traversal across capability edges, not a SQL JOIN.

3. **Schedule cascade analysis requires path traversal.** "If Machine A breaks down at 2pm, which downstream jobs are affected and by how much?" is a graph traversal problem. In a relational model, this requires multiple recursive CTEs.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Application Layer                       │
│   (REST/GraphQL API, WebSocket for real-time updates)        │
├──────────────────────┬──────────────────────────────────────┤
│                      │                                       │
│   Neo4j (Graph DB)   │   PostgreSQL (Relational)             │
│                      │                                       │
│   - Job workflows    │   - Users & authentication            │
│   - Production routes│   - Organisations & tenancy           │
│   - Machine graph    │   - Invoices & payments               │
│   - Schedule graph   │   - Purchase orders                   │
│   - Capability map   │   - Inventory quantities              │
│   - Customer ↔ Job   │   - Material cost history             │
│     relationships    │   - Accounting sync state             │
│   - Outsource links  │   - Activity/audit log                │
│   - Dependency chains│   - Notification queue                │
│                      │   - File metadata                     │
└──────────────────────┴──────────────────────────────────────┘
```

---

## Neo4j Graph Schema

### Node Types

```cypher
// === CORE DOMAIN NODES ===

// Customer node
CREATE CONSTRAINT customer_id IF NOT EXISTS
FOR (c:Customer) REQUIRE c.id IS UNIQUE;

// Properties: id, organisation_id, company_name, primary_contact_name,
//   primary_contact_email, phone, address_city, tags[], portal_access,
//   payment_terms_days, credit_limit, created_at

// ---

// Job node (central entity)
CREATE CONSTRAINT job_id IF NOT EXISTS
FOR (j:Job) REQUIRE j.id IS UNIQUE;

CREATE INDEX job_number IF NOT EXISTS
FOR (j:Job) ON (j.organisation_id, j.job_number);

CREATE INDEX job_status IF NOT EXISTS
FOR (j:Job) ON (j.organisation_id, j.status);

// Properties: id, organisation_id, job_number, title, description,
//   status, priority, date_received, date_required, date_promised,
//   date_completed, quoted_total, actual_cost, barcode, created_at

// ---

// JobItem node (individual product within a job)
CREATE CONSTRAINT job_item_id IF NOT EXISTS
FOR (ji:JobItem) REQUIRE ji.id IS UNIQUE;

// Properties: id, description, quantity, width_mm, height_mm, area_sqm,
//   job_type, pricing_method, unit_price, total_price, status

// ---

// ProductionStep node (single operation in the production route)
CREATE CONSTRAINT step_id IF NOT EXISTS
FOR (ps:ProductionStep) REQUIRE ps.id IS UNIQUE;

// Properties: id, step_order, step_type, status,
//   estimated_duration_minutes, actual_start, actual_end,
//   actual_duration_minutes

// ---

// Machine node
CREATE CONSTRAINT machine_id IF NOT EXISTS
FOR (m:Machine) REQUIRE m.id IS UNIQUE;

// Properties: id, organisation_id, name, machine_type, manufacturer,
//   model, hourly_rate, setup_cost, setup_time_minutes, status,
//   max_width_mm, max_height_mm, speed_sqm_hr, location,
//   jdf_device_id, jmf_endpoint_url

// ---

// Material node (substrates, inks, consumables)
CREATE CONSTRAINT material_id IF NOT EXISTS
FOR (mat:Material) REQUIRE mat.id IS UNIQUE;

// Properties: id, organisation_id, sku, name, material_type,
//   unit_of_measure, cost_per_unit, width_mm, height_mm,
//   weight_gsm, colour_space, icc_profile_name

// ---

// Supplier node
CREATE CONSTRAINT supplier_id IF NOT EXISTS
FOR (s:Supplier) REQUIRE s.id IS UNIQUE;

// Properties: id, organisation_id, name, contact_email, phone,
//   is_trade_printer, capabilities[]

// ---

// Estimate node
CREATE CONSTRAINT estimate_id IF NOT EXISTS
FOR (e:Estimate) REQUIRE e.id IS UNIQUE;

// Properties: id, organisation_id, estimate_number, title, status,
//   subtotal, tax_amount, total, margin_percent, valid_until,
//   created_at

// ---

// EstimateItem node
CREATE CONSTRAINT estimate_item_id IF NOT EXISTS
FOR (ei:EstimateItem) REQUIRE ei.id IS UNIQUE;

// Properties: id, description, job_type, quantity, width_mm, height_mm,
//   pricing_method, material_cost, labour_cost, machine_cost,
//   overhead_cost, total_cost, unit_price, total_price

// ---

// ScheduleBlock node (time slot on a machine)
CREATE CONSTRAINT schedule_block_id IF NOT EXISTS
FOR (sb:ScheduleBlock) REQUIRE sb.id IS UNIQUE;

// Properties: id, scheduled_start, scheduled_end, actual_start,
//   actual_end, setup_minutes, changeover_minutes, status, priority

// ---

// Proof node
CREATE CONSTRAINT proof_id IF NOT EXISTS
FOR (p:Proof) REQUIRE p.id IS UNIQUE;

// Properties: id, version, status, sent_at, responded_at,
//   approved_by, rejection_reason, chase_count

// ---

// ArtworkFile node
CREATE CONSTRAINT artwork_id IF NOT EXISTS
FOR (af:ArtworkFile) REQUIRE af.id IS UNIQUE;

// Properties: id, file_name, file_type, file_size_bytes, storage_path,
//   width_mm, height_mm, resolution_dpi, colour_space, has_bleed,
//   pdf_x_compliant, preflight_status, version

// ---

// Delivery node
CREATE CONSTRAINT delivery_id IF NOT EXISTS
FOR (d:Delivery) REQUIRE d.id IS UNIQUE;

// Properties: id, organisation_id, delivery_date, delivery_type, status,
//   delivery_address, pod_received_by, pod_timestamp

// ---

// User node (lightweight reference — full auth data in PostgreSQL)
CREATE CONSTRAINT user_id IF NOT EXISTS
FOR (u:User) REQUIRE u.id IS UNIQUE;

// Properties: id, organisation_id, full_name, role, email

// ---

// Scan node (barcode/QR scan event on the shop floor)
CREATE CONSTRAINT scan_id IF NOT EXISTS
FOR (sc:Scan) REQUIRE sc.id IS UNIQUE;

// Properties: id, scan_type, scanned_at, device_info, notes

// ---

// DowntimeBlock node (machine unavailability)
CREATE CONSTRAINT downtime_id IF NOT EXISTS
FOR (dt:DowntimeBlock) REQUIRE dt.id IS UNIQUE;

// Properties: id, reason, start_time, end_time, notes
```

### Relationship Types

```cypher
// === CUSTOMER ↔ JOB RELATIONSHIPS ===

// Customer ordered this job
// (:Customer)-[:ORDERED]->(:Job)

// Customer was given this estimate
// (:Customer)-[:RECEIVED_ESTIMATE]->(:Estimate)

// Estimate was converted to a job
// (:Estimate)-[:CONVERTED_TO]->(:Job)
// Properties: converted_at, converted_by_user_id

// Estimate contains items
// (:Estimate)-[:HAS_ITEM]->(:EstimateItem)

// Estimate item specified a material
// (:EstimateItem)-[:USES_MATERIAL]->(:Material)

// Estimate item targeted a machine
// (:EstimateItem)-[:TARGETED_MACHINE]->(:Machine)


// === JOB ↔ PRODUCTION RELATIONSHIPS ===

// Job contains items
// (:Job)-[:CONTAINS]->(:JobItem)

// Job item requires production steps (ordered)
// (:JobItem)-[:REQUIRES_STEP {step_order: 1}]->(:ProductionStep)

// Production steps have ordering dependencies
// (:ProductionStep)-[:FOLLOWED_BY]->(:ProductionStep)

// Production step runs on a machine
// (:ProductionStep)-[:RUNS_ON]->(:Machine)

// Production step assigned to an operator
// (:ProductionStep)-[:ASSIGNED_TO]->(:User)

// Production step consumes material
// (:ProductionStep)-[:CONSUMES {quantity: 17.0, waste: 0.5}]->(:Material)

// Production step recorded a scan
// (:ProductionStep)-[:SCANNED {scan_type: "complete"}]->(:Scan)

// Scan performed by user
// (:Scan)-[:SCANNED_BY]->(:User)


// === MACHINE CAPABILITY GRAPH ===

// Machine can process a material type
// (:Machine)-[:CAN_PROCESS {speed_sqm_hr: 45, quality: "photo"}]->(:Material)

// Machine can perform a step type
// (:Machine)-[:CAN_PERFORM {step_type: "laminating", setup_minutes: 10}]->(:StepType)

// Material requires specific step type for processing
// (:Material)-[:REQUIRES_STEP_TYPE]->(:StepType)

// Machine feeds into another machine (production line topology)
// (:Machine)-[:FEEDS_INTO {transfer_time_minutes: 5}]->(:Machine)


// === SCHEDULING GRAPH ===

// Schedule block occupies a machine
// (:ScheduleBlock)-[:OCCUPIES]->(:Machine)

// Schedule block is for a production step
// (:ScheduleBlock)-[:SCHEDULES]->(:ProductionStep)

// Schedule block is for a job
// (:ScheduleBlock)-[:FOR_JOB]->(:Job)

// Schedule block depends on another (cascade dependencies)
// (:ScheduleBlock)-[:DEPENDS_ON]->(:ScheduleBlock)

// Schedule block follows another on same machine (timeline ordering)
// (:ScheduleBlock)-[:NEXT_ON_MACHINE]->(:ScheduleBlock)

// Machine has downtime block
// (:Machine)-[:HAS_DOWNTIME]->(:DowntimeBlock)


// === OUTSOURCING RELATIONSHIPS ===

// Job item is outsourced to a supplier
// (:JobItem)-[:OUTSOURCED_TO {po_number: "PO-1234", cost: 450.00,
//   sent_at: datetime(), expected_return: datetime()}]->(:Supplier)

// Supplier supplies material
// (:Supplier)-[:SUPPLIES {part_number: "BNR-440", lead_time_days: 3}]->(:Material)


// === PROOFING RELATIONSHIPS ===

// Job has proof
// (:Job)-[:HAS_PROOF]->(:Proof)

// Proof is for a job item
// (:Proof)-[:PROOFS]->(:JobItem)

// Proof uses artwork file
// (:Proof)-[:USES_FILE]->(:ArtworkFile)

// Artwork file uploaded for job
// (:ArtworkFile)-[:UPLOADED_FOR]->(:Job)

// Artwork file uploaded by user
// (:ArtworkFile)-[:UPLOADED_BY]->(:User)

// Proof approved/rejected by customer contact
// (:Proof)-[:APPROVED_BY {approved_at: datetime()}]->(:Customer)
// (:Proof)-[:REJECTED_BY {rejected_at: datetime(), reason: "..."}]->(:Customer)


// === DELIVERY RELATIONSHIPS ===

// Delivery carries job items
// (:Delivery)-[:CARRIES {quantity: 10}]->(:JobItem)

// Delivery assigned to driver
// (:Delivery)-[:DRIVEN_BY]->(:User)

// Delivery uses vehicle (if applicable — vehicle as a node)
// (:Delivery)-[:USES_VEHICLE]->(:Vehicle)

// Delivery to customer
// (:Delivery)-[:DELIVERED_TO]->(:Customer)

// Installation subcontracted
// (:Delivery)-[:SUBCONTRACTED_TO]->(:Supplier)


// === USER ↔ JOB RELATIONSHIPS ===

// User assigned to job
// (:User)-[:ASSIGNED_TO]->(:Job)

// User is the salesperson for job
// (:User)-[:SOLD]->(:Job)

// User created estimate
// (:User)-[:CREATED]->(:Estimate)

// User operates machine
// (:User)-[:OPERATES]->(:Machine)
```

---

## Key Graph Queries (Cypher)

### 1. Production Route Visualisation

```cypher
// Get the full production route for a job, including machine assignments
// and step dependencies
MATCH (j:Job {job_number: "JOB-2024-0142"})-[:CONTAINS]->(ji:JobItem)
MATCH (ji)-[:REQUIRES_STEP]->(ps:ProductionStep)
OPTIONAL MATCH (ps)-[:RUNS_ON]->(m:Machine)
OPTIONAL MATCH (ps)-[:FOLLOWED_BY]->(next:ProductionStep)
RETURN ji.description AS item,
       ps.step_order AS step,
       ps.step_type AS type,
       ps.status AS status,
       m.name AS machine,
       ps.estimated_duration_minutes AS est_minutes,
       next.step_type AS next_step
ORDER BY ji.description, ps.step_order
```

### 2. Machine Capability Matching

```cypher
// Find all machines that can print on a specific substrate
// and then laminate it, with estimated setup time
MATCH (mat:Material {name: "440gsm PVC Banner"})
MATCH (m1:Machine)-[:CAN_PROCESS]->(mat)
WHERE m1.machine_type IN ['wide_format_printer', 'digital_press']
  AND m1.status = 'available'
OPTIONAL MATCH (m1)-[:FEEDS_INTO]->(m2:Machine)
WHERE m2.machine_type = 'laminator'
RETURN m1.name AS printer,
       m1.speed_sqm_hr AS print_speed,
       m1.hourly_rate AS print_rate,
       m2.name AS laminator,
       m2.hourly_rate AS laminate_rate
ORDER BY m1.hourly_rate ASC
```

### 3. Schedule Cascade Impact Analysis

```cypher
// When Machine "HP Latex 800" goes down, find all downstream affected jobs
// via schedule dependency chains
MATCH (m:Machine {name: "HP Latex 800"})
MATCH (sb1:ScheduleBlock)-[:OCCUPIES]->(m)
WHERE sb1.scheduled_start >= datetime('2026-06-01T14:00:00')
  AND sb1.status = 'scheduled'

// Follow the dependency chain to find all cascading impacts
MATCH path = (sb1)-[:DEPENDS_ON*0..10]->(downstream:ScheduleBlock)
MATCH (downstream)-[:FOR_JOB]->(j:Job)
MATCH (downstream)-[:OCCUPIES]->(affected_machine:Machine)

RETURN j.job_number AS affected_job,
       j.title AS job_title,
       j.date_required AS due_date,
       affected_machine.name AS affected_machine,
       downstream.scheduled_start AS originally_scheduled,
       length(path) AS cascade_depth
ORDER BY cascade_depth, downstream.scheduled_start
```

### 4. Customer Job History with Full Lifecycle

```cypher
// Complete relationship graph for a customer's recent jobs
MATCH (c:Customer {id: $customer_id})-[:ORDERED]->(j:Job)
WHERE j.date_received >= datetime('2026-01-01')
OPTIONAL MATCH (j)-[:CONTAINS]->(ji:JobItem)
OPTIONAL MATCH (ji)-[:REQUIRES_STEP]->(ps:ProductionStep)-[:RUNS_ON]->(m:Machine)
OPTIONAL MATCH (j)-[:HAS_PROOF]->(p:Proof)
OPTIONAL MATCH (d:Delivery)-[:CARRIES]->(ji)
RETURN j.job_number, j.title, j.status,
       collect(DISTINCT ji.description) AS items,
       collect(DISTINCT m.name) AS machines_used,
       collect(DISTINCT p.status) AS proof_statuses,
       collect(DISTINCT d.status) AS delivery_statuses
ORDER BY j.date_received DESC
```

### 5. Optimal Production Route Suggestion

```cypher
// Given a job item specification, find the optimal machine sequence
// considering capabilities, current availability, and cost
MATCH (mat:Material {id: $material_id})

// Find printers that can handle this substrate
MATCH (printer:Machine)-[cp:CAN_PROCESS]->(mat)
WHERE printer.machine_type IN ['wide_format_printer', 'digital_press']
  AND printer.status = 'available'
  AND printer.max_width_mm >= $width_mm

// Find available schedule slots on each printer
OPTIONAL MATCH (printer)<-[:OCCUPIES]-(existing:ScheduleBlock)
WHERE existing.scheduled_end > datetime()
  AND existing.status <> 'cancelled'

// Find downstream finishing machines
OPTIONAL MATCH (printer)-[:FEEDS_INTO]->(finisher:Machine)
WHERE finisher.status = 'available'

WITH printer, cp, finisher,
     count(existing) AS current_load,
     printer.hourly_rate AS print_cost

RETURN printer.name AS suggested_printer,
       print_cost,
       current_load AS jobs_queued,
       cp.speed_sqm_hr AS speed,
       finisher.name AS suggested_finisher,
       finisher.hourly_rate AS finishing_cost,
       (print_cost + coalesce(finisher.hourly_rate, 0)) AS total_hourly_cost
ORDER BY current_load ASC, total_hourly_cost ASC
LIMIT 5
```

### 6. Estimator Accuracy via Historical Similar Jobs

```cypher
// Find historical jobs similar to a new estimate for accuracy comparison
MATCH (e:Estimate {id: $estimate_id})-[:HAS_ITEM]->(ei:EstimateItem)
MATCH (ei)-[:USES_MATERIAL]->(mat:Material)

// Find completed jobs using the same material and similar job type
MATCH (past_job:Job {status: 'complete'})-[:CONTAINS]->(past_item:JobItem)
MATCH (past_item)-[:REQUIRES_STEP]->(:ProductionStep)-[:CONSUMES]->(mat)
WHERE past_item.job_type = ei.job_type

// Get the original estimate for comparison
OPTIONAL MATCH (past_est:Estimate)-[:CONVERTED_TO]->(past_job)
OPTIONAL MATCH (past_est)-[:HAS_ITEM]->(past_ei:EstimateItem)
WHERE past_ei.job_type = ei.job_type

RETURN past_job.job_number,
       past_job.quoted_total AS quoted,
       past_job.actual_cost AS actual,
       (past_job.actual_cost / past_job.quoted_total * 100) AS cost_accuracy_pct,
       past_ei.total_price AS estimated_price,
       past_item.quantity AS quantity,
       mat.name AS material
ORDER BY past_job.date_completed DESC
LIMIT 20
```

### 7. Material Usage Network

```cypher
// Trace where a material is used across the production graph
MATCH (mat:Material {id: $material_id})
MATCH path = (mat)<-[:CONSUMES]-(ps:ProductionStep)<-[:REQUIRES_STEP]-(ji:JobItem)<-[:CONTAINS]-(j:Job)
WHERE j.status IN ['scheduled', 'in_production', 'in_finishing']
MATCH (ps)-[:RUNS_ON]->(m:Machine)
RETURN j.job_number, ji.description, ps.step_type,
       m.name AS machine,
       // Sum of material reserved for upcoming jobs
       sum(CASE WHEN j.status IN ['scheduled'] THEN ps.material_quantity ELSE 0 END) AS reserved_qty,
       sum(CASE WHEN j.status IN ['in_production', 'in_finishing'] THEN ps.material_quantity ELSE 0 END) AS in_use_qty
```

---

## PostgreSQL Schema (Financial and Transactional Data)

Financial data stays in PostgreSQL for ACID guarantees, accounting compliance, and integration with QuickBooks/Xero.

```sql
-- Lightweight user table (full auth and profile)
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    email               VARCHAR(255) NOT NULL UNIQUE,
    password_hash       VARCHAR(255),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    role                VARCHAR(50) NOT NULL,
    is_active           BOOLEAN DEFAULT TRUE,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE organisations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL UNIQUE,
    contact_email       VARCHAR(255),
    currency_code       CHAR(3) DEFAULT 'AUD',
    timezone            VARCHAR(50) DEFAULT 'Australia/Sydney',
    settings            JSONB DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Invoices (must be relational for financial audit)
CREATE TABLE invoices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    customer_id         UUID NOT NULL,  -- References Neo4j Customer.id
    job_id              UUID,           -- References Neo4j Job.id
    invoice_number      VARCHAR(50) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    issue_date          DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date            DATE,
    subtotal            NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax_rate            NUMERIC(5,4) DEFAULT 0.10,
    tax_amount          NUMERIC(12,2) DEFAULT 0,
    total               NUMERIC(12,2) NOT NULL DEFAULT 0,
    amount_paid         NUMERIC(12,2) DEFAULT 0,
    balance_due         NUMERIC(12,2) DEFAULT 0,
    quickbooks_id       VARCHAR(100),
    xero_id             VARCHAR(100),
    external_sync_at    TIMESTAMPTZ,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_invoices_number ON invoices(organisation_id, invoice_number);
CREATE INDEX idx_invoices_customer ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(organisation_id, status);

CREATE TABLE invoice_lines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id          UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    line_order          INT DEFAULT 0,
    description         VARCHAR(500) NOT NULL,
    quantity            NUMERIC(12,2) DEFAULT 1,
    unit_price          NUMERIC(12,4) NOT NULL,
    total               NUMERIC(12,2) NOT NULL,
    tax_code            VARCHAR(20),
    job_item_id         UUID,  -- References Neo4j JobItem.id
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id          UUID NOT NULL REFERENCES invoices(id),
    amount              NUMERIC(12,2) NOT NULL,
    payment_method      VARCHAR(30),
    payment_reference   VARCHAR(255),
    payment_date        DATE NOT NULL,
    quickbooks_id       VARCHAR(100),
    xero_id             VARCHAR(100),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);

-- Inventory quantities (transactional updates with ACID guarantees)
CREATE TABLE inventory_levels (
    material_id         UUID PRIMARY KEY,  -- References Neo4j Material.id
    organisation_id     UUID NOT NULL,
    current_stock       NUMERIC(12,2) NOT NULL DEFAULT 0,
    reserved_stock      NUMERIC(12,2) NOT NULL DEFAULT 0,
    reorder_point       NUMERIC(12,2) DEFAULT 0,
    reorder_quantity    NUMERIC(12,2) DEFAULT 0,
    cost_per_unit       NUMERIC(12,4) NOT NULL DEFAULT 0,
    last_received_at    TIMESTAMPTZ,
    last_consumed_at    TIMESTAMPTZ,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_inventory_reorder ON inventory_levels(organisation_id)
    WHERE current_stock <= reorder_point;

-- Material cost history (for dynamic pricing recalculation)
CREATE TABLE material_cost_history (
    id                  BIGSERIAL PRIMARY KEY,
    material_id         UUID NOT NULL,
    old_cost            NUMERIC(12,4),
    new_cost            NUMERIC(12,4) NOT NULL,
    effective_date      DATE NOT NULL,
    reason              VARCHAR(255),
    changed_by_user_id  UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_cost_history ON material_cost_history(material_id, effective_date DESC);

-- Purchase orders
CREATE TABLE purchase_orders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    supplier_id         UUID NOT NULL,  -- References Neo4j Supplier.id
    po_number           VARCHAR(50) NOT NULL,
    status              VARCHAR(20) DEFAULT 'draft',
    issue_date          DATE NOT NULL DEFAULT CURRENT_DATE,
    expected_date       DATE,
    subtotal            NUMERIC(12,2) DEFAULT 0,
    tax_amount          NUMERIC(12,2) DEFAULT 0,
    total               NUMERIC(12,2) DEFAULT 0,
    triggered_by_job_id UUID,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_po_number ON purchase_orders(organisation_id, po_number);

CREATE TABLE purchase_order_lines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id   UUID NOT NULL REFERENCES purchase_orders(id) ON DELETE CASCADE,
    material_id         UUID NOT NULL,
    quantity_ordered    NUMERIC(12,2) NOT NULL,
    quantity_received   NUMERIC(12,2) DEFAULT 0,
    unit_cost           NUMERIC(12,4) NOT NULL,
    total_cost          NUMERIC(12,2) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- File metadata (artwork storage references)
CREATE TABLE artwork_files (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id              UUID NOT NULL,  -- References Neo4j Job.id
    item_id             UUID,           -- References Neo4j JobItem.id
    uploaded_by_user_id UUID REFERENCES users(id),
    file_name           VARCHAR(500) NOT NULL,
    file_type           VARCHAR(20),
    file_size_bytes     BIGINT,
    storage_path        VARCHAR(1000) NOT NULL,
    version             INT NOT NULL DEFAULT 1,
    is_current          BOOLEAN DEFAULT TRUE,
    preflight_status    VARCHAR(20),
    preflight_report    JSONB DEFAULT '{}',
    file_metadata       JSONB DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_artwork_job ON artwork_files(job_id);

-- Activity / audit log
CREATE TABLE activity_log (
    id                  BIGSERIAL PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    user_id             UUID,
    entity_type         VARCHAR(50) NOT NULL,
    entity_id           UUID NOT NULL,
    action              VARCHAR(50) NOT NULL,
    summary             TEXT,
    details             JSONB DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_activity_entity ON activity_log(entity_type, entity_id);
CREATE INDEX idx_activity_org ON activity_log(organisation_id, created_at DESC);

-- Notification queue
CREATE TABLE notifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    user_id             UUID,
    customer_id         UUID,
    channel             VARCHAR(20) NOT NULL,
    notification_type   VARCHAR(50) NOT NULL,
    content             JSONB NOT NULL,
    status              VARCHAR(20) DEFAULT 'pending',
    sent_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Data Synchronisation Between Neo4j and PostgreSQL

The two databases share entity IDs (UUIDs) as the synchronisation key. The application layer is the source of truth for writes, ensuring both databases are updated within the same logical operation.

### Synchronisation Patterns

```
Write Flow:
1. Create Job command →
   a. Write Job node to Neo4j (production graph)
   b. Write financial reference to PostgreSQL (for invoice generation)
   c. Both writes in a saga pattern with compensation on failure

2. Invoice Creation command →
   a. Read Job details from Neo4j (job items, quantities, prices)
   b. Write Invoice to PostgreSQL (financial record)
   c. Create Invoice relationship in Neo4j (for customer portal traversal)

3. Barcode Scan →
   a. Write Scan node and relationships to Neo4j
   b. No PostgreSQL write (scans are pure production graph data)

4. Material Stock Adjustment →
   a. Update inventory_levels in PostgreSQL (transactional)
   b. No Neo4j write (stock quantities are not graph data)

5. Material Cost Change →
   a. Update cost_per_unit in PostgreSQL inventory_levels
   b. Update Material node cost_per_unit in Neo4j
   c. Trigger estimate re-pricing for affected open estimates
```

### Conflict Resolution

```
PostgreSQL is authoritative for:
  - Financial amounts (invoice totals, payment amounts, stock quantities)
  - User authentication and session data
  - External sync state (QuickBooks IDs, Xero IDs)

Neo4j is authoritative for:
  - Job status and workflow state
  - Production step sequences and dependencies
  - Machine capabilities and assignments
  - Schedule entries and dependency chains
  - Customer-job-proof-delivery relationship graph
```

---

## Graph-Powered Features

### 1. AI-Assisted Production Routing

The machine capability graph enables intelligent routing: given a job specification (substrate, size, finishing requirements), traverse the capability graph to find the optimal machine sequence considering current load, cost, and speed.

```cypher
// Suggest production route for a new banner job
MATCH (mat:Material {name: "440gsm PVC Banner"})
MATCH (printer:Machine)-[:CAN_PROCESS]->(mat)
WHERE printer.status = 'available'
  AND printer.max_width_mm >= 1500

// Find finishing chain
MATCH (printer)-[:FEEDS_INTO*1..3]->(finisher:Machine)
WHERE finisher.machine_type IN ['laminator', 'cutter']

// Calculate total route cost
WITH printer, collect(finisher) AS finishing_chain,
     printer.hourly_rate AS print_rate

RETURN printer.name,
       [f IN finishing_chain | f.name] AS finishing_steps,
       print_rate + reduce(cost = 0, f IN finishing_chain | cost + f.hourly_rate) AS total_hourly_cost
ORDER BY total_hourly_cost ASC
```

### 2. Schedule Impact Analysis

When a machine goes down, graph traversal instantly identifies all affected downstream jobs and calculates the cascade delay.

```cypher
// Find cascade impact of machine downtime
MATCH (m:Machine {id: $machine_id})
MATCH (sb:ScheduleBlock)-[:OCCUPIES]->(m)
WHERE sb.scheduled_start >= $downtime_start
  AND sb.scheduled_start < $downtime_end
  AND sb.status = 'scheduled'

// Traverse dependency chain
MATCH path = (sb)-[:DEPENDS_ON*1..20]->(downstream:ScheduleBlock)
MATCH (downstream)-[:FOR_JOB]->(j:Job)

WITH j, downstream, length(path) AS depth,
     duration.between(sb.scheduled_start, downstream.scheduled_start) AS cascade_delay

RETURN j.job_number, j.title, j.date_required,
       downstream.scheduled_start AS affected_slot,
       depth AS cascade_depth,
       cascade_delay AS estimated_delay,
       CASE WHEN downstream.scheduled_end > j.date_required
            THEN true ELSE false END AS will_miss_deadline
ORDER BY depth, downstream.scheduled_start
```

### 3. Customer Relationship Intelligence

Graph traversal reveals customer patterns that are difficult to extract from relational data.

```cypher
// Customer value analysis: jobs, materials used, machines occupied
MATCH (c:Customer {organisation_id: $org_id})-[:ORDERED]->(j:Job)
WHERE j.date_received >= datetime() - duration({months: 12})
OPTIONAL MATCH (j)-[:CONTAINS]->(ji:JobItem)-[:REQUIRES_STEP]->(ps:ProductionStep)
OPTIONAL MATCH (ps)-[:CONSUMES]->(mat:Material)
OPTIONAL MATCH (ps)-[:RUNS_ON]->(m:Machine)

RETURN c.company_name,
       count(DISTINCT j) AS jobs_last_12m,
       sum(j.quoted_total) AS total_revenue,
       avg(j.actual_cost / j.quoted_total * 100) AS avg_cost_ratio,
       collect(DISTINCT mat.name) AS materials_used,
       collect(DISTINCT m.name) AS machines_used,
       collect(DISTINCT ji.job_type) AS job_types
ORDER BY total_revenue DESC
```

---

## Pros and Cons

### Pros

1. **Natural production routing model.** The print production workflow -- where a job splits across parallel paths (print, fabricate, source components) and converges at assembly -- is inherently a directed acyclic graph. Neo4j represents this natively without recursive CTEs or self-referencing tables.

2. **Schedule cascade analysis in milliseconds.** "Which jobs are affected if Machine X goes down?" is a simple depth-first traversal in Neo4j. In PostgreSQL, this requires recursive CTEs that become slow as dependency chains grow.

3. **Machine capability matching is a traversal.** "Find machines that can print on this substrate, then laminate, then cut" is a graph pattern match. In SQL, this requires multiple JOINs across capability tables with complex WHERE clauses.

4. **Rich relationship properties.** Edges carry data: the `CONSUMES` relationship stores quantity and waste; `OUTSOURCED_TO` carries PO number and cost; `FEEDS_INTO` records transfer time between machines. This eliminates junction tables.

5. **AI-friendly graph structure.** Graph neural networks for schedule optimisation, recommendation systems for production routing, and anomaly detection for estimation all benefit from the native graph representation. No ETL transformation from tables to graphs is needed.

6. **Visual debugging.** Neo4j Browser provides instant visual exploration of job production graphs, schedule dependency chains, and machine capability networks. This is invaluable during development and for customer support troubleshooting.

7. **Financial integrity preserved.** Invoices, payments, and inventory quantities remain in PostgreSQL with full ACID guarantees. The graph database handles the workflow; the relational database handles the money.

### Cons

1. **Operational complexity.** Running and maintaining two databases (Neo4j + PostgreSQL) doubles the operational burden: backups, monitoring, upgrades, connection pooling, and disaster recovery all need to be handled for both systems.

2. **Data synchronisation challenges.** Keeping Neo4j and PostgreSQL in sync requires careful application logic. There is no native cross-database transaction. Saga patterns or eventual consistency must be accepted for cross-database operations.

3. **Smaller talent pool.** Neo4j and Cypher query language are less widely known than PostgreSQL and SQL. Hiring developers and DBAs who are proficient in both is harder and more expensive.

4. **No native exclusion constraints for scheduling.** PostgreSQL's `EXCLUDE USING gist` constraint prevents double-booking at the database level. Neo4j has no equivalent -- schedule conflict detection must be implemented in application logic or via APOC triggers.

5. **Cost and licensing.** Neo4j Community Edition is open source but lacks clustering, hot backup, and role-based access control. Neo4j Enterprise is commercially licensed. For a multi-tenant SaaS, this may be a significant cost.

6. **ACID guarantees are weaker.** While Neo4j supports ACID transactions within a single database, cross-database transactions between Neo4j and PostgreSQL require distributed transaction coordination (saga pattern), which adds latency and complexity.

7. **Reporting limitations.** Aggregate financial reports (monthly revenue, AR aging, tax summaries) are better served by SQL. The dual-database model means reporting queries may need to join data from both stores, adding application complexity.

8. **Scaling characteristics differ.** Neo4j scales differently from PostgreSQL. Neo4j Enterprise supports causal clustering, but the operational model (leader-follower with read replicas) is different from PostgreSQL replication, requiring separate expertise.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Graph database | Neo4j 5.x (Community for development; Enterprise for production multi-tenant) |
| Relational database | PostgreSQL 16+ for financial data, auth, and inventory |
| Neo4j driver | Official Neo4j JavaScript/TypeScript driver or neo4j-driver for Node.js |
| ORM (PostgreSQL) | Prisma or Drizzle for the relational side |
| Graph visualisation | Neo4j Browser for development; neovis.js or d3.js for application UI |
| API layer | GraphQL (natural fit for graph traversal on the API surface) |
| Sync mechanism | Application-layer saga coordinator with idempotency keys |
| Caching | Redis for schedule view caching and session management |
| Real-time | WebSocket for Gantt schedule updates and production floor dashboard |
| Search | Neo4j full-text search indexes for job and customer lookups |
| Monitoring | Neo4j Ops Manager + PostgreSQL pg_stat_statements |

---

## Migration and Scaling Considerations

### Initial Deployment
- Deploy Neo4j Community Edition alongside PostgreSQL for development and early production.
- Use the Neo4j JavaScript driver's transaction functions for automatic retry on transient errors.
- Implement idempotent write handlers to safely retry failed cross-database operations.
- Start with a single Neo4j instance; the Community Edition handles up to ~34 billion nodes and ~34 billion relationships.

### Growth Phase
- Upgrade to Neo4j Enterprise for causal clustering (3-node minimum) and hot backup.
- Add Neo4j read replicas for the Gantt schedule view and customer portal graph queries.
- Implement a message queue (NATS or RabbitMQ) between the application layer and the two databases to decouple writes and improve resilience.
- Monitor Neo4j heap usage and page cache hit ratio; graph traversals are memory-intensive.

### Scale Phase (Multi-Tenant SaaS)
- Consider Neo4j Aura (managed cloud service) to offload operational complexity.
- Partition tenants across multiple Neo4j databases (Neo4j 5.x supports multi-database within a single instance) using `organisation_id` as the database selector.
- Implement a read-through cache (Redis) for frequently accessed graph patterns (machine capabilities, active schedule, customer job history).
- Archive completed job subgraphs: detach completed jobs from the active graph and archive node/relationship data to PostgreSQL or a data lake for historical reporting.

### Migration from Relational Model
- Build a graph import pipeline that reads PostgreSQL tables and creates corresponding Neo4j nodes and relationships.
- Start with the machine capability graph and schedule graph (highest-value graph use cases) and migrate other entities incrementally.
- Run both models in parallel during migration, comparing query results for consistency validation.
- Use Neo4j's `LOAD CSV` for bulk initial import; use the Bolt driver for incremental synchronisation.

### Alternative Graph Options
If Neo4j's licensing is a concern, consider these alternatives:
- **Apache AGE (PostgreSQL extension):** Adds graph query capability (openCypher) directly to PostgreSQL, eliminating the dual-database architecture. Less mature than Neo4j but avoids operational complexity.
- **Memgraph:** Open-source, in-memory graph database compatible with Cypher. Better for real-time scheduling queries but requires more memory.
- **Amazon Neptune:** Managed graph database supporting both property graph (openCypher) and RDF (SPARQL). Good for AWS-native deployments.
