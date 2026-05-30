# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

> Project: Printing & Signage Management (Candidate #463)
> Generated: 2026-05-25

---

## Approach Summary

A hybrid model that uses normalised relational tables for stable, high-integrity data (customers, invoices, payments, machines, schedule entries) and PostgreSQL JSONB columns for data that varies by job type, customer workflow, or production configuration. This approach recognises a fundamental characteristic of the print and signage domain: while the business workflow (estimate → proof → produce → deliver → invoice) is universal, the details at each stage vary enormously depending on whether the job is a stack of business cards, a building-wrap, a set of vehicle graphics, or a CNC-cut acrylic sign.

Key design principles:
- **Relational for what is always the same:** customer records, financial transactions, machine definitions, schedule slots, user accounts.
- **JSONB for what varies:** job specifications, estimation formulas, production step configurations, material attributes, preflight results, delivery requirements, and AI model outputs.
- **GIN indexes on JSONB:** Targeted indexing on frequently queried JSONB paths to maintain query performance without full normalisation.
- **JSON Schema validation:** Application-layer validation of JSONB content against registered schemas to prevent unstructured data sprawl while retaining flexibility.

---

## Schema Definition

### Module 1: Core Relational Tables (Stable Data)

```sql
-- Organisations, users, and tenancy (fully relational — never varies)
CREATE TABLE organisations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL UNIQUE,
    contact_email       VARCHAR(255),
    phone               VARCHAR(30),
    address             JSONB NOT NULL DEFAULT '{}',
    -- address: { line_1, line_2, city, state, postal_code, country_code }
    currency_code       CHAR(3) DEFAULT 'AUD',
    timezone            VARCHAR(50) DEFAULT 'Australia/Sydney',
    tax_config          JSONB NOT NULL DEFAULT '{"default_rate": 0.10, "tax_id": null}',
    branding            JSONB DEFAULT '{}',
    -- branding: { logo_url, primary_colour, quote_footer_text, invoice_terms }
    feature_flags       JSONB DEFAULT '{}',
    -- feature_flags: { ai_estimation: true, web2print: false, ... }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    email               VARCHAR(255) NOT NULL,
    password_hash       VARCHAR(255),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    role                VARCHAR(50) NOT NULL,
    permissions         JSONB DEFAULT '[]',
    -- permissions: ["estimates.create", "jobs.view", "schedule.edit", ...]
    phone               VARCHAR(30),
    preferences         JSONB DEFAULT '{}',
    -- preferences: { default_dashboard, notification_channels, timezone }
    is_active           BOOLEAN DEFAULT TRUE,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organisation_id, email)
);

CREATE INDEX idx_users_org ON users(organisation_id);

CREATE TABLE customers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    customer_type       VARCHAR(20) DEFAULT 'company',
    company_name        VARCHAR(255),
    primary_contact     JSONB NOT NULL DEFAULT '{}',
    -- primary_contact: { first_name, last_name, email, phone, mobile }
    billing_address     JSONB DEFAULT '{}',
    shipping_address    JSONB DEFAULT '{}',
    -- addresses: { line_1, line_2, city, state, postal_code, country_code }
    additional_contacts JSONB DEFAULT '[]',
    -- additional_contacts: [{ name, email, role, receives_proofs, receives_invoices }]
    financial_terms     JSONB NOT NULL DEFAULT '{"payment_terms_days": 30}',
    -- financial_terms: { payment_terms_days, credit_limit, tax_exempt, tax_id, discount_percent }
    tags                TEXT[] DEFAULT '{}',
    custom_fields       JSONB DEFAULT '{}',
    -- custom_fields: Org-defined per-customer metadata
    portal_access       BOOLEAN DEFAULT FALSE,
    portal_user_id      UUID REFERENCES users(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_customers_org ON customers(organisation_id);
CREATE INDEX idx_customers_company ON customers(organisation_id, company_name);
CREATE INDEX idx_customers_tags ON customers USING GIN(tags);
```

### Module 2: Materials and Machines (Relational + JSONB Attributes)

```sql
CREATE TABLE materials (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    sku                 VARCHAR(50),
    name                VARCHAR(255) NOT NULL,
    material_type       VARCHAR(50) NOT NULL,
    -- material_type: 'substrate', 'ink', 'toner', 'laminate', 'vinyl',
    --   'mounting_board', 'hardware', 'consumable', 'other'

    -- Common relational fields (queried/filtered/aggregated frequently)
    unit_of_measure     VARCHAR(20) NOT NULL DEFAULT 'sheet',
    cost_per_unit       NUMERIC(12,4) NOT NULL DEFAULT 0,
    current_stock       NUMERIC(12,2) DEFAULT 0,
    reorder_point       NUMERIC(12,2) DEFAULT 0,
    is_active           BOOLEAN DEFAULT TRUE,

    -- Variable attributes that differ by material type (JSONB)
    dimensions          JSONB DEFAULT '{}',
    -- For substrates: { width_mm, height_mm, thickness_microns, weight_gsm }
    -- For vinyl: { width_mm, roll_length_m, adhesive_type, durability_years }
    -- For ink: { colour, ink_type, volume_ml, compatible_machines: [...] }

    colour_profile      JSONB DEFAULT '{}',
    -- { icc_profile_name, colour_space, coated, finish_type }

    supplier_info       JSONB DEFAULT '{}',
    -- { preferred_supplier_id, supplier_part_number, lead_time_days,
    --   alternative_suppliers: [{ supplier_id, part_number }] }

    custom_attributes   JSONB DEFAULT '{}',
    -- Org-defined material metadata: { fire_rating, indoor_outdoor,
    --   eco_certification, min_order_qty, ... }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_materials_org_type ON materials(organisation_id, material_type);
CREATE INDEX idx_materials_sku ON materials(organisation_id, sku);
CREATE INDEX idx_materials_stock ON materials(organisation_id, current_stock)
    WHERE current_stock <= reorder_point;
-- GIN index for querying JSONB attributes
CREATE INDEX idx_materials_dimensions ON materials USING GIN(dimensions);
CREATE INDEX idx_materials_custom ON materials USING GIN(custom_attributes);

CREATE TABLE suppliers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(255) NOT NULL,
    contact             JSONB NOT NULL DEFAULT '{}',
    -- { name, email, phone, address: {...} }
    terms               JSONB DEFAULT '{"payment_terms_days": 30}',
    capabilities        JSONB DEFAULT '[]',
    -- For trade printers: ["wide_format", "offset", "finishing", "installation"]
    is_trade_printer    BOOLEAN DEFAULT FALSE,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE machines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(255) NOT NULL,
    machine_type        VARCHAR(50) NOT NULL,
    -- 'offset_press', 'digital_press', 'wide_format_printer', 'cutter',
    -- 'laminator', 'folder', 'binder', 'cnc_router', 'vinyl_cutter',
    -- 'engraver', 'led_channel_bender', 'other'

    -- Common relational fields for scheduling and costing
    hourly_rate         NUMERIC(10,2) DEFAULT 0,
    setup_cost          NUMERIC(10,2) DEFAULT 0,
    setup_time_minutes  INT DEFAULT 15,
    status              VARCHAR(20) DEFAULT 'available',
    is_active           BOOLEAN DEFAULT TRUE,

    -- Variable specifications (differ wildly by machine type)
    specifications      JSONB NOT NULL DEFAULT '{}',
    -- For wide_format_printer: { max_width_mm, max_height_mm, speed_sqm_hr,
    --   roll_fed: true, flatbed: true, resolution_dpi, ink_type,
    --   white_ink: true, uv_cure: true }
    -- For offset_press: { max_width_mm, max_height_mm, num_colours,
    --   perfecting: true, speed_sheets_hr, min_run_length }
    -- For cutter: { max_width_mm, max_height_mm, cutting_type, speed }
    -- For cnc_router: { bed_width_mm, bed_height_mm, max_depth_mm,
    --   spindle_speed, tool_positions }

    -- Cost model (varies by machine)
    cost_model          JSONB NOT NULL DEFAULT '{}',
    -- { click_cost, ink_cost_sqm, substrate_changeover_minutes,
    --   curing_delay_minutes, warmup_minutes }

    -- JDF/XJDF integration settings
    integration         JSONB DEFAULT '{}',
    -- { jdf_device_id, jmf_endpoint_url, fiery_api_url, protocol_version }

    -- Scheduling constraints
    scheduling          JSONB DEFAULT '{}',
    -- { available_shifts: 2, shift_hours: [{ start: "06:00", end: "14:00" },
    --   { start: "14:00", end: "22:00" }], max_daily_hours: 16,
    --   changeover_matrix: { "substrate_A→substrate_B": 20 } }

    location            VARCHAR(100),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_machines_org ON machines(organisation_id, machine_type);
CREATE INDEX idx_machines_specs ON machines USING GIN(specifications);
```

### Module 3: Estimation and Quoting (Heavy JSONB Usage)

```sql
CREATE TABLE estimates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    estimate_number     VARCHAR(50) NOT NULL,
    title               VARCHAR(255) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    -- 'draft', 'sent', 'approved', 'rejected', 'expired', 'converted'

    -- Financial summary (relational for reporting)
    subtotal            NUMERIC(12,2) DEFAULT 0,
    tax_amount          NUMERIC(12,2) DEFAULT 0,
    total               NUMERIC(12,2) DEFAULT 0,
    margin_percent      NUMERIC(5,2),

    -- Detailed line items as JSONB (variable structure per job type)
    items               JSONB NOT NULL DEFAULT '[]',
    /*
    items: [{
        item_id: "uuid",
        description: "Pull-up banner 850x2000mm",
        job_type: "banner",
        quantity: 10,
        pricing_method: "per_unit",      -- or "per_sqm", "per_linear_m", "per_panel"

        -- Specifications (vary by job type)
        specifications: {
            width_mm: 850,
            height_mm: 2000,
            area_sqm: 1.7,
            substrate: { material_id: "uuid", name: "440gsm PVC Banner" },
            finishing: ["hemmed_edges", "grommets_top_bottom"],
            laminate: { material_id: "uuid", name: "Matte cold laminate" },
            print_sides: "single",
            colour_mode: "CMYK",
            resolution_dpi: 720,
            -- For vehicle wraps:
            vehicle_type: null,
            panel_count: null,
            -- For signage:
            mounting_type: null,
            illumination: null,
            -- For brochures:
            pages: null,
            fold_type: null,
            binding_type: null
        },

        -- Cost breakdown
        costs: {
            substrate: { quantity: 17.0, unit_cost: 4.50, total: 76.50 },
            ink: { coverage_pct: 80, sqm: 17.0, cost_per_sqm: 2.10, total: 35.70 },
            laminate: { sqm: 17.0, cost_per_sqm: 1.80, total: 30.60 },
            finishing: [
                { type: "hemming", units: 10, cost_per_unit: 3.00, total: 30.00 },
                { type: "grommets", count: 40, cost_per_unit: 0.50, total: 20.00 }
            ],
            machine_time: { machine_id: "uuid", hours: 1.5, rate: 85.00, total: 127.50 },
            setup: { time_minutes: 20, cost: 35.00 },
            labour: [
                { type: "finishing", hours: 0.5, rate: 45.00, total: 22.50 }
            ],
            outsource: null,
            delivery: { type: "local", cost: 45.00 },
            total_cost: 423.80
        },

        -- Pricing
        unit_price: 59.00,
        total_price: 590.00,
        margin_percent: 28.2
    }]
    */

    -- Approval and conversion tracking
    approval            JSONB DEFAULT '{}',
    -- { valid_until, sent_to_email, sent_at, approved_by, approved_at,
    --   approval_signature, rejection_reason }

    converted_to_job_id UUID,
    created_by_user_id  UUID REFERENCES users(id),
    notes               TEXT,

    -- AI estimation metadata
    ai_metadata         JSONB DEFAULT '{}',
    -- { estimation_method: "template" | "ai_assisted",
    --   similar_jobs: [{ job_id, similarity_score }],
    --   anomaly_score: 0.12, anomaly_flags: [...],
    --   suggested_adjustments: [...] }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_estimates_number ON estimates(organisation_id, estimate_number);
CREATE INDEX idx_estimates_customer ON estimates(customer_id);
CREATE INDEX idx_estimates_status ON estimates(organisation_id, status);
-- Index for querying items within the JSONB array
CREATE INDEX idx_estimates_items ON estimates USING GIN(items);

-- Estimation templates (JSONB-heavy — stores complete template structures)
CREATE TABLE estimate_templates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(255) NOT NULL,
    job_type            VARCHAR(50) NOT NULL,
    description         TEXT,
    is_active           BOOLEAN DEFAULT TRUE,

    -- Template definition (what fields to collect, default values, pricing rules)
    template_definition JSONB NOT NULL,
    /*
    template_definition: {
        fields: [
            { name: "width_mm", type: "number", label: "Width (mm)", required: true, default: 850 },
            { name: "height_mm", type: "number", label: "Height (mm)", required: true, default: 2000 },
            { name: "substrate", type: "material_select", filter: { material_type: "substrate" } },
            { name: "laminate", type: "material_select", filter: { material_type: "laminate" }, required: false },
            { name: "finishing", type: "multi_select", options: ["hemming", "grommets", "eyelets", "pole_pocket"] }
        ],
        pricing_rules: {
            base_method: "per_sqm",
            base_rate: 45.00,
            quantity_breaks: [
                { min_qty: 10, discount_pct: 5 },
                { min_qty: 50, discount_pct: 10 },
                { min_qty: 100, discount_pct: 15 }
            ],
            finishing_costs: {
                hemming: { per_unit: 3.00 },
                grommets: { per_unit: 0.50, default_count_formula: "2 * (width_mm / 500 + height_mm / 500)" }
            },
            minimum_charge: 85.00
        },
        default_machine_id: "uuid",
        default_material_ids: { substrate: "uuid", laminate: "uuid" }
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_templates_org ON estimate_templates(organisation_id, job_type);
```

### Module 4: Jobs and Production (Relational Structure + JSONB Details)

```sql
CREATE TABLE jobs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    estimate_id         UUID REFERENCES estimates(id),
    job_number          VARCHAR(50) NOT NULL,
    title               VARCHAR(255) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
    priority            VARCHAR(20) DEFAULT 'normal',

    -- Key dates (relational for queries and scheduling)
    date_received       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    date_required       TIMESTAMPTZ,
    date_promised       TIMESTAMPTZ,
    date_completed      TIMESTAMPTZ,

    -- Financial summary (relational for reporting)
    quoted_total        NUMERIC(12,2),
    actual_cost         NUMERIC(12,2) DEFAULT 0,
    actual_revenue      NUMERIC(12,2),

    -- Assignment
    assigned_to_user_id UUID REFERENCES users(id),
    sales_user_id       UUID REFERENCES users(id),

    -- Barcode/QR for shop floor
    barcode             VARCHAR(100),

    -- Job items with full specifications (JSONB — variable by job type)
    items               JSONB NOT NULL DEFAULT '[]',
    /*
    items: [{
        item_id: "uuid",
        description: "...",
        quantity: 10,
        specifications: { ... },  -- Same structure as estimate items
        production_route: [
            { step_id: "uuid", step_order: 1, step_type: "printing",
              machine_id: "uuid", estimated_minutes: 90 },
            { step_id: "uuid", step_order: 2, step_type: "laminating",
              machine_id: "uuid", estimated_minutes: 30 },
            { step_id: "uuid", step_order: 3, step_type: "cutting",
              machine_id: "uuid", estimated_minutes: 20 }
        ],
        status: "in_production",
        current_step: 2
    }]
    */

    -- Outsourcing details (JSONB — only populated for outsourced jobs)
    outsourcing         JSONB DEFAULT '{}',
    -- { is_outsourced: true, supplier_id: "uuid", supplier_name: "...",
    --   po_number: "PO-1234", cost: 450.00, sent_at: "...",
    --   expected_return: "...", received_at: "...", quality_notes: "..." }

    -- Actual costs breakdown (JSONB — accumulated during production)
    actual_costs        JSONB DEFAULT '{}',
    -- { materials: [{ material_id, name, quantity, unit_cost, total, waste }],
    --   labour: [{ user_id, step_type, hours, rate, total }],
    --   machine_time: [{ machine_id, hours, rate, total }],
    --   outsource: 0, delivery: 0, total: 0 }

    notes               TEXT,
    tags                TEXT[] DEFAULT '{}',
    custom_fields       JSONB DEFAULT '{}',

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_jobs_number ON jobs(organisation_id, job_number);
CREATE INDEX idx_jobs_org_status ON jobs(organisation_id, status);
CREATE INDEX idx_jobs_customer ON jobs(customer_id);
CREATE INDEX idx_jobs_date_required ON jobs(organisation_id, date_required);
CREATE INDEX idx_jobs_barcode ON jobs(barcode);
CREATE INDEX idx_jobs_tags ON jobs USING GIN(tags);
CREATE INDEX idx_jobs_items ON jobs USING GIN(items);

-- Production scans remain relational (high-frequency, time-series-like data)
CREATE TABLE production_scans (
    id                  BIGSERIAL PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    job_id              UUID NOT NULL REFERENCES jobs(id),
    item_id             UUID NOT NULL,
    step_id             UUID NOT NULL,
    scan_type           VARCHAR(20) NOT NULL,
    -- 'start', 'pause', 'resume', 'complete', 'reject'
    scanned_by_user_id  UUID REFERENCES users(id),
    machine_id          UUID REFERENCES machines(id),
    scanned_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    device_info         VARCHAR(255),
    scan_data           JSONB DEFAULT '{}',
    -- { location, photo_url, reject_reason, quality_score }
    notes               TEXT
) PARTITION BY RANGE (scanned_at);

-- Create partitions by month
CREATE TABLE production_scans_2026_01 PARTITION OF production_scans
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE production_scans_2026_02 PARTITION OF production_scans
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE production_scans_2026_03 PARTITION OF production_scans
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE production_scans_2026_04 PARTITION OF production_scans
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE production_scans_2026_05 PARTITION OF production_scans
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE production_scans_2026_06 PARTITION OF production_scans
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- ... additional partitions created by scheduled job

CREATE INDEX idx_scans_job ON production_scans(job_id, scanned_at);
CREATE INDEX idx_scans_machine ON production_scans(machine_id, scanned_at);
```

### Module 5: Scheduling (Relational for Constraint Enforcement)

```sql
-- Schedule entries are relational because they need exclusion constraints
-- and are heavily queried by date range and machine
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE schedule_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    job_id              UUID NOT NULL REFERENCES jobs(id),
    item_id             UUID,
    step_id             UUID,
    machine_id          UUID NOT NULL REFERENCES machines(id),
    assigned_user_id    UUID REFERENCES users(id),
    scheduled_start     TIMESTAMPTZ NOT NULL,
    scheduled_end       TIMESTAMPTZ NOT NULL,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    status              VARCHAR(20) DEFAULT 'scheduled',
    priority            INT DEFAULT 0,

    -- Variable scheduling metadata (JSONB)
    scheduling_data     JSONB DEFAULT '{}',
    -- { setup_minutes, changeover_minutes, curing_delay_minutes,
    --   substrate_loaded, ink_type, previous_entry_id, dependency_ids: [],
    --   rescheduled_from: { original_start, original_end, reason } }

    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Prevent double-booking at the database level
    EXCLUDE USING gist (
        machine_id WITH =,
        tstzrange(scheduled_start, scheduled_end) WITH &&
    ) WHERE (status NOT IN ('cancelled'))
);

CREATE INDEX idx_schedule_machine_date ON schedule_entries(machine_id, scheduled_start);
CREATE INDEX idx_schedule_org_date ON schedule_entries(organisation_id, scheduled_start, scheduled_end);
CREATE INDEX idx_schedule_job ON schedule_entries(job_id);

CREATE TABLE machine_downtime (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    machine_id          UUID NOT NULL REFERENCES machines(id),
    reason              VARCHAR(50) NOT NULL,
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ NOT NULL,
    details             JSONB DEFAULT '{}',
    -- { technician, fault_code, parts_replaced: [], cost }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_downtime_machine ON machine_downtime(machine_id, start_time);
```

### Module 6: Proofing and File Management

```sql
CREATE TABLE artwork_files (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id              UUID NOT NULL REFERENCES jobs(id),
    item_id             UUID,
    uploaded_by_user_id UUID REFERENCES users(id),
    file_name           VARCHAR(500) NOT NULL,
    file_type           VARCHAR(20),
    file_size_bytes     BIGINT,
    storage_path        VARCHAR(1000) NOT NULL,
    version             INT NOT NULL DEFAULT 1,
    is_current          BOOLEAN DEFAULT TRUE,

    -- Preflight and metadata (JSONB — output varies by tool and file type)
    file_metadata       JSONB DEFAULT '{}',
    -- { width_mm, height_mm, resolution_dpi, colour_space, has_bleed,
    --   bleed_mm, trim_marks, num_pages, font_embedded: true,
    --   spot_colours: ["Pantone 485 C"], transparency: false }

    preflight_result    JSONB DEFAULT '{}',
    -- { status: "passed" | "warnings" | "failed",
    --   checks: [
    --     { check: "resolution", status: "passed", value: "300dpi", min: "150dpi" },
    --     { check: "bleed", status: "warning", message: "Bleed is only 2mm (3mm recommended)" },
    --     { check: "colour_space", status: "passed", value: "CMYK" },
    --     { check: "pdf_x_compliance", status: "passed", version: "PDF/X-4" }
    --   ],
    --   tool: "enfocus_switch", run_at: "..." }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_artwork_job ON artwork_files(job_id);

CREATE TABLE proofs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id              UUID NOT NULL REFERENCES jobs(id),
    item_id             UUID,
    artwork_file_id     UUID REFERENCES artwork_files(id),
    version             INT NOT NULL DEFAULT 1,
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- 'pending', 'sent', 'viewed', 'approved', 'rejected', 'revision_requested'

    -- Interaction history (JSONB — grows as customer interacts)
    interaction_history JSONB DEFAULT '[]',
    -- [{ action: "sent", to: "client@co.com", at: "...", by: "user_id" },
    --  { action: "viewed", at: "..." },
    --  { action: "comment", by: "Client Name", at: "...",
    --    annotations: [{ page: 1, x: 120, y: 340, text: "Make logo bigger" }] },
    --  { action: "rejected", by: "Client Name", at: "...", reason: "Wrong colour" },
    --  { action: "revision_uploaded", file_id: "uuid", at: "..." },
    --  { action: "approved", by: "Jane Smith", at: "...", signature: "base64..." }]

    chase_count         INT DEFAULT 0,
    last_chased_at      TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_proofs_job ON proofs(job_id);
CREATE INDEX idx_proofs_status ON proofs(status);
```

### Module 7: Delivery and Installation

```sql
CREATE TABLE deliveries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    delivery_date       DATE NOT NULL,
    delivery_type       VARCHAR(30) NOT NULL,
    -- 'delivery', 'installation', 'pickup', 'courier', 'post'
    status              VARCHAR(20) DEFAULT 'planned',
    vehicle_id          UUID,
    driver_user_id      UUID REFERENCES users(id),

    -- Delivery address and instructions (JSONB)
    destination         JSONB NOT NULL DEFAULT '{}',
    -- { address: { line_1, line_2, city, postal_code },
    --   contact_name, contact_phone, access_instructions,
    --   site_requirements: ["cherry_picker", "scaffolding"] }

    -- Items in this delivery
    delivery_items      JSONB NOT NULL DEFAULT '[]',
    -- [{ job_id, item_id, job_number, description, quantity }]

    -- Proof of delivery (JSONB — captured on mobile)
    proof_of_delivery   JSONB DEFAULT '{}',
    -- { received_by, signature_data: "base64...",
    --   photos: [{ url, caption, taken_at }],
    --   timestamp, gps_location: { lat, lng },
    --   condition_notes: "All items received in good condition" }

    -- Installation specifics (JSONB — only for installation deliveries)
    installation        JSONB DEFAULT '{}',
    -- { planned_days: 3, actual_days: 2,
    --   subcontractor_id: "uuid", subcontractor_name: "...",
    --   daily_log: [
    --     { day: 1, date: "2026-06-01", crew: ["John", "Mike"],
    --       work_completed: "Mounting brackets installed",
    --       photos: [...], hours: 8 },
    --     { day: 2, date: "2026-06-02", crew: ["John", "Mike"],
    --       work_completed: "Signage panels mounted and wired",
    --       photos: [...], hours: 6, sign_off: { by: "...", signature: "..." } }
    --   ] }

    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_deliveries_org_date ON deliveries(organisation_id, delivery_date);
CREATE INDEX idx_deliveries_status ON deliveries(organisation_id, status);

CREATE TABLE vehicles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(100) NOT NULL,
    registration        VARCHAR(20),
    specifications      JSONB DEFAULT '{}',
    -- { vehicle_type, max_load_kg, internal_length_mm, internal_width_mm,
    --   internal_height_mm, has_tail_lift, has_a_frame }
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE deliveries ADD CONSTRAINT fk_deliveries_vehicle
    FOREIGN KEY (vehicle_id) REFERENCES vehicles(id);
```

### Module 8: Invoicing (Relational for Financial Integrity)

```sql
CREATE TABLE invoices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    job_id              UUID REFERENCES jobs(id),
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

    -- Line items (JSONB — derived from job items)
    lines               JSONB NOT NULL DEFAULT '[]',
    -- [{ description, quantity, unit_price, total, tax_code, job_item_id }]

    -- External accounting sync (JSONB)
    external_sync       JSONB DEFAULT '{}',
    -- { quickbooks: { id, synced_at, sync_status },
    --   xero: { id, synced_at, sync_status } }

    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_invoices_number ON invoices(organisation_id, invoice_number);
CREATE INDEX idx_invoices_customer ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(organisation_id, status);

-- Payments remain fully relational (financial audit requirements)
CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id          UUID NOT NULL REFERENCES invoices(id),
    amount              NUMERIC(12,2) NOT NULL,
    payment_method      VARCHAR(30),
    payment_reference   VARCHAR(255),
    payment_date        DATE NOT NULL,
    external_sync       JSONB DEFAULT '{}',
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
```

### Module 9: Purchase Orders

```sql
CREATE TABLE purchase_orders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    supplier_id         UUID NOT NULL REFERENCES suppliers(id),
    po_number           VARCHAR(50) NOT NULL,
    status              VARCHAR(20) DEFAULT 'draft',
    issue_date          DATE NOT NULL DEFAULT CURRENT_DATE,
    expected_date       DATE,
    subtotal            NUMERIC(12,2) DEFAULT 0,
    tax_amount          NUMERIC(12,2) DEFAULT 0,
    total               NUMERIC(12,2) DEFAULT 0,
    triggered_by_job_id UUID REFERENCES jobs(id),

    -- Line items (JSONB)
    lines               JSONB NOT NULL DEFAULT '[]',
    -- [{ material_id, name, sku, quantity_ordered, quantity_received,
    --    unit_cost, total_cost }]

    -- Receiving log (JSONB — appended as goods arrive)
    receiving_log       JSONB DEFAULT '[]',
    -- [{ received_at, received_by_user_id, items: [
    --    { material_id, quantity, condition_notes }
    -- ]}]

    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_po_number ON purchase_orders(organisation_id, po_number);
```

### Module 10: Activity Log and Notifications

```sql
CREATE TABLE activity_log (
    id                  BIGSERIAL PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    entity_type         VARCHAR(50) NOT NULL,
    entity_id           UUID NOT NULL,
    action              VARCHAR(50) NOT NULL,
    user_id             UUID,
    summary             TEXT NOT NULL,
    details             JSONB DEFAULT '{}',
    -- { changed_fields: { field: { old, new } }, ip_address, user_agent }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_activity_entity ON activity_log(entity_type, entity_id, created_at DESC);
CREATE INDEX idx_activity_org ON activity_log(organisation_id, created_at DESC);

CREATE TABLE notifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    user_id             UUID REFERENCES users(id),
    customer_id         UUID REFERENCES customers(id),
    channel             VARCHAR(20) NOT NULL,
    -- 'in_app', 'email', 'sms', 'webhook'
    notification_type   VARCHAR(50) NOT NULL,
    -- 'proof_ready', 'proof_approved', 'job_complete', 'delivery_dispatched',
    -- 'invoice_sent', 'stock_low', 'schedule_conflict'
    reference_type      VARCHAR(50),
    reference_id        UUID,
    content             JSONB NOT NULL,
    -- { subject, body, template_id, variables: { ... } }
    status              VARCHAR(20) DEFAULT 'pending',
    sent_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications(user_id, created_at DESC);
```

---

## JSONB Schema Validation Strategy

To prevent JSONB columns from becoming unstructured data swamps, the application layer enforces JSON Schema validation.

```sql
-- JSON Schema registry (application-managed)
CREATE TABLE json_schemas (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schema_name         VARCHAR(100) NOT NULL UNIQUE,
    version             INT NOT NULL DEFAULT 1,
    schema_definition   JSONB NOT NULL,
    -- Full JSON Schema 2020-12 definition
    description         TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Example schemas to register:
-- 'estimate_item_v1' — validates each item in estimates.items
-- 'job_item_v1' — validates each item in jobs.items
-- 'machine_specifications_v1' — validates machines.specifications
-- 'material_dimensions_v1' — validates materials.dimensions
-- 'proof_interaction_v1' — validates proofs.interaction_history entries
-- 'delivery_pod_v1' — validates deliveries.proof_of_delivery
-- 'installation_log_v1' — validates deliveries.installation.daily_log entries
```

The application validates JSONB content against the registered schema before writing. This provides the flexibility of schemaless storage with the safety of schema enforcement.

---

## Pros and Cons

### Pros

1. **Natural fit for variable job specifications.** A business card, a vehicle wrap, and a CNC-cut acrylic sign have completely different attributes. JSONB handles this without requiring dozens of nullable columns or an EAV (Entity-Attribute-Value) pattern that destroys query performance.

2. **Single-table reads for job details.** The entire job specification, production route, and cost breakdown live in a single `jobs` row. The job detail page requires one query instead of joining 5-6 tables. This dramatically simplifies the API layer and improves latency.

3. **Estimation template flexibility.** Print shops constantly add new product types. A new template for "channel letter signage" can be created entirely by inserting a row into `estimate_templates` with a new `template_definition` JSONB structure -- no schema migration required.

4. **Relational where it matters.** Financial data (invoices, payments), scheduling (exclusion constraints), and identity (users, customers, organisations) remain fully relational with foreign keys and constraints.

5. **GIN indexing preserves query performance.** Queries like "find all jobs using 440gsm PVC Banner substrate" or "list all machines with flatbed capability" work efficiently via GIN indexes on JSONB columns.

6. **Preflight results as structured data.** Preflight check output varies by tool (Enfocus Switch, custom validators) and file type. JSONB accommodates this naturally without a rigid preflight result schema.

7. **Installation daily logs.** Multi-day signage installations produce variable daily reports with photos, crew assignments, and work summaries. JSONB arrays capture this naturally.

8. **Reduced migration burden.** Adding a new field to job specifications (e.g., "eco_certification" for substrate) requires no database migration -- just an application code update and JSON Schema version bump.

### Cons

1. **No referential integrity within JSONB.** A `material_id` referenced inside a JSONB cost breakdown is not enforced by a foreign key. If a material is deleted, stale references inside JSONB documents persist silently.

2. **Query complexity for JSONB aggregation.** "Total substrate cost across all jobs this month" requires `jsonb_array_elements` and JSONB path extraction, which is more complex and slower than summing a relational column.

3. **GIN index limitations.** GIN indexes support containment queries (`@>`) well but do not support range queries on JSONB values. "Find all jobs where width_mm > 1000" requires a functional index: `CREATE INDEX ON jobs ((items->0->'specifications'->>'width_mm'))`.

4. **Schema drift risk.** Without discipline, different estimate items may use subtly different JSONB structures (e.g., `cost_per_sqm` vs. `sqm_cost`). JSON Schema validation mitigates this but adds application complexity.

5. **JSONB updates are full-document rewrites.** Updating a single field within a JSONB document (e.g., changing the status of one item in `jobs.items`) rewrites the entire JSONB column. For very large documents (jobs with 50+ items), this creates write amplification.

6. **Backup and restore complexity.** JSONB data is harder to inspect in raw database dumps compared to normalised tables. Troubleshooting data issues requires JSON-aware tooling.

7. **ORM friction.** Most ORMs handle JSONB with varying levels of support. Prisma's JSONB support is basic; SQLAlchemy handles it well. The development team must be comfortable writing raw SQL for complex JSONB queries.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ with btree_gist extension |
| JSONB validation | AJV (TypeScript) or jsonschema (Python) at the application layer |
| ORM | Drizzle ORM or Kysely (TypeScript) — both have strong JSONB support; or SQLAlchemy (Python) |
| API layer | REST with OpenAPI 3.1; JSONB columns map directly to nested API response objects |
| Search | PostgreSQL GIN indexes for primary queries; Meilisearch for full-text job search |
| File storage | S3-compatible with pre-signed URLs |
| Caching | Redis for schedule view and dashboard caching |
| Real-time | WebSocket for Gantt schedule and job status updates |
| Schema registry | Custom table (json_schemas) with application-layer validation |

---

## Migration and Scaling Considerations

### Initial Deployment
- Start with a single PostgreSQL instance. The hybrid model reduces table count by ~40% compared to the fully normalised model, simplifying operations.
- Implement JSON Schema validation from day one to prevent data quality issues.
- Create GIN indexes on frequently queried JSONB columns; defer functional indexes until query patterns are established.
- Use PostgreSQL's `jsonb_path_query` (SQL/JSON Path) for complex JSONB queries -- this is more performant than nested `->` operators.

### Growth Phase
- Partition `production_scans` and `activity_log` by month.
- Add functional indexes on hot JSONB paths identified by query monitoring (e.g., `pg_stat_statements`).
- Consider materialised views for reporting that aggregates JSONB data (e.g., monthly substrate cost by job type).
- Add read replicas for the Gantt schedule and dashboard queries.

### Scale Phase (Multi-Tenant SaaS)
- Implement Row-Level Security (RLS) by `organisation_id`.
- Monitor JSONB document sizes; consider extracting large JSONB arrays (e.g., `jobs.items` with 50+ items) into separate relational tables if write amplification becomes measurable.
- Archive completed jobs to a data warehouse, preserving the JSONB structure in Parquet or JSON format for historical analysis.
- Consider TOAST (The Oversized-Attribute Storage Technique) monitoring for large JSONB documents.

### Migration from Fully Normalised Model
- The hybrid model can be adopted incrementally: start with relational tables and progressively move variable-structure data into JSONB columns as job types expand.
- Write a migration script that consolidates normalised sub-tables (e.g., `estimate_items`, `estimate_cost_lines`) into JSONB arrays within the parent table.
- Maintain backward-compatible API endpoints during migration to avoid breaking integrations.
