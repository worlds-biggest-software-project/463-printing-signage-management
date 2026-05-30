# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Printing & Signage Management (Candidate #463)
> Generated: 2026-05-25

---

## Approach Summary

A fully normalized relational model using PostgreSQL, designed around the core domain entities of a printing and signage MIS: customers, jobs, estimates, production schedules, inventory, deliveries, and invoices. Every entity is in at least Third Normal Form (3NF). Relationships are enforced via foreign keys with cascading rules. This model prioritises data integrity, transactional consistency, and query flexibility over schema agility.

The schema is organised into logical modules that mirror the print shop workflow: CRM and customer management, estimation and quoting, production and scheduling, inventory and procurement, delivery and installation, invoicing and accounting, and proofing and file management.

---

## Schema Definition

### Module 1: Organisation and Tenancy

```sql
-- Multi-tenant support for SaaS deployment
CREATE TABLE organisations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL UNIQUE,
    address_line_1      VARCHAR(255),
    address_line_2      VARCHAR(255),
    city                VARCHAR(100),
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    country_code        CHAR(2) DEFAULT 'AU',
    phone               VARCHAR(30),
    email               VARCHAR(255),
    website             VARCHAR(255),
    tax_id              VARCHAR(50),
    currency_code       CHAR(3) DEFAULT 'AUD',
    timezone            VARCHAR(50) DEFAULT 'Australia/Sydney',
    logo_url            VARCHAR(500),
    settings            JSONB DEFAULT '{}',
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
    role                VARCHAR(50) NOT NULL CHECK (role IN (
        'admin', 'estimator', 'production_manager', 'operator',
        'delivery_driver', 'sales', 'accounts', 'customer'
    )),
    phone               VARCHAR(30),
    is_active           BOOLEAN DEFAULT TRUE,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organisation_id, email)
);

CREATE INDEX idx_users_org ON users(organisation_id);
CREATE INDEX idx_users_role ON users(organisation_id, role);
```

### Module 2: CRM and Customer Management

```sql
CREATE TABLE customers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    company_name        VARCHAR(255),
    first_name          VARCHAR(100),
    last_name           VARCHAR(100),
    email               VARCHAR(255),
    phone               VARCHAR(30),
    mobile              VARCHAR(30),
    address_line_1      VARCHAR(255),
    address_line_2      VARCHAR(255),
    city                VARCHAR(100),
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    country_code        CHAR(2) DEFAULT 'AU',
    tax_exempt          BOOLEAN DEFAULT FALSE,
    tax_id              VARCHAR(50),
    payment_terms_days  INT DEFAULT 30,
    credit_limit        NUMERIC(12,2),
    notes               TEXT,
    portal_access       BOOLEAN DEFAULT FALSE,
    portal_user_id      UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_customers_org ON customers(organisation_id);
CREATE INDEX idx_customers_company ON customers(organisation_id, company_name);

CREATE TABLE customer_contacts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    email               VARCHAR(255),
    phone               VARCHAR(30),
    role                VARCHAR(100),
    is_primary          BOOLEAN DEFAULT FALSE,
    receives_proofs     BOOLEAN DEFAULT FALSE,
    receives_invoices   BOOLEAN DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_customer_contacts ON customer_contacts(customer_id);
```

### Module 3: Products, Substrates, and Materials Catalogue

```sql
-- Substrate and material categories
CREATE TABLE material_categories (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(100) NOT NULL,
    parent_id           UUID REFERENCES material_categories(id),
    sort_order          INT DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Individual substrates and consumables
CREATE TABLE materials (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    category_id         UUID REFERENCES material_categories(id),
    sku                 VARCHAR(50),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    material_type       VARCHAR(50) NOT NULL CHECK (material_type IN (
        'substrate', 'ink', 'toner', 'laminate', 'vinyl',
        'mounting_board', 'consumable', 'hardware', 'other'
    )),
    -- Dimensional attributes for substrates
    unit_of_measure     VARCHAR(20) NOT NULL DEFAULT 'sheet',
    width_mm            NUMERIC(10,2),
    height_mm           NUMERIC(10,2),
    thickness_microns   INT,
    weight_gsm          NUMERIC(8,2),
    -- Cost tracking
    cost_per_unit       NUMERIC(12,4) NOT NULL DEFAULT 0,
    cost_currency       CHAR(3) DEFAULT 'AUD',
    -- ICC and colour profile reference
    icc_profile_name    VARCHAR(255),
    colour_space        VARCHAR(20) CHECK (colour_space IN ('CMYK', 'RGB', 'spot', 'n_channel')),
    -- Stock management
    reorder_point       NUMERIC(12,2) DEFAULT 0,
    reorder_quantity    NUMERIC(12,2) DEFAULT 0,
    current_stock       NUMERIC(12,2) DEFAULT 0,
    stock_location      VARCHAR(100),
    -- Supplier reference
    preferred_supplier_id UUID,
    supplier_part_number VARCHAR(100),
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_materials_org ON materials(organisation_id);
CREATE INDEX idx_materials_type ON materials(organisation_id, material_type);
CREATE INDEX idx_materials_sku ON materials(organisation_id, sku);

-- Suppliers
CREATE TABLE suppliers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(255) NOT NULL,
    contact_name        VARCHAR(200),
    email               VARCHAR(255),
    phone               VARCHAR(30),
    address_line_1      VARCHAR(255),
    city                VARCHAR(100),
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    country_code        CHAR(2) DEFAULT 'AU',
    payment_terms_days  INT DEFAULT 30,
    notes               TEXT,
    is_trade_printer    BOOLEAN DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE materials ADD CONSTRAINT fk_materials_supplier
    FOREIGN KEY (preferred_supplier_id) REFERENCES suppliers(id);
```

### Module 4: Machines and Production Resources

```sql
CREATE TABLE machines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(255) NOT NULL,
    machine_type        VARCHAR(50) NOT NULL CHECK (machine_type IN (
        'offset_press', 'digital_press', 'wide_format_printer',
        'cutter', 'laminator', 'folder', 'binder', 'cnc_router',
        'vinyl_cutter', 'engraver', 'other'
    )),
    manufacturer        VARCHAR(100),
    model               VARCHAR(100),
    -- Capacity and speed
    max_width_mm        NUMERIC(10,2),
    max_height_mm       NUMERIC(10,2),
    speed_sheets_hr     INT,
    speed_sqm_hr        NUMERIC(10,2),
    -- Cost rates for estimation
    setup_cost          NUMERIC(10,2) DEFAULT 0,
    hourly_rate         NUMERIC(10,2) DEFAULT 0,
    click_cost          NUMERIC(10,4) DEFAULT 0,
    -- Scheduling
    available_shifts    INT DEFAULT 1,
    setup_time_minutes  INT DEFAULT 15,
    changeover_time_minutes INT DEFAULT 10,
    -- JDF/XJDF integration
    jdf_device_id       VARCHAR(255),
    jmf_endpoint_url    VARCHAR(500),
    -- Status
    status              VARCHAR(20) DEFAULT 'available' CHECK (status IN (
        'available', 'in_use', 'maintenance', 'offline'
    )),
    location            VARCHAR(100),
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_machines_org ON machines(organisation_id);
CREATE INDEX idx_machines_type ON machines(organisation_id, machine_type);

-- Machine capabilities (which substrates a machine can handle)
CREATE TABLE machine_material_capabilities (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    machine_id          UUID NOT NULL REFERENCES machines(id) ON DELETE CASCADE,
    material_id         UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE,
    notes               TEXT,
    UNIQUE (machine_id, material_id)
);

-- Labour rate categories
CREATE TABLE labour_rates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(100) NOT NULL,
    rate_type           VARCHAR(30) NOT NULL CHECK (rate_type IN (
        'setup', 'operation', 'finishing', 'design', 'installation', 'delivery'
    )),
    hourly_rate         NUMERIC(10,2) NOT NULL,
    overtime_multiplier NUMERIC(4,2) DEFAULT 1.5,
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Module 5: Estimation and Quoting

```sql
CREATE TABLE estimates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    estimate_number     VARCHAR(50) NOT NULL,
    title               VARCHAR(255) NOT NULL,
    description         TEXT,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'sent', 'approved', 'rejected', 'expired', 'converted'
    )),
    -- Pricing
    subtotal            NUMERIC(12,2) DEFAULT 0,
    tax_rate            NUMERIC(5,4) DEFAULT 0.10,
    tax_amount          NUMERIC(12,2) DEFAULT 0,
    total               NUMERIC(12,2) DEFAULT 0,
    margin_percent      NUMERIC(5,2),
    markup_percent      NUMERIC(5,2),
    -- Validity
    valid_until         DATE,
    -- Approval tracking
    approved_at         TIMESTAMPTZ,
    approved_by         VARCHAR(255),
    approval_signature  TEXT,
    -- Conversion to job
    converted_to_job_id UUID,
    -- Estimator reference
    created_by_user_id  UUID REFERENCES users(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_estimates_org ON estimates(organisation_id);
CREATE INDEX idx_estimates_customer ON estimates(customer_id);
CREATE INDEX idx_estimates_status ON estimates(organisation_id, status);
CREATE UNIQUE INDEX idx_estimates_number ON estimates(organisation_id, estimate_number);

-- Estimate line items (individual products/services in a quote)
CREATE TABLE estimate_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    estimate_id         UUID NOT NULL REFERENCES estimates(id) ON DELETE CASCADE,
    item_order          INT NOT NULL DEFAULT 0,
    description         VARCHAR(500) NOT NULL,
    job_type            VARCHAR(50) CHECK (job_type IN (
        'brochure', 'banner', 'poster', 'business_card', 'flyer',
        'vehicle_wrap', 'signage', 'exhibition_display', 'sticker',
        'packaging', 'book', 'custom'
    )),
    quantity            INT NOT NULL DEFAULT 1,
    -- Dimensions (for wide-format)
    width_mm            NUMERIC(10,2),
    height_mm           NUMERIC(10,2),
    area_sqm            NUMERIC(10,4),
    -- Pricing method
    pricing_method      VARCHAR(30) DEFAULT 'fixed' CHECK (pricing_method IN (
        'fixed', 'per_unit', 'per_sqm', 'per_linear_m', 'per_panel'
    )),
    unit_price          NUMERIC(12,4),
    total_price         NUMERIC(12,2),
    -- Estimated costs
    material_cost       NUMERIC(12,2) DEFAULT 0,
    labour_cost         NUMERIC(12,2) DEFAULT 0,
    machine_cost        NUMERIC(12,2) DEFAULT 0,
    overhead_cost       NUMERIC(12,2) DEFAULT 0,
    outsource_cost      NUMERIC(12,2) DEFAULT 0,
    total_cost          NUMERIC(12,2) DEFAULT 0,
    -- Production reference
    estimated_machine_id UUID REFERENCES machines(id),
    estimated_hours     NUMERIC(8,2),
    -- Substrate / material
    material_id         UUID REFERENCES materials(id),
    substrate_description VARCHAR(255),
    finishing_notes     TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_estimate_items ON estimate_items(estimate_id);

-- Estimate cost breakdown lines (BOM for each estimate item)
CREATE TABLE estimate_cost_lines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    estimate_item_id    UUID NOT NULL REFERENCES estimate_items(id) ON DELETE CASCADE,
    cost_type           VARCHAR(30) NOT NULL CHECK (cost_type IN (
        'substrate', 'ink', 'laminate', 'finishing_material',
        'machine_time', 'setup', 'labour', 'outsource',
        'delivery', 'overhead', 'other'
    )),
    description         VARCHAR(255),
    material_id         UUID REFERENCES materials(id),
    quantity            NUMERIC(12,4),
    unit_cost           NUMERIC(12,4),
    total_cost          NUMERIC(12,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_estimate_cost_lines ON estimate_cost_lines(estimate_item_id);

-- Estimation templates for common job types
CREATE TABLE estimate_templates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(255) NOT NULL,
    job_type            VARCHAR(50) NOT NULL,
    description         TEXT,
    default_material_id UUID REFERENCES materials(id),
    default_machine_id  UUID REFERENCES machines(id),
    default_finishing    TEXT,
    cost_formula        TEXT,
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Module 6: Jobs and Production

```sql
CREATE TABLE jobs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    estimate_id         UUID REFERENCES estimates(id),
    job_number          VARCHAR(50) NOT NULL,
    title               VARCHAR(255) NOT NULL,
    description         TEXT,
    priority            VARCHAR(20) DEFAULT 'normal' CHECK (priority IN (
        'low', 'normal', 'high', 'rush'
    )),
    status              VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'artwork_pending', 'proof_pending', 'proof_approved',
        'scheduled', 'in_prepress', 'in_production', 'in_finishing',
        'quality_check', 'ready_for_delivery', 'out_for_delivery',
        'delivered', 'invoiced', 'complete', 'on_hold', 'cancelled'
    )),
    -- Dates
    date_received       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    date_required       TIMESTAMPTZ,
    date_promised       TIMESTAMPTZ,
    date_completed      TIMESTAMPTZ,
    -- Financial
    quoted_total        NUMERIC(12,2),
    actual_cost         NUMERIC(12,2) DEFAULT 0,
    invoice_total       NUMERIC(12,2),
    -- Assignment
    assigned_to_user_id UUID REFERENCES users(id),
    sales_user_id       UUID REFERENCES users(id),
    -- Outsourcing
    is_outsourced       BOOLEAN DEFAULT FALSE,
    outsource_supplier_id UUID REFERENCES suppliers(id),
    outsource_cost      NUMERIC(12,2),
    outsource_po_number VARCHAR(50),
    -- Barcode / QR for shop floor
    barcode             VARCHAR(100),
    qr_code_url         VARCHAR(500),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_jobs_org ON jobs(organisation_id);
CREATE INDEX idx_jobs_customer ON jobs(customer_id);
CREATE INDEX idx_jobs_status ON jobs(organisation_id, status);
CREATE INDEX idx_jobs_date_required ON jobs(organisation_id, date_required);
CREATE UNIQUE INDEX idx_jobs_number ON jobs(organisation_id, job_number);
CREATE INDEX idx_jobs_barcode ON jobs(barcode);

-- Job items (individual products within a job — maps to estimate items)
CREATE TABLE job_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id              UUID NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
    estimate_item_id    UUID REFERENCES estimate_items(id),
    description         VARCHAR(500) NOT NULL,
    quantity            INT NOT NULL DEFAULT 1,
    width_mm            NUMERIC(10,2),
    height_mm           NUMERIC(10,2),
    material_id         UUID REFERENCES materials(id),
    machine_id          UUID REFERENCES machines(id),
    status              VARCHAR(30) DEFAULT 'pending',
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_job_items ON job_items(job_id);

-- Production steps for each job item
CREATE TABLE production_steps (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_item_id         UUID NOT NULL REFERENCES job_items(id) ON DELETE CASCADE,
    step_order          INT NOT NULL,
    step_type           VARCHAR(50) NOT NULL CHECK (step_type IN (
        'prepress', 'rip', 'printing', 'cutting', 'laminating',
        'folding', 'binding', 'mounting', 'welding', 'hemming',
        'grommeting', 'cnc_routing', 'vinyl_application',
        'assembly', 'quality_check', 'packing', 'other'
    )),
    machine_id          UUID REFERENCES machines(id),
    assigned_user_id    UUID REFERENCES users(id),
    status              VARCHAR(20) DEFAULT 'pending' CHECK (status IN (
        'pending', 'scheduled', 'in_progress', 'paused',
        'complete', 'skipped', 'failed'
    )),
    estimated_duration_minutes INT,
    actual_start_at     TIMESTAMPTZ,
    actual_end_at       TIMESTAMPTZ,
    actual_duration_minutes INT,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_production_steps ON production_steps(job_item_id);
CREATE INDEX idx_production_steps_machine ON production_steps(machine_id, status);

-- Shop floor scan events (barcode/QR scans at each step)
CREATE TABLE production_scans (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    production_step_id  UUID NOT NULL REFERENCES production_steps(id),
    job_id              UUID NOT NULL REFERENCES jobs(id),
    scanned_by_user_id  UUID REFERENCES users(id),
    scan_type           VARCHAR(20) NOT NULL CHECK (scan_type IN (
        'start', 'pause', 'resume', 'complete', 'reject'
    )),
    scanned_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    device_info         VARCHAR(255),
    notes               TEXT
);

CREATE INDEX idx_production_scans_step ON production_scans(production_step_id);
CREATE INDEX idx_production_scans_job ON production_scans(job_id);

-- Actual material usage per job (for cost tracking)
CREATE TABLE job_material_usage (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id              UUID NOT NULL REFERENCES jobs(id),
    job_item_id         UUID REFERENCES job_items(id),
    material_id         UUID NOT NULL REFERENCES materials(id),
    quantity_used       NUMERIC(12,4) NOT NULL,
    unit_cost           NUMERIC(12,4),
    total_cost          NUMERIC(12,2),
    waste_quantity      NUMERIC(12,4) DEFAULT 0,
    recorded_by_user_id UUID REFERENCES users(id),
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_job_material_usage ON job_material_usage(job_id);

-- Time tracking per job
CREATE TABLE job_time_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id              UUID NOT NULL REFERENCES jobs(id),
    job_item_id         UUID REFERENCES job_items(id),
    production_step_id  UUID REFERENCES production_steps(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ,
    duration_minutes    INT,
    labour_rate_id      UUID REFERENCES labour_rates(id),
    hourly_rate         NUMERIC(10,2),
    total_cost          NUMERIC(10,2),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_job_time_entries ON job_time_entries(job_id);
```

### Module 7: Scheduling

```sql
-- Gantt schedule entries — one per machine-job-item assignment
CREATE TABLE schedule_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    job_id              UUID NOT NULL REFERENCES jobs(id),
    job_item_id         UUID REFERENCES job_items(id),
    production_step_id  UUID REFERENCES production_steps(id),
    machine_id          UUID NOT NULL REFERENCES machines(id),
    assigned_user_id    UUID REFERENCES users(id),
    -- Time window
    scheduled_start     TIMESTAMPTZ NOT NULL,
    scheduled_end       TIMESTAMPTZ NOT NULL,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    -- Setup and changeover
    setup_minutes       INT DEFAULT 0,
    changeover_minutes  INT DEFAULT 0,
    curing_delay_minutes INT DEFAULT 0,
    -- Status
    status              VARCHAR(20) DEFAULT 'scheduled' CHECK (status IN (
        'scheduled', 'in_progress', 'complete', 'delayed', 'cancelled'
    )),
    -- Scheduling metadata
    shift_number        INT DEFAULT 1,
    priority            INT DEFAULT 0,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Prevent double-booking
    EXCLUDE USING gist (
        machine_id WITH =,
        tstzrange(scheduled_start, scheduled_end) WITH &&
    ) WHERE (status NOT IN ('cancelled'))
);

CREATE INDEX idx_schedule_org ON schedule_entries(organisation_id);
CREATE INDEX idx_schedule_machine ON schedule_entries(machine_id, scheduled_start);
CREATE INDEX idx_schedule_job ON schedule_entries(job_id);
CREATE INDEX idx_schedule_date ON schedule_entries(scheduled_start, scheduled_end);

-- Machine downtime / maintenance blocks
CREATE TABLE machine_downtime (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    machine_id          UUID NOT NULL REFERENCES machines(id),
    reason              VARCHAR(50) NOT NULL CHECK (reason IN (
        'maintenance', 'breakdown', 'calibration', 'holiday', 'other'
    )),
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ NOT NULL,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_machine_downtime ON machine_downtime(machine_id, start_time);
```

### Module 8: Proofing and File Management

```sql
CREATE TABLE artwork_files (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id              UUID NOT NULL REFERENCES jobs(id),
    job_item_id         UUID REFERENCES job_items(id),
    uploaded_by_user_id UUID REFERENCES users(id),
    file_name           VARCHAR(500) NOT NULL,
    file_type           VARCHAR(20) CHECK (file_type IN (
        'pdf', 'ai', 'eps', 'psd', 'tiff', 'jpg', 'png', 'svg', 'other'
    )),
    file_size_bytes     BIGINT,
    storage_path        VARCHAR(1000) NOT NULL,
    storage_provider    VARCHAR(20) DEFAULT 's3',
    -- Prepress metadata
    width_mm            NUMERIC(10,2),
    height_mm           NUMERIC(10,2),
    resolution_dpi      INT,
    colour_space        VARCHAR(20),
    has_bleed           BOOLEAN,
    pdf_x_compliant     BOOLEAN,
    icc_profile         VARCHAR(255),
    -- Preflight results
    preflight_status    VARCHAR(20) CHECK (preflight_status IN (
        'pending', 'passed', 'warnings', 'failed'
    )),
    preflight_report    TEXT,
    version             INT NOT NULL DEFAULT 1,
    is_current          BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_artwork_files_job ON artwork_files(job_id);

CREATE TABLE proofs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id              UUID NOT NULL REFERENCES jobs(id),
    job_item_id         UUID REFERENCES job_items(id),
    artwork_file_id     UUID REFERENCES artwork_files(id),
    version             INT NOT NULL DEFAULT 1,
    status              VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'sent', 'viewed', 'approved', 'rejected', 'revision_requested'
    )),
    sent_to_email       VARCHAR(255),
    sent_at             TIMESTAMPTZ,
    viewed_at           TIMESTAMPTZ,
    responded_at        TIMESTAMPTZ,
    approved_by         VARCHAR(255),
    rejection_reason    TEXT,
    notes               TEXT,
    -- Chase tracking
    chase_count         INT DEFAULT 0,
    last_chased_at      TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_proofs_job ON proofs(job_id);
CREATE INDEX idx_proofs_status ON proofs(status);

CREATE TABLE proof_annotations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    proof_id            UUID NOT NULL REFERENCES proofs(id) ON DELETE CASCADE,
    annotated_by        VARCHAR(255),
    annotation_type     VARCHAR(20) CHECK (annotation_type IN (
        'comment', 'correction', 'approval', 'rejection'
    )),
    page_number         INT,
    x_position          NUMERIC(8,2),
    y_position          NUMERIC(8,2),
    content             TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Module 9: Delivery and Installation

```sql
CREATE TABLE deliveries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    delivery_date       DATE NOT NULL,
    delivery_type       VARCHAR(30) NOT NULL CHECK (delivery_type IN (
        'delivery', 'installation', 'pickup', 'courier', 'post'
    )),
    status              VARCHAR(20) DEFAULT 'planned' CHECK (status IN (
        'planned', 'dispatched', 'in_transit', 'delivered',
        'partially_delivered', 'failed', 'cancelled'
    )),
    -- Vehicle and crew
    vehicle_id          UUID,
    driver_user_id      UUID REFERENCES users(id),
    crew_notes          TEXT,
    -- Address
    delivery_address_line_1 VARCHAR(255),
    delivery_address_line_2 VARCHAR(255),
    delivery_city       VARCHAR(100),
    delivery_postal_code VARCHAR(20),
    delivery_instructions TEXT,
    -- Proof of delivery
    pod_signature       TEXT,
    pod_photo_urls      TEXT[],
    pod_received_by     VARCHAR(255),
    pod_timestamp       TIMESTAMPTZ,
    -- Installation specific
    installation_days   INT DEFAULT 1,
    subcontractor_id    UUID REFERENCES suppliers(id),
    installation_notes  TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_deliveries_org ON deliveries(organisation_id, delivery_date);
CREATE INDEX idx_deliveries_status ON deliveries(organisation_id, status);

-- Jobs included in a delivery
CREATE TABLE delivery_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    delivery_id         UUID NOT NULL REFERENCES deliveries(id) ON DELETE CASCADE,
    job_id              UUID NOT NULL REFERENCES jobs(id),
    job_item_id         UUID REFERENCES job_items(id),
    quantity            INT DEFAULT 1,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_delivery_items ON delivery_items(delivery_id);
CREATE INDEX idx_delivery_items_job ON delivery_items(job_id);

-- Vehicles
CREATE TABLE vehicles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(100) NOT NULL,
    registration        VARCHAR(20),
    vehicle_type        VARCHAR(50),
    max_load_kg         NUMERIC(8,2),
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE deliveries ADD CONSTRAINT fk_deliveries_vehicle
    FOREIGN KEY (vehicle_id) REFERENCES vehicles(id);
```

### Module 10: Invoicing and Accounting

```sql
CREATE TABLE invoices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    job_id              UUID REFERENCES jobs(id),
    invoice_number      VARCHAR(50) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'sent', 'viewed', 'partially_paid', 'paid',
        'overdue', 'void', 'credited'
    )),
    -- Dates
    issue_date          DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date            DATE,
    paid_date           DATE,
    -- Amounts
    subtotal            NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax_rate            NUMERIC(5,4) DEFAULT 0.10,
    tax_amount          NUMERIC(12,2) DEFAULT 0,
    total               NUMERIC(12,2) NOT NULL DEFAULT 0,
    amount_paid         NUMERIC(12,2) DEFAULT 0,
    balance_due         NUMERIC(12,2) DEFAULT 0,
    -- External sync
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
    job_item_id         UUID REFERENCES job_items(id),
    line_order          INT DEFAULT 0,
    description         VARCHAR(500) NOT NULL,
    quantity            NUMERIC(12,2) DEFAULT 1,
    unit_price          NUMERIC(12,4) NOT NULL,
    total               NUMERIC(12,2) NOT NULL,
    tax_code            VARCHAR(20),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_invoice_lines ON invoice_lines(invoice_id);

CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id          UUID NOT NULL REFERENCES invoices(id),
    amount              NUMERIC(12,2) NOT NULL,
    payment_method      VARCHAR(30) CHECK (payment_method IN (
        'bank_transfer', 'credit_card', 'cash', 'cheque', 'paypal', 'other'
    )),
    payment_reference   VARCHAR(255),
    payment_date        DATE NOT NULL,
    quickbooks_id       VARCHAR(100),
    xero_id             VARCHAR(100),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments ON payments(invoice_id);
```

### Module 11: Purchase Orders

```sql
CREATE TABLE purchase_orders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    supplier_id         UUID NOT NULL REFERENCES suppliers(id),
    po_number           VARCHAR(50) NOT NULL,
    status              VARCHAR(20) DEFAULT 'draft' CHECK (status IN (
        'draft', 'sent', 'acknowledged', 'partially_received',
        'received', 'cancelled'
    )),
    issue_date          DATE NOT NULL DEFAULT CURRENT_DATE,
    expected_date       DATE,
    subtotal            NUMERIC(12,2) DEFAULT 0,
    tax_amount          NUMERIC(12,2) DEFAULT 0,
    total               NUMERIC(12,2) DEFAULT 0,
    notes               TEXT,
    -- Link to triggering job (if applicable)
    triggered_by_job_id UUID REFERENCES jobs(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_po_number ON purchase_orders(organisation_id, po_number);

CREATE TABLE purchase_order_lines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id   UUID NOT NULL REFERENCES purchase_orders(id) ON DELETE CASCADE,
    material_id         UUID NOT NULL REFERENCES materials(id),
    quantity_ordered    NUMERIC(12,2) NOT NULL,
    quantity_received   NUMERIC(12,2) DEFAULT 0,
    unit_cost           NUMERIC(12,4) NOT NULL,
    total_cost          NUMERIC(12,2) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_po_lines ON purchase_order_lines(purchase_order_id);
```

### Module 12: Audit and Activity Log

```sql
CREATE TABLE activity_log (
    id                  BIGSERIAL PRIMARY KEY,
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID REFERENCES users(id),
    entity_type         VARCHAR(50) NOT NULL,
    entity_id           UUID NOT NULL,
    action              VARCHAR(30) NOT NULL CHECK (action IN (
        'created', 'updated', 'deleted', 'status_changed',
        'approved', 'rejected', 'sent', 'scanned', 'synced'
    )),
    old_values          JSONB,
    new_values          JSONB,
    ip_address          INET,
    user_agent          VARCHAR(500),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_activity_log_org ON activity_log(organisation_id, created_at DESC);
CREATE INDEX idx_activity_log_entity ON activity_log(entity_type, entity_id);
```

---

## Entity Relationship Summary

```
organisations 1──* users
organisations 1──* customers
organisations 1──* materials
organisations 1──* machines
organisations 1──* suppliers
organisations 1──* estimates
organisations 1──* jobs
organisations 1──* deliveries
organisations 1──* invoices

customers     1──* estimates
customers     1──* jobs
customers     1──* invoices
customers     1──* customer_contacts

estimates     1──* estimate_items
estimate_items 1──* estimate_cost_lines
estimates     1──1 jobs (conversion)

jobs          1──* job_items
job_items     1──* production_steps
production_steps 1──* production_scans
jobs          1──* job_material_usage
jobs          1──* job_time_entries
jobs          1──* artwork_files
jobs          1──* proofs

machines      1──* schedule_entries
machines      1──* machine_downtime
machines      *──* materials (via machine_material_capabilities)

jobs          1──* delivery_items
deliveries    1──* delivery_items

jobs          1──* invoice_lines (via invoices)
invoices      1──* invoice_lines
invoices      1──* payments

suppliers     1──* purchase_orders
purchase_orders 1──* purchase_order_lines
```

---

## Pros and Cons

### Pros

1. **Referential integrity everywhere.** Foreign keys with cascading rules prevent orphaned records across the entire estimation-to-invoice pipeline. A job cannot exist without a customer; a production step cannot exist without a job item.

2. **Strong consistency for financial data.** Invoicing, payments, and cost tracking rely on ACID transactions. Partial payment reconciliation, margin analysis, and accounting sync all benefit from the transactional guarantees of PostgreSQL.

3. **Mature query ecosystem.** Complex reporting queries (estimate vs. actual cost per job type, machine utilisation over time, customer profitability) are straightforward with standard SQL joins. No specialised query language required.

4. **PostgreSQL exclusion constraints for scheduling.** The `EXCLUDE USING gist` constraint on `schedule_entries` prevents double-booking a machine at the database level, which is critical for production scheduling integrity.

5. **Multi-tenant via organisation_id.** Row-level security (RLS) can be layered on top for true tenant isolation in a SaaS deployment without schema-per-tenant complexity.

6. **Well-understood operational model.** PostgreSQL backup, replication, monitoring, and scaling patterns are deeply documented. Point-in-time recovery protects against data loss.

7. **Direct mapping to CIP4 JDF/XJDF entities.** The machines, materials, production_steps, and job_items tables map cleanly to JDF Resource, Process, and Intent nodes, simplifying import/export.

### Cons

1. **Schema rigidity.** Adding a new production step type, material attribute, or job field requires a migration. Print shops with unusual workflows (e.g., LED channel letter fabrication, CNC cutting of acrylic signage panels) may hit schema limitations quickly.

2. **Wide-format estimation complexity.** Panel-count, linear-metre, and square-metre pricing models for signage do not fit neatly into a single `estimate_items` table structure. The `pricing_method` column helps but may need additional pricing tables as complexity grows.

3. **Gantt scheduling at scale.** A busy shop with 30 machines and 500 active jobs generates thousands of schedule_entries. Complex rescheduling queries (cascade a delay across downstream dependent steps) require careful indexing and may need materialised views.

4. **Join-heavy reporting.** A single job margin report must join jobs, job_items, production_steps, job_material_usage, job_time_entries, estimate_items, and estimate_cost_lines. Query performance needs attention as data volumes grow.

5. **Limited flexibility for AI features.** Storing ML model outputs (estimation suggestions, anomaly scores, demand forecasts) requires either additional columns or separate tables, making the schema grow unpredictably.

6. **No native event history.** The `activity_log` table captures changes but is not a true event store. Replaying state transitions (e.g., auditing every status change a job went through) requires careful log querying rather than native event replay.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ with btree_gist extension (for exclusion constraints) |
| Connection pooling | PgBouncer or Supabase Pooler |
| Migrations | Prisma Migrate or golang-migrate |
| ORM / query builder | Prisma (TypeScript), SQLAlchemy (Python), or Drizzle ORM |
| Full-text search | PostgreSQL tsvector + GIN index on job titles and descriptions |
| File storage | S3-compatible (AWS S3 / MinIO for self-hosted) with signed URLs |
| Caching | Redis for session data and schedule view caching |
| API layer | REST with OpenAPI 3.1; consider GraphQL for the customer portal |
| Real-time updates | WebSocket (via Socket.io or Supabase Realtime) for Gantt schedule changes |

---

## Migration and Scaling Considerations

### Initial Deployment
- Start with a single PostgreSQL instance (db.r6g.xlarge or equivalent) which can handle most small-to-medium print shops (< 10,000 jobs/year).
- Use connection pooling from day one to support WebSocket connections for real-time schedule updates.
- Enable WAL archiving for point-in-time recovery.

### Growth Phase (10,000-100,000 jobs/year)
- Add read replicas for reporting queries. Direct Gantt view reads and dashboard queries to replicas.
- Partition `production_scans` and `activity_log` by month (range partitioning on `created_at`) as these are high-volume write tables.
- Create materialised views for common reports (monthly revenue, machine utilisation, estimator accuracy).
- Add connection pooling capacity (PgBouncer with transaction-mode pooling).

### Scale Phase (100,000+ jobs/year or multi-tenant SaaS)
- Implement Row-Level Security (RLS) for tenant isolation.
- Consider Citus extension for horizontal sharding by `organisation_id`.
- Archive completed jobs older than the retention period to a data warehouse (e.g., BigQuery, Snowflake) for historical reporting.
- Move full-text search to Elasticsearch or Meilisearch if PostgreSQL tsvector becomes a bottleneck.
- Evaluate partitioning `schedule_entries` by date range to keep active schedule queries fast.

### Data Retention
- Active data: Current year + 2 years of completed jobs in the primary database.
- Archive: Older data exported to Parquet files in S3 or a columnar data warehouse.
- GDPR compliance: Customer data deletion must cascade through all related tables (jobs, artwork_files, proofs, invoices).
