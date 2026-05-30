# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Fleet Management Platform · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational design where every real-world concept — vehicle, driver, trip, work order, fuel transaction, geofence — receives its own dedicated table with strict foreign key relationships. Reference data (vehicle makes, fuel types, jurisdiction codes) is stored in lookup tables aligned to industry standards (ISO 3166 for jurisdictions, SAE J1939 SPN/FMI for fault codes, FMCSA status codes for HOS). The schema prioritises data integrity, auditability, and query flexibility at the cost of a larger table count and more complex joins.

Real-world systems that follow this pattern include Fleetio's maintenance-first architecture (separate entities for work orders, service entries, service tasks, parts, and line items) and Geotab's object model (Device, Driver, Trip, LogRecord, StatusData, FaultData as distinct queryable entities). Enterprise CMMS platforms like SAP PM and IBM Maximo also use deeply normalised schemas with hundreds of tables.

This approach is best when the platform must support complex cross-entity reporting (e.g., "total maintenance cost per vehicle per jurisdiction per quarter"), regulatory compliance queries that span multiple entities (HOS logs joined with driver records joined with carrier data), and when data integrity is more important than raw write throughput for telemetry ingestion.

**Best for:** Teams building a compliance-heavy, report-rich fleet platform where data integrity and query flexibility outweigh telemetry ingestion speed.

**Trade-offs:**
- (+) Maximum referential integrity — broken references are impossible
- (+) Standard SQL joins enable any ad-hoc query without denormalisation
- (+) Straightforward mapping to REST API resources (one table ≈ one endpoint)
- (+) Well-understood by most development teams; excellent ORM support
- (-) High table count (~55-65 tables) increases schema complexity
- (-) Telemetry ingestion at scale (thousands of position reports/second) requires careful indexing and partitioning
- (-) Many-to-many junction tables add insert overhead
- (-) Schema migrations required for every new field or entity type

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1/3166-2 | `jurisdiction` lookup table uses ISO codes for countries and subdivisions; IFTA reporting joins on these codes |
| SAE J1939 (SPN/FMI) | `fault_code` table references J1939 Suspect Parameter Numbers and Failure Mode Identifiers for heavy-duty vehicle diagnostics |
| SAE J1979 / ISO 15031 (OBD-II) | `diagnostic_trouble_code` table stores OBD-II PIDs and DTCs with standard code prefixes (P, C, B, U) |
| FMCSA 49 CFR Part 395 | `hos_log` and `eld_event` tables match FMCSA ELD data element requirements (date, time, location, engine hours, vehicle miles, duty status) |
| IFTA | `ifta_trip_segment` table captures per-jurisdiction mileage and fuel for quarterly IFTA reporting |
| GeoJSON / RFC 7946 | Geofence boundaries stored as GeoJSON-compatible polygons via PostGIS `GEOGRAPHY(POLYGON, 4326)` |
| ISO 39001 | Safety event tables support Road Traffic Safety Management System reporting requirements |
| OpenAPI 3.x | Schema maps directly to REST resources; one table per API endpoint |
| OAuth 2.0 / OIDC | `user`, `role`, `permission` tables support standard RBAC with tenant scoping |

---

## Core Platform Tables

### Organisations and Tenancy

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    parent_org_id   UUID REFERENCES organisation(id),  -- sub-fleet hierarchy
    dot_number      VARCHAR(20),          -- US DOT number for FMCSA compliance
    mc_number       VARCHAR(20),          -- Motor Carrier number
    ifta_account    VARCHAR(30),          -- IFTA licence account number
    ein             VARCHAR(20),          -- Employer Identification Number
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) REFERENCES jurisdiction(country_code),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisation_parent ON organisation(parent_org_id);
CREATE INDEX idx_organisation_slug ON organisation(slug);
```

```sql
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    phone           VARCHAR(30),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_user_org ON app_user(organisation_id);
CREATE INDEX idx_user_email ON app_user(email);
```

```sql
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(100) NOT NULL,  -- e.g., 'fleet_manager', 'driver', 'mechanic', 'admin'
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false,  -- system-defined vs custom
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, name)
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource        VARCHAR(100) NOT NULL,  -- e.g., 'vehicle', 'work_order', 'driver'
    action          VARCHAR(50) NOT NULL,   -- e.g., 'read', 'write', 'delete', 'export'
    description     TEXT,
    UNIQUE(resource, action)
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permission(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    fleet_scope_id  UUID REFERENCES fleet(id),  -- optional: restrict role to specific fleet
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id)
);
```

### Fleet and Vehicle Management

```sql
CREATE TABLE fleet (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fleet_org ON fleet(organisation_id);
```

```sql
CREATE TABLE vehicle_make (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE  -- e.g., 'Freightliner', 'Peterbilt', 'Ford'
);

CREATE TABLE vehicle_model (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    make_id         UUID NOT NULL REFERENCES vehicle_make(id),
    name            VARCHAR(100) NOT NULL,  -- e.g., 'Cascadia', '579', 'F-150'
    UNIQUE(make_id, name)
);
```

```sql
CREATE TABLE vehicle (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    fleet_id            UUID REFERENCES fleet(id),
    name                VARCHAR(100) NOT NULL,     -- unit number / display name
    vin                 VARCHAR(17),               -- Vehicle Identification Number
    licence_plate       VARCHAR(20),
    licence_state       VARCHAR(10),
    vehicle_model_id    UUID REFERENCES vehicle_model(id),
    year                SMALLINT,
    colour              VARCHAR(30),
    vehicle_type        VARCHAR(50) NOT NULL,      -- 'truck', 'van', 'car', 'trailer', 'equipment'
    fuel_type           VARCHAR(30) NOT NULL DEFAULT 'diesel',  -- 'diesel', 'gasoline', 'electric', 'hybrid', 'cng'
    tank_capacity_litres NUMERIC(8,2),
    odometer_km         NUMERIC(12,2),
    engine_hours        NUMERIC(12,2),
    status              VARCHAR(30) NOT NULL DEFAULT 'active',  -- 'active', 'inactive', 'in_maintenance', 'retired'
    acquisition_date    DATE,
    acquisition_cost    NUMERIC(12,2),
    disposal_date       DATE,
    disposal_value      NUMERIC(12,2),
    insurance_policy    VARCHAR(100),
    insurance_expiry    DATE,
    registration_expiry DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vehicle_org ON vehicle(organisation_id);
CREATE INDEX idx_vehicle_fleet ON vehicle(fleet_id);
CREATE INDEX idx_vehicle_vin ON vehicle(vin);
CREATE INDEX idx_vehicle_status ON vehicle(status);
```

```sql
CREATE TABLE telematics_device (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID REFERENCES vehicle(id),
    serial_number       VARCHAR(100) NOT NULL UNIQUE,
    device_type         VARCHAR(50) NOT NULL,    -- 'obd2', 'j1939', 'gps_tracker', 'dashcam', 'eld'
    manufacturer        VARCHAR(100),
    model               VARCHAR(100),
    firmware_version    VARCHAR(50),
    imei                VARCHAR(20),
    sim_iccid           VARCHAR(25),
    installed_at        TIMESTAMPTZ,
    status              VARCHAR(30) NOT NULL DEFAULT 'active',
    last_communication  TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_device_vehicle ON telematics_device(vehicle_id);
CREATE INDEX idx_device_serial ON telematics_device(serial_number);
```

### Driver Management

```sql
CREATE TABLE driver (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID REFERENCES app_user(id),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    employee_number     VARCHAR(50),
    licence_number      VARCHAR(50),
    licence_state       VARCHAR(10),
    licence_class       VARCHAR(20),
    licence_expiry      DATE,
    medical_card_expiry DATE,
    hazmat_endorsed     BOOLEAN NOT NULL DEFAULT false,
    tanker_endorsed     BOOLEAN NOT NULL DEFAULT false,
    hire_date           DATE,
    termination_date    DATE,
    status              VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_driver_org ON driver(organisation_id);
CREATE INDEX idx_driver_user ON driver(user_id);
CREATE INDEX idx_driver_status ON driver(status);
```

```sql
CREATE TABLE driver_vehicle_assignment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    driver_id           UUID NOT NULL REFERENCES driver(id),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    assigned_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    unassigned_at       TIMESTAMPTZ,
    is_primary          BOOLEAN NOT NULL DEFAULT false
);

CREATE INDEX idx_dva_driver ON driver_vehicle_assignment(driver_id);
CREATE INDEX idx_dva_vehicle ON driver_vehicle_assignment(vehicle_id);
CREATE INDEX idx_dva_active ON driver_vehicle_assignment(driver_id) WHERE unassigned_at IS NULL;
```

### GPS Tracking and Trips

```sql
CREATE TABLE vehicle_position (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    device_id           UUID REFERENCES telematics_device(id),
    recorded_at         TIMESTAMPTZ NOT NULL,
    received_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    latitude            NUMERIC(10,7) NOT NULL,
    longitude           NUMERIC(10,7) NOT NULL,
    altitude_m          NUMERIC(8,2),
    speed_kmh           NUMERIC(6,2),
    heading             NUMERIC(5,2),      -- degrees 0-360
    odometer_km         NUMERIC(12,2),
    engine_hours        NUMERIC(12,2),
    ignition_on         BOOLEAN,
    satellites          SMALLINT,
    hdop                NUMERIC(4,2),      -- horizontal dilution of precision
    location            GEOGRAPHY(POINT, 4326) -- PostGIS point for spatial queries
) PARTITION BY RANGE (recorded_at);

-- Create monthly partitions
-- CREATE TABLE vehicle_position_2026_01 PARTITION OF vehicle_position
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE INDEX idx_position_vehicle_time ON vehicle_position(vehicle_id, recorded_at DESC);
CREATE INDEX idx_position_location ON vehicle_position USING GIST(location);
CREATE INDEX idx_position_recorded ON vehicle_position(recorded_at DESC);
```

```sql
CREATE TABLE trip (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    driver_id           UUID REFERENCES driver(id),
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ,
    start_latitude      NUMERIC(10,7),
    start_longitude     NUMERIC(10,7),
    end_latitude        NUMERIC(10,7),
    end_longitude       NUMERIC(10,7),
    start_address       TEXT,
    end_address         TEXT,
    distance_km         NUMERIC(10,2),
    duration_seconds    INTEGER,
    max_speed_kmh       NUMERIC(6,2),
    avg_speed_kmh       NUMERIC(6,2),
    idle_duration_sec   INTEGER,
    fuel_consumed_l     NUMERIC(8,2),
    start_odometer_km   NUMERIC(12,2),
    end_odometer_km     NUMERIC(12,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trip_vehicle ON trip(vehicle_id, start_time DESC);
CREATE INDEX idx_trip_driver ON trip(driver_id, start_time DESC);
```

### Geofencing

```sql
CREATE TABLE geofence (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    geofence_type       VARCHAR(30) NOT NULL DEFAULT 'polygon',  -- 'polygon', 'circle'
    boundary            GEOGRAPHY(POLYGON, 4326),  -- PostGIS polygon for spatial queries
    centre_point        GEOGRAPHY(POINT, 4326),    -- for circle geofences
    radius_m            NUMERIC(10,2),             -- for circle geofences
    colour              VARCHAR(7),                -- hex colour for map display
    speed_limit_kmh     NUMERIC(6,2),              -- optional speed limit within zone
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_geofence_org ON geofence(organisation_id);
CREATE INDEX idx_geofence_boundary ON geofence USING GIST(boundary);
```

```sql
CREATE TABLE geofence_event (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    geofence_id         UUID NOT NULL REFERENCES geofence(id),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    driver_id           UUID REFERENCES driver(id),
    event_type          VARCHAR(20) NOT NULL,  -- 'enter', 'exit'
    occurred_at         TIMESTAMPTZ NOT NULL,
    latitude            NUMERIC(10,7) NOT NULL,
    longitude           NUMERIC(10,7) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_geofence_event_vehicle ON geofence_event(vehicle_id, occurred_at DESC);
CREATE INDEX idx_geofence_event_geofence ON geofence_event(geofence_id, occurred_at DESC);
```

### Driver Safety Events

```sql
CREATE TABLE safety_event (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    driver_id           UUID REFERENCES driver(id),
    device_id           UUID REFERENCES telematics_device(id),
    event_type          VARCHAR(50) NOT NULL,  -- 'harsh_braking', 'harsh_acceleration', 'harsh_cornering', 'speeding', 'distraction', 'drowsiness', 'collision'
    severity            VARCHAR(20) NOT NULL DEFAULT 'medium',  -- 'low', 'medium', 'high', 'critical'
    occurred_at         TIMESTAMPTZ NOT NULL,
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    speed_kmh           NUMERIC(6,2),
    speed_limit_kmh     NUMERIC(6,2),
    duration_seconds    INTEGER,
    g_force             NUMERIC(5,2),
    video_clip_url      TEXT,               -- dashcam clip if available
    reviewed            BOOLEAN NOT NULL DEFAULT false,
    reviewed_by         UUID REFERENCES app_user(id),
    reviewed_at         TIMESTAMPTZ,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_safety_event_vehicle ON safety_event(vehicle_id, occurred_at DESC);
CREATE INDEX idx_safety_event_driver ON safety_event(driver_id, occurred_at DESC);
CREATE INDEX idx_safety_event_type ON safety_event(event_type, occurred_at DESC);
CREATE INDEX idx_safety_event_unreviewed ON safety_event(occurred_at DESC) WHERE reviewed = false;
```

```sql
CREATE TABLE driver_safety_score (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    driver_id           UUID NOT NULL REFERENCES driver(id),
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    overall_score       NUMERIC(5,2) NOT NULL,  -- 0-100
    harsh_braking_score NUMERIC(5,2),
    speeding_score      NUMERIC(5,2),
    acceleration_score  NUMERIC(5,2),
    cornering_score     NUMERIC(5,2),
    idle_score          NUMERIC(5,2),
    total_events        INTEGER NOT NULL DEFAULT 0,
    total_distance_km   NUMERIC(10,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(driver_id, period_start, period_end)
);

CREATE INDEX idx_safety_score_driver ON driver_safety_score(driver_id, period_start DESC);
```

### Vehicle Diagnostics

```sql
CREATE TABLE diagnostic_trouble_code (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    device_id           UUID REFERENCES telematics_device(id),
    protocol            VARCHAR(10) NOT NULL,  -- 'obd2', 'j1939'
    code                VARCHAR(20) NOT NULL,  -- e.g., 'P0301' (OBD-II) or SPN/FMI combo
    spn                 INTEGER,               -- J1939 Suspect Parameter Number
    fmi                 INTEGER,               -- J1939 Failure Mode Identifier
    description         TEXT,
    severity            VARCHAR(20),           -- 'info', 'warning', 'critical'
    occurred_at         TIMESTAMPTZ NOT NULL,
    cleared_at          TIMESTAMPTZ,
    odometer_km         NUMERIC(12,2),
    engine_hours        NUMERIC(12,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dtc_vehicle ON diagnostic_trouble_code(vehicle_id, occurred_at DESC);
CREATE INDEX idx_dtc_code ON diagnostic_trouble_code(code);
CREATE INDEX idx_dtc_active ON diagnostic_trouble_code(vehicle_id) WHERE cleared_at IS NULL;
```

### Maintenance Management

```sql
CREATE TABLE service_task_template (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    name                VARCHAR(255) NOT NULL,  -- e.g., 'Oil Change', 'Brake Inspection', 'Tyre Rotation'
    description         TEXT,
    category            VARCHAR(100),           -- 'engine', 'brakes', 'tyres', 'electrical', 'body'
    estimated_hours     NUMERIC(5,2),
    estimated_cost      NUMERIC(10,2),
    interval_km         NUMERIC(10,2),          -- PM interval by distance
    interval_hours      NUMERIC(10,2),          -- PM interval by engine hours
    interval_days       INTEGER,                -- PM interval by calendar days
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_stt_org ON service_task_template(organisation_id);
```

```sql
CREATE TABLE work_order (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    work_order_number   VARCHAR(50) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'open',  -- 'open', 'in_progress', 'waiting_parts', 'completed', 'cancelled'
    priority            VARCHAR(20) NOT NULL DEFAULT 'medium',  -- 'low', 'medium', 'high', 'emergency'
    work_order_type     VARCHAR(30) NOT NULL,  -- 'preventive', 'corrective', 'inspection', 'recall'
    title               VARCHAR(255) NOT NULL,
    description         TEXT,
    assigned_to         UUID REFERENCES app_user(id),  -- mechanic
    vendor_id           UUID REFERENCES vendor(id),    -- external service provider
    scheduled_date      DATE,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    odometer_km         NUMERIC(12,2),
    engine_hours        NUMERIC(12,2),
    total_parts_cost    NUMERIC(10,2) DEFAULT 0,
    total_labour_cost   NUMERIC(10,2) DEFAULT 0,
    total_cost          NUMERIC(10,2) DEFAULT 0,
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_org ON work_order(organisation_id);
CREATE INDEX idx_wo_vehicle ON work_order(vehicle_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_number ON work_order(work_order_number);
```

```sql
CREATE TABLE work_order_line_item (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id       UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    service_task_id     UUID REFERENCES service_task_template(id),
    description         VARCHAR(255) NOT NULL,
    line_type           VARCHAR(20) NOT NULL,  -- 'labour', 'part', 'service'
    part_id             UUID REFERENCES part(id),
    quantity            NUMERIC(8,2) NOT NULL DEFAULT 1,
    unit_cost           NUMERIC(10,2) NOT NULL DEFAULT 0,
    total_cost          NUMERIC(10,2) NOT NULL DEFAULT 0,
    labour_hours        NUMERIC(5,2),
    completed           BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_woli_wo ON work_order_line_item(work_order_id);
```

```sql
CREATE TABLE vendor (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    name                VARCHAR(255) NOT NULL,
    contact_name        VARCHAR(100),
    email               VARCHAR(255),
    phone               VARCHAR(30),
    address             TEXT,
    specialities        TEXT[],        -- array of service specialities
    is_preferred        BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vendor_org ON vendor(organisation_id);
```

```sql
CREATE TABLE part (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    part_number         VARCHAR(100) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    category            VARCHAR(100),
    manufacturer        VARCHAR(100),
    unit_cost           NUMERIC(10,2),
    quantity_on_hand    INTEGER NOT NULL DEFAULT 0,
    reorder_point       INTEGER,
    reorder_quantity    INTEGER,
    location            VARCHAR(100),    -- bin/shelf location
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, part_number)
);

CREATE INDEX idx_part_org ON part(organisation_id);
```

### Vehicle Inspections (DVIR)

```sql
CREATE TABLE inspection_template (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    name                VARCHAR(255) NOT NULL,
    vehicle_type        VARCHAR(50),    -- applies to specific vehicle types
    inspection_type     VARCHAR(30) NOT NULL,  -- 'pre_trip', 'post_trip', 'dot', 'custom'
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inspection_template_item (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id         UUID NOT NULL REFERENCES inspection_template(id) ON DELETE CASCADE,
    category            VARCHAR(100) NOT NULL,  -- 'tyres', 'brakes', 'lights', 'engine', 'body'
    item_name           VARCHAR(255) NOT NULL,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    is_required         BOOLEAN NOT NULL DEFAULT true
);
```

```sql
CREATE TABLE inspection (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    driver_id           UUID NOT NULL REFERENCES driver(id),
    template_id         UUID REFERENCES inspection_template(id),
    inspection_type     VARCHAR(30) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'passed',  -- 'passed', 'failed', 'requires_attention'
    odometer_km         NUMERIC(12,2),
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    notes               TEXT,
    signature_url       TEXT,
    inspected_at        TIMESTAMPTZ NOT NULL,
    reviewed_by         UUID REFERENCES app_user(id),
    reviewed_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_vehicle ON inspection(vehicle_id, inspected_at DESC);
CREATE INDEX idx_inspection_driver ON inspection(driver_id, inspected_at DESC);
```

```sql
CREATE TABLE inspection_item_result (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id       UUID NOT NULL REFERENCES inspection(id) ON DELETE CASCADE,
    template_item_id    UUID REFERENCES inspection_template_item(id),
    category            VARCHAR(100) NOT NULL,
    item_name           VARCHAR(255) NOT NULL,
    result              VARCHAR(20) NOT NULL,  -- 'pass', 'fail', 'na'
    defect_severity     VARCHAR(20),           -- 'minor', 'major', 'critical'
    notes               TEXT,
    photo_url           TEXT
);

CREATE INDEX idx_iir_inspection ON inspection_item_result(inspection_id);
```

### Fuel Management

```sql
CREATE TABLE fuel_entry (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    driver_id           UUID REFERENCES driver(id),
    fuel_type           VARCHAR(30) NOT NULL,
    quantity_litres     NUMERIC(8,2) NOT NULL,
    unit_price          NUMERIC(8,4),
    total_cost          NUMERIC(10,2),
    currency            CHAR(3) NOT NULL DEFAULT 'USD',
    odometer_km         NUMERIC(12,2),
    is_full_tank        BOOLEAN,
    fuel_card_number    VARCHAR(50),
    merchant_name       VARCHAR(255),
    merchant_address    TEXT,
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    jurisdiction_code   VARCHAR(10),    -- ISO 3166-2 for IFTA
    transaction_date    TIMESTAMPTZ NOT NULL,
    receipt_url         TEXT,
    source              VARCHAR(30) NOT NULL DEFAULT 'manual',  -- 'manual', 'fuel_card', 'telematics'
    is_anomalous        BOOLEAN NOT NULL DEFAULT false,  -- AI-flagged anomaly
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fuel_vehicle ON fuel_entry(vehicle_id, transaction_date DESC);
CREATE INDEX idx_fuel_driver ON fuel_entry(driver_id, transaction_date DESC);
CREATE INDEX idx_fuel_jurisdiction ON fuel_entry(jurisdiction_code);
```

### ELD / HOS Compliance

```sql
CREATE TABLE hos_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    driver_id           UUID NOT NULL REFERENCES driver(id),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    log_date            DATE NOT NULL,
    duty_status         VARCHAR(20) NOT NULL,  -- 'off_duty', 'sleeper_berth', 'driving', 'on_duty_not_driving'
    status_start        TIMESTAMPTZ NOT NULL,
    status_end          TIMESTAMPTZ,
    duration_minutes    INTEGER,
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    location_desc       VARCHAR(255),    -- nearest city/landmark
    odometer_km         NUMERIC(12,2),
    engine_hours        NUMERIC(12,2),
    origin              VARCHAR(20) NOT NULL DEFAULT 'auto',  -- 'auto' (ELD), 'driver_edit', 'unidentified'
    annotation          TEXT,
    certified           BOOLEAN NOT NULL DEFAULT false,
    certified_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(driver_id, log_date, status_start)
);

CREATE INDEX idx_hos_driver ON hos_log(driver_id, log_date DESC);
CREATE INDEX idx_hos_vehicle ON hos_log(vehicle_id, log_date DESC);
```

```sql
CREATE TABLE hos_violation (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    driver_id           UUID NOT NULL REFERENCES driver(id),
    violation_type      VARCHAR(50) NOT NULL,  -- '11_hour_driving', '14_hour_window', '30_min_break', '60_70_hour'
    detected_at         TIMESTAMPTZ NOT NULL,
    violation_start     TIMESTAMPTZ NOT NULL,
    violation_end       TIMESTAMPTZ,
    duration_minutes    INTEGER,
    severity            VARCHAR(20) NOT NULL DEFAULT 'warning',
    acknowledged        BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by     UUID REFERENCES app_user(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hos_violation_driver ON hos_violation(driver_id, detected_at DESC);
```

### IFTA Reporting

```sql
CREATE TABLE jurisdiction (
    country_code        CHAR(2) NOT NULL,         -- ISO 3166-1 alpha-2
    subdivision_code    VARCHAR(6) NOT NULL,       -- ISO 3166-2 subdivision
    name                VARCHAR(100) NOT NULL,
    ifta_member         BOOLEAN NOT NULL DEFAULT false,
    fuel_tax_rate       NUMERIC(8,4),              -- tax rate per gallon/litre
    fuel_tax_unit       VARCHAR(10) DEFAULT 'gallon',
    PRIMARY KEY (country_code, subdivision_code)
);
```

```sql
CREATE TABLE ifta_trip_segment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trip_id             UUID NOT NULL REFERENCES trip(id),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    jurisdiction_code   VARCHAR(10) NOT NULL,  -- ISO 3166-2
    distance_km         NUMERIC(10,2) NOT NULL,
    fuel_consumed_l     NUMERIC(8,2),
    entered_at          TIMESTAMPTZ NOT NULL,
    exited_at           TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ifta_vehicle ON ifta_trip_segment(vehicle_id);
CREATE INDEX idx_ifta_jurisdiction ON ifta_trip_segment(jurisdiction_code);
CREATE INDEX idx_ifta_trip ON ifta_trip_segment(trip_id);
```

### Alerts and Notifications

```sql
CREATE TABLE alert_rule (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    name                VARCHAR(255) NOT NULL,
    alert_type          VARCHAR(50) NOT NULL,  -- 'speed', 'geofence', 'idle', 'maintenance_due', 'hos_warning', 'dtc', 'fuel_anomaly'
    conditions          JSONB NOT NULL,         -- rule parameters
    -- Example conditions: {"max_speed_kmh": 120, "duration_seconds": 30}
    severity            VARCHAR(20) NOT NULL DEFAULT 'medium',
    is_active           BOOLEAN NOT NULL DEFAULT true,
    notify_email        BOOLEAN NOT NULL DEFAULT false,
    notify_push         BOOLEAN NOT NULL DEFAULT true,
    notify_sms          BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_rule_org ON alert_rule(organisation_id);
```

```sql
CREATE TABLE alert (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_rule_id       UUID REFERENCES alert_rule(id),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    vehicle_id          UUID REFERENCES vehicle(id),
    driver_id           UUID REFERENCES driver(id),
    alert_type          VARCHAR(50) NOT NULL,
    severity            VARCHAR(20) NOT NULL,
    title               VARCHAR(255) NOT NULL,
    message             TEXT,
    occurred_at         TIMESTAMPTZ NOT NULL,
    acknowledged        BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by     UUID REFERENCES app_user(id),
    acknowledged_at     TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_org ON alert(organisation_id, occurred_at DESC);
CREATE INDEX idx_alert_vehicle ON alert(vehicle_id, occurred_at DESC);
CREATE INDEX idx_alert_unack ON alert(organisation_id, occurred_at DESC) WHERE acknowledged = false;
```

### EV Fleet Management

```sql
CREATE TABLE ev_status (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id              UUID NOT NULL REFERENCES vehicle(id),
    recorded_at             TIMESTAMPTZ NOT NULL,
    battery_soc_pct         NUMERIC(5,2),      -- state of charge percentage
    battery_soh_pct         NUMERIC(5,2),      -- state of health percentage
    estimated_range_km      NUMERIC(8,2),
    is_charging             BOOLEAN,
    charging_power_kw       NUMERIC(6,2),
    charging_station_id     UUID REFERENCES charging_station(id),
    energy_consumed_kwh     NUMERIC(8,2),
    energy_regenerated_kwh  NUMERIC(8,2),
    battery_temp_c          NUMERIC(5,1),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ev_status_vehicle ON ev_status(vehicle_id, recorded_at DESC);
```

```sql
CREATE TABLE charging_station (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID REFERENCES organisation(id),  -- NULL for public stations
    name                VARCHAR(255) NOT NULL,
    location            GEOGRAPHY(POINT, 4326),
    address             TEXT,
    charger_type        VARCHAR(30),      -- 'level1', 'level2', 'dc_fast'
    max_power_kw        NUMERIC(6,2),
    connector_type      VARCHAR(30),      -- 'j1772', 'ccs', 'chademo', 'tesla'
    status              VARCHAR(20) NOT NULL DEFAULT 'available',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_charging_station_location ON charging_station USING GIST(location);
```

### Audit Log

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    user_id             UUID,
    action              VARCHAR(50) NOT NULL,    -- 'create', 'update', 'delete', 'login', 'export'
    resource_type       VARCHAR(100) NOT NULL,   -- 'vehicle', 'work_order', 'hos_log', etc.
    resource_id         UUID,
    changes             JSONB,                   -- {"field": {"old": "...", "new": "..."}}
    ip_address          INET,
    user_agent          TEXT,
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_audit_org ON audit_log(organisation_id, occurred_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_user ON audit_log(user_id, occurred_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Auth | 6 | organisation, app_user, role, permission, role_permission, user_role |
| Fleet & Vehicles | 5 | fleet, vehicle_make, vehicle_model, vehicle, telematics_device |
| Drivers | 2 | driver, driver_vehicle_assignment |
| GPS & Trips | 3 | vehicle_position (partitioned), trip, geofence_event |
| Geofencing | 2 | geofence, geofence_event (counted above) |
| Safety | 2 | safety_event, driver_safety_score |
| Diagnostics | 1 | diagnostic_trouble_code |
| Maintenance | 5 | service_task_template, work_order, work_order_line_item, vendor, part |
| Inspections | 4 | inspection_template, inspection_template_item, inspection, inspection_item_result |
| Fuel | 1 | fuel_entry |
| ELD / HOS | 2 | hos_log, hos_violation |
| IFTA | 2 | jurisdiction, ifta_trip_segment |
| Alerts | 2 | alert_rule, alert |
| EV | 2 | ev_status, charging_station |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **~38** | Plus partitions for vehicle_position and audit_log |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe cross-service references, and prevents enumeration attacks on public APIs.

2. **PostGIS for spatial data** — geofence boundaries as `GEOGRAPHY(POLYGON, 4326)` and vehicle positions as `GEOGRAPHY(POINT, 4326)` enable native spatial queries (`ST_Contains`, `ST_DWithin`) without application-level geometry calculations.

3. **Range-partitioned high-volume tables** — `vehicle_position` and `audit_log` are partitioned by month on their timestamp column. This keeps individual partitions manageable and enables efficient time-range queries and partition-level retention policies.

4. **Separate vehicle and telematics_device** — a vehicle can have multiple devices (GPS tracker + ELD + dashcam), and devices can be moved between vehicles. This many-to-one relationship reflects hardware reality.

5. **Normalised vehicle make/model** — lookup tables for makes and models prevent inconsistent free-text entries ("Freightliner" vs "FREIGHTLINER" vs "Freight Liner") and enable consistent filtering and reporting.

6. **driver_vehicle_assignment as a temporal table** — tracks which driver was assigned to which vehicle over time, with `assigned_at`/`unassigned_at` timestamps. Essential for answering "who was driving this vehicle on date X?" for compliance and incident investigation.

7. **FMCSA-aligned HOS schema** — `hos_log` stores duty status changes with all fields required by 49 CFR Part 395: date, time, location, engine hours, vehicle miles, and duty status. The `origin` field distinguishes auto-recorded (ELD) from driver-edited entries.

8. **IFTA trip segments per jurisdiction** — `ifta_trip_segment` captures distance and fuel consumed per jurisdiction per trip, enabling automated IFTA quarterly report generation by summing across jurisdictions.

9. **Work order lifecycle modelling** — work orders follow Fleetio's proven pattern: open → in_progress → waiting_parts → completed, with line items for labour, parts, and services. Service task templates define PM intervals by distance, hours, or calendar days.

10. **Audit log with JSONB change capture** — the `changes` column stores a structured diff of what changed, enabling compliance reporting ("who changed this HOS log entry and what did they change?") without schema changes per audited field.
