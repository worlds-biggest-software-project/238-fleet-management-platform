# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Fleet Management Platform · Created: 2026-05-22

## Philosophy

This model keeps a stable relational core for the entities that are well-understood and universal across fleet deployments — vehicles, drivers, trips, work orders — while using PostgreSQL JSONB columns for everything that varies by customer, jurisdiction, vehicle type, or integration source. Instead of creating dozens of lookup tables and junction tables for every possible attribute, the schema stores structured-but-variable data in validated JSONB fields alongside indexed relational columns.

This pattern is common in SaaS platforms that serve diverse customers. Samsara's API returns vehicle objects with standard fields (id, name, vin, make, model) alongside flexible custom attributes. Geotab's MyGeotab platform stores engine data from 157 different OEMs, each producing different sensor parameters — a scenario that is impractical to model with fixed relational columns. FleetBase's open-source platform uses a similar approach: core logistics entities are relational while custom metadata flows through JSON fields. Multi-jurisdiction compliance (US FMCSA ELD rules differ from Canadian ELD rules; IFTA reporting fields vary by state) is another strong driver for JSONB flexibility.

This approach is best when: the platform must support diverse fleet types (trucking, last-mile delivery, municipal, construction) with different data requirements, rapid iteration and MVP development are priorities, jurisdiction-specific fields vary widely, and the team wants fewer total tables and faster schema evolution without migrations for every new field.

**Best for:** Rapid MVP development and multi-tenant platforms serving diverse fleet types where flexibility and speed of iteration outweigh strict normalisation.

**Trade-offs:**
- (+) Fewer tables (~25 vs ~38 in normalised) — simpler to understand and maintain
- (+) No schema migration needed for new fields — just add keys to JSONB columns
- (+) Naturally handles jurisdiction-specific and vehicle-type-specific attributes
- (+) GIN indexes on JSONB provide fast containment and key-existence queries
- (+) Easier integration with varied telematics devices that send different field sets
- (-) JSONB fields lack foreign key constraints — referential integrity is application-enforced
- (-) Complex JSONB queries can be slower than indexed relational columns for some patterns
- (-) Risk of "JSONB sprawl" — without discipline, JSONB columns become unstructured dumping grounds
- (-) Reporting and BI tools handle relational columns better than nested JSONB
- (-) Type safety for JSONB contents requires application-layer validation (JSON Schema)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1/3166-2 | `jurisdiction_code` relational columns use ISO codes; jurisdiction-specific fields stored in JSONB `compliance_data` |
| SAE J1939 / OBD-II | Telematics readings stored as JSONB with `protocol` discriminator; sensor parameters vary by protocol and stored flexibly |
| FMCSA 49 CFR Part 395 | Core HOS fields are relational; ELD-specific metadata (CMV power unit, trailer numbers, shipping docs) in JSONB `eld_data` |
| IFTA | Standard IFTA fields are relational; jurisdiction-specific tax rates and surcharges in JSONB |
| GeoJSON / RFC 7946 | Geofence boundaries stored as PostGIS GEOGRAPHY with GeoJSON properties in JSONB `properties` |
| FMS Standard | European truck data fields stored in JSONB alongside J1939/OBD-II data using a `source` discriminator |
| ISO 55000 | Asset lifecycle attributes (depreciation method, residual value, replacement criteria) in JSONB `lifecycle_data` |
| JSON Schema | All JSONB columns validated against published JSON Schemas at the application layer |

---

## Core Tables

### Organisation and Multi-Tenancy

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    parent_org_id   UUID REFERENCES organisation(id),
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "timezone": "America/Chicago",
    --   "distance_unit": "miles",
    --   "fuel_unit": "gallons",
    --   "currency": "USD",
    --   "dot_number": "1234567",
    --   "mc_number": "MC-987654",
    --   "ifta_account": "IL-123456",
    --   "ein": "12-3456789",
    --   "compliance_jurisdictions": ["US", "CA"],
    --   "features_enabled": ["eld", "ifta", "dashcam", "ev_management"]
    -- }
    address         JSONB,
    -- address example:
    -- {
    --   "line1": "123 Fleet Dr",
    --   "city": "Chicago",
    --   "state": "IL",
    --   "postal_code": "60601",
    --   "country": "US"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_slug ON organisation(slug);
CREATE INDEX idx_org_parent ON organisation(parent_org_id);
```

```sql
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "first_name": "Jane",
    --   "last_name": "Smith",
    --   "phone": "+1-555-0123",
    --   "avatar_url": "https://...",
    --   "notification_prefs": {"email": true, "push": true, "sms": false}
    -- }
    roles           TEXT[] NOT NULL DEFAULT '{}',  -- array of role names: ['fleet_manager', 'admin']
    fleet_scopes    UUID[],                        -- restrict to specific fleet IDs (NULL = all)
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_user_org ON app_user(organisation_id);
CREATE INDEX idx_user_email ON app_user(email);
CREATE INDEX idx_user_roles ON app_user USING GIN(roles);
```

### Vehicles

```sql
CREATE TABLE vehicle (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    fleet_id            UUID REFERENCES fleet(id),
    
    -- Core relational fields (queried/filtered/sorted frequently)
    name                VARCHAR(100) NOT NULL,
    vin                 VARCHAR(17),
    licence_plate       VARCHAR(20),
    vehicle_type        VARCHAR(50) NOT NULL,     -- 'truck', 'van', 'car', 'trailer', 'equipment', 'bus'
    fuel_type           VARCHAR(30) NOT NULL DEFAULT 'diesel',
    status              VARCHAR(30) NOT NULL DEFAULT 'active',
    year                SMALLINT,
    odometer_km         NUMERIC(12,2),
    engine_hours        NUMERIC(12,2),
    
    -- Flexible attributes that vary by vehicle type, OEM, and customer
    specs               JSONB NOT NULL DEFAULT '{}',
    -- specs example (truck):
    -- {
    --   "make": "Freightliner",
    --   "model": "Cascadia",
    --   "engine": "Detroit DD15",
    --   "transmission": "DT12 automated",
    --   "gvwr_kg": 36287,
    --   "axle_count": 3,
    --   "sleeper": true,
    --   "tank_capacity_litres": 450,
    --   "def_tank_litres": 75
    -- }
    -- specs example (electric van):
    -- {
    --   "make": "Rivian",
    --   "model": "EDV 700",
    --   "battery_capacity_kwh": 135,
    --   "max_range_km": 370,
    --   "charging_port": "ccs",
    --   "cargo_volume_m3": 19.8
    -- }
    
    lifecycle_data      JSONB NOT NULL DEFAULT '{}',
    -- lifecycle_data example:
    -- {
    --   "acquisition_date": "2024-03-15",
    --   "acquisition_cost": 185000,
    --   "depreciation_method": "straight_line",
    --   "useful_life_years": 7,
    --   "residual_value": 35000,
    --   "warranty_expiry": "2027-03-15",
    --   "insurance_policy": "POL-2024-8892",
    --   "insurance_expiry": "2027-01-01",
    --   "registration_expiry": "2027-06-30",
    --   "disposal_date": null,
    --   "disposal_value": null
    -- }
    
    compliance_data     JSONB NOT NULL DEFAULT '{}',
    -- compliance_data example:
    -- {
    --   "dot_inspection_due": "2026-09-01",
    --   "emissions_class": "EPA 2024",
    --   "carb_compliant": true,
    --   "smartway_enrolled": true,
    --   "ifta_qualified": true,
    --   "hazmat_placarded": false
    -- }
    
    custom_fields       JSONB NOT NULL DEFAULT '{}',
    -- customer-defined fields:
    -- {
    --   "department": "Northwest Region",
    --   "cost_centre": "CC-4420",
    --   "project_code": "PROJ-2026-A"
    -- }
    
    tags                TEXT[] NOT NULL DEFAULT '{}',
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vehicle_org ON vehicle(organisation_id);
CREATE INDEX idx_vehicle_fleet ON vehicle(fleet_id);
CREATE INDEX idx_vehicle_vin ON vehicle(vin);
CREATE INDEX idx_vehicle_status ON vehicle(status);
CREATE INDEX idx_vehicle_type ON vehicle(vehicle_type);
CREATE INDEX idx_vehicle_tags ON vehicle USING GIN(tags);
CREATE INDEX idx_vehicle_specs ON vehicle USING GIN(specs);
CREATE INDEX idx_vehicle_custom ON vehicle USING GIN(custom_fields);
```

```sql
CREATE TABLE fleet (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fleet_org ON fleet(organisation_id);
```

### Telematics Devices

```sql
CREATE TABLE telematics_device (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID REFERENCES vehicle(id),
    serial_number       VARCHAR(100) NOT NULL UNIQUE,
    device_type         VARCHAR(50) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'active',
    
    device_info         JSONB NOT NULL DEFAULT '{}',
    -- device_info example:
    -- {
    --   "manufacturer": "Geotab",
    --   "model": "GO9",
    --   "firmware": "v4.2.1",
    --   "imei": "351234567890123",
    --   "sim_iccid": "8901234567890123456",
    --   "protocols": ["obd2", "j1939"],
    --   "capabilities": ["gps", "engine_data", "accelerometer"]
    -- }
    
    last_communication  TIMESTAMPTZ,
    installed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_device_vehicle ON telematics_device(vehicle_id);
```

### Drivers

```sql
CREATE TABLE driver (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID REFERENCES app_user(id),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    
    -- Core relational fields
    status              VARCHAR(30) NOT NULL DEFAULT 'active',
    employee_number     VARCHAR(50),
    
    -- Flexible driver details (vary by jurisdiction and employer)
    licence_info        JSONB NOT NULL DEFAULT '{}',
    -- licence_info example:
    -- {
    --   "number": "D123-4567-8901",
    --   "state": "IL",
    --   "class": "A",
    --   "expiry": "2028-06-15",
    --   "endorsements": ["hazmat", "tanker", "doubles_triples"],
    --   "restrictions": [],
    --   "medical_card_expiry": "2027-12-01"
    -- }
    
    employment_info     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "hire_date": "2022-04-01",
    --   "department": "Long Haul",
    --   "home_terminal": "Chicago Hub",
    --   "pay_type": "per_mile",
    --   "pay_rate": 0.62
    -- }
    
    qualifications      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "drug_test_date": "2026-03-15",
    --   "drug_test_result": "negative",
    --   "background_check_date": "2022-03-20",
    --   "training_completed": ["smith_system", "hazmat_awareness", "defensive_driving"],
    --   "certifications": [
    --     {"name": "TWIC Card", "number": "TWC-123456", "expiry": "2028-01-15"}
    --   ]
    -- }
    
    custom_fields       JSONB NOT NULL DEFAULT '{}',
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_driver_org ON driver(organisation_id);
CREATE INDEX idx_driver_user ON driver(user_id);
CREATE INDEX idx_driver_status ON driver(status);
```

### GPS Tracking and Trips

```sql
CREATE TABLE vehicle_position (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    recorded_at         TIMESTAMPTZ NOT NULL,
    latitude            NUMERIC(10,7) NOT NULL,
    longitude           NUMERIC(10,7) NOT NULL,
    speed_kmh           NUMERIC(6,2),
    heading             NUMERIC(5,2),
    location            GEOGRAPHY(POINT, 4326),
    
    -- All other position data in JSONB — varies by device and protocol
    telemetry           JSONB NOT NULL DEFAULT '{}',
    -- telemetry example:
    -- {
    --   "altitude_m": 210.5,
    --   "odometer_km": 234567.8,
    --   "engine_hours": 12345.6,
    --   "ignition": true,
    --   "satellites": 12,
    --   "hdop": 0.8,
    --   "battery_voltage": 13.8,
    --   "fuel_level_pct": 72.5,
    --   "engine_rpm": 1450,
    --   "coolant_temp_c": 91,
    --   "intake_temp_c": 32,
    --   "barometric_kpa": 101.3
    -- }
    
    device_id           UUID REFERENCES telematics_device(id)
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_pos_vehicle_time ON vehicle_position(vehicle_id, recorded_at DESC);
CREATE INDEX idx_pos_location ON vehicle_position USING GIST(location);
CREATE INDEX idx_pos_telemetry ON vehicle_position USING GIN(telemetry);
```

```sql
CREATE TABLE trip (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    driver_id           UUID REFERENCES driver(id),
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ,
    
    -- Core metrics (relational for fast queries)
    distance_km         NUMERIC(10,2),
    duration_seconds    INTEGER,
    max_speed_kmh       NUMERIC(6,2),
    avg_speed_kmh       NUMERIC(6,2),
    fuel_consumed_l     NUMERIC(8,2),
    
    -- Detailed trip data in JSONB
    route_data          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "start_address": "123 Main St, Chicago, IL",
    --   "end_address": "456 Oak Ave, Indianapolis, IN",
    --   "start_coords": [41.8781, -87.6298],
    --   "end_coords": [39.7684, -86.1581],
    --   "start_odometer_km": 234000,
    --   "end_odometer_km": 234290,
    --   "idle_duration_sec": 1245,
    --   "stop_count": 3,
    --   "jurisdictions_crossed": ["IL", "IN"],
    --   "toll_charges": [
    --     {"plaza": "I-90 Skyway", "amount": 5.60, "jurisdiction": "IL"}
    --   ]
    -- }
    
    safety_summary      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "harsh_braking_count": 2,
    --   "harsh_accel_count": 0,
    --   "speeding_events": 1,
    --   "max_speed_over_limit_kmh": 12,
    --   "safety_score": 85.5
    -- }
    
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
    geofence_type       VARCHAR(30) NOT NULL DEFAULT 'polygon',
    boundary            GEOGRAPHY(POLYGON, 4326),
    centre_point        GEOGRAPHY(POINT, 4326),
    radius_m            NUMERIC(10,2),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    
    properties          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "colour": "#FF5722",
    --   "speed_limit_kmh": 30,
    --   "category": "depot",
    --   "address": "789 Warehouse Rd, Memphis, TN",
    --   "operating_hours": {"start": "06:00", "end": "22:00"},
    --   "contact_name": "Bob Johnson",
    --   "contact_phone": "+1-555-0456"
    -- }
    
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
    event_type          VARCHAR(20) NOT NULL,   -- 'enter', 'exit'
    occurred_at         TIMESTAMPTZ NOT NULL,
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    metadata            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gf_event_vehicle ON geofence_event(vehicle_id, occurred_at DESC);
```

### Safety Events

```sql
CREATE TABLE safety_event (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    driver_id           UUID REFERENCES driver(id),
    event_type          VARCHAR(50) NOT NULL,
    severity            VARCHAR(20) NOT NULL DEFAULT 'medium',
    occurred_at         TIMESTAMPTZ NOT NULL,
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    
    -- Event-specific details vary by type
    event_data          JSONB NOT NULL DEFAULT '{}',
    -- harsh_braking example:
    -- {
    --   "speed_kmh": 95,
    --   "deceleration_g": -0.65,
    --   "duration_seconds": 3,
    --   "road_type": "highway"
    -- }
    -- speeding example:
    -- {
    --   "speed_kmh": 132,
    --   "speed_limit_kmh": 105,
    --   "duration_seconds": 45,
    --   "distance_km": 1.6
    -- }
    -- distraction example:
    -- {
    --   "distraction_type": "phone_use",
    --   "confidence": 0.92,
    --   "video_clip_url": "https://..."
    -- }
    
    review_status       VARCHAR(20) NOT NULL DEFAULT 'pending',  -- 'pending', 'reviewed', 'dismissed'
    reviewed_by         UUID REFERENCES app_user(id),
    review_notes        TEXT,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_safety_vehicle ON safety_event(vehicle_id, occurred_at DESC);
CREATE INDEX idx_safety_driver ON safety_event(driver_id, occurred_at DESC);
CREATE INDEX idx_safety_type ON safety_event(event_type, occurred_at DESC);
CREATE INDEX idx_safety_pending ON safety_event(occurred_at DESC) WHERE review_status = 'pending';
```

### Maintenance

```sql
CREATE TABLE work_order (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    work_order_number   VARCHAR(50) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    priority            VARCHAR(20) NOT NULL DEFAULT 'medium',
    work_order_type     VARCHAR(30) NOT NULL,
    title               VARCHAR(255) NOT NULL,
    description         TEXT,
    
    -- Flexible line items stored as JSONB array instead of a separate table
    line_items          JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {
    --     "id": "uuid",
    --     "type": "labour",
    --     "description": "Brake pad replacement — front axle",
    --     "service_task": "Brake Inspection",
    --     "hours": 2.5,
    --     "rate": 95.00,
    --     "cost": 237.50,
    --     "mechanic_id": "uuid",
    --     "completed": true
    --   },
    --   {
    --     "id": "uuid",
    --     "type": "part",
    --     "description": "Front brake pads (set of 4)",
    --     "part_number": "BP-4420-HD",
    --     "quantity": 1,
    --     "unit_cost": 185.00,
    --     "cost": 185.00,
    --     "completed": true
    --   }
    -- ]
    
    scheduling          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "assigned_to": "uuid",
    --   "vendor_name": "Joe's Truck Service",
    --   "vendor_id": "uuid",
    --   "scheduled_date": "2026-05-25",
    --   "started_at": "2026-05-25T09:00:00Z",
    --   "completed_at": "2026-05-25T14:30:00Z",
    --   "bay_number": "3"
    -- }
    
    costs               JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "parts": 185.00,
    --   "labour": 237.50,
    --   "tax": 33.80,
    --   "total": 456.30,
    --   "currency": "USD"
    -- }
    
    vehicle_state       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "odometer_km": 145678,
    --   "engine_hours": 8234.5
    -- }
    
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
CREATE TABLE service_schedule (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    name                VARCHAR(255) NOT NULL,
    
    -- PM schedule rules in JSONB — supports varied trigger types
    triggers            JSONB NOT NULL,
    -- {
    --   "interval_km": 25000,
    --   "interval_hours": 500,
    --   "interval_days": 180,
    --   "trigger_mode": "any",
    --   "applies_to": {
    --     "vehicle_types": ["truck", "van"],
    --     "fuel_types": ["diesel"],
    --     "tags": ["heavy_duty"]
    --   }
    -- }
    
    task_template       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "category": "engine",
    --   "default_line_items": [
    --     {"type": "service", "description": "Oil change", "estimated_hours": 1.0, "estimated_cost": 150.00},
    --     {"type": "part", "description": "Oil filter", "part_number": "OF-1234", "estimated_cost": 25.00}
    --   ]
    -- }
    
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schedule_org ON service_schedule(organisation_id);
```

### Inspections (DVIR)

```sql
CREATE TABLE inspection (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    driver_id           UUID NOT NULL REFERENCES driver(id),
    inspection_type     VARCHAR(30) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'passed',
    inspected_at        TIMESTAMPTZ NOT NULL,
    
    -- Inspection results as a structured JSONB array
    items               JSONB NOT NULL,
    -- [
    --   {"category": "tyres", "item": "LF Tyre Condition", "result": "pass", "notes": null},
    --   {"category": "tyres", "item": "RF Tyre Condition", "result": "pass", "notes": null},
    --   {"category": "brakes", "item": "Brake Pedal", "result": "fail", "severity": "major",
    --    "notes": "Excessive pedal travel", "photo_url": "https://..."},
    --   {"category": "lights", "item": "Headlights", "result": "pass", "notes": null}
    -- ]
    
    location            JSONB NOT NULL DEFAULT '{}',
    -- {"latitude": 41.8781, "longitude": -87.6298, "address": "Chicago terminal"}
    
    odometer_km         NUMERIC(12,2),
    signature_url       TEXT,
    reviewed_by         UUID REFERENCES app_user(id),
    reviewed_at         TIMESTAMPTZ,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_vehicle ON inspection(vehicle_id, inspected_at DESC);
CREATE INDEX idx_inspection_driver ON inspection(driver_id, inspected_at DESC);
CREATE INDEX idx_inspection_failed ON inspection(vehicle_id) WHERE status = 'failed';
```

### Fuel Management

```sql
CREATE TABLE fuel_entry (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    driver_id           UUID REFERENCES driver(id),
    transaction_date    TIMESTAMPTZ NOT NULL,
    fuel_type           VARCHAR(30) NOT NULL,
    quantity_litres     NUMERIC(8,2) NOT NULL,
    total_cost          NUMERIC(10,2),
    currency            CHAR(3) NOT NULL DEFAULT 'USD',
    odometer_km         NUMERIC(12,2),
    jurisdiction_code   VARCHAR(10),       -- ISO 3166-2 for IFTA
    source              VARCHAR(30) NOT NULL DEFAULT 'manual',
    is_anomalous        BOOLEAN NOT NULL DEFAULT false,
    
    transaction_data    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "unit_price": 3.459,
    --   "is_full_tank": true,
    --   "fuel_card_number": "5678-xxxx-xxxx-1234",
    --   "fuel_card_provider": "comdata",
    --   "merchant_name": "Pilot Travel Center #342",
    --   "merchant_address": "I-65 Exit 234, Indianapolis, IN",
    --   "latitude": 39.7684,
    --   "longitude": -86.1581,
    --   "receipt_url": "https://...",
    --   "def_litres": 10.5,
    --   "def_cost": 8.40
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fuel_vehicle ON fuel_entry(vehicle_id, transaction_date DESC);
CREATE INDEX idx_fuel_jurisdiction ON fuel_entry(jurisdiction_code);
CREATE INDEX idx_fuel_anomalous ON fuel_entry(vehicle_id) WHERE is_anomalous = true;
```

### ELD / HOS Compliance

```sql
CREATE TABLE hos_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    driver_id           UUID NOT NULL REFERENCES driver(id),
    vehicle_id          UUID NOT NULL REFERENCES vehicle(id),
    log_date            DATE NOT NULL,
    duty_status         VARCHAR(20) NOT NULL,
    status_start        TIMESTAMPTZ NOT NULL,
    status_end          TIMESTAMPTZ,
    duration_minutes    INTEGER,
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    origin              VARCHAR(20) NOT NULL DEFAULT 'auto',
    certified           BOOLEAN NOT NULL DEFAULT false,
    
    -- ELD-specific metadata varies by device and jurisdiction
    eld_data            JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "location_description": "Chicago, IL (5mi S)",
    --   "odometer_km": 234567.8,
    --   "engine_hours": 12345.6,
    --   "cmv_power_unit": "TRUCK-042",
    --   "trailer_numbers": ["TRAIL-101", "TRAIL-102"],
    --   "shipping_doc": "BOL-2026-4521",
    --   "co_driver_id": "uuid",
    --   "malfunction_indicator": false,
    --   "data_diagnostic_indicator": false,
    --   "annotation": "Driver note here",
    --   "edit_history": [
    --     {
    --       "edited_at": "2026-05-21T14:30:00Z",
    --       "edited_by": "uuid",
    --       "original_status": "off_duty",
    --       "reason": "Incorrect status — was in sleeper berth"
    --     }
    --   ]
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(driver_id, log_date, status_start)
);

CREATE INDEX idx_hos_driver ON hos_log(driver_id, log_date DESC);
CREATE INDEX idx_hos_vehicle ON hos_log(vehicle_id, log_date DESC);
```

### Alerts

```sql
CREATE TABLE alert (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    vehicle_id          UUID REFERENCES vehicle(id),
    driver_id           UUID REFERENCES driver(id),
    alert_type          VARCHAR(50) NOT NULL,
    severity            VARCHAR(20) NOT NULL,
    title               VARCHAR(255) NOT NULL,
    message             TEXT,
    occurred_at         TIMESTAMPTZ NOT NULL,
    acknowledged        BOOLEAN NOT NULL DEFAULT false,
    
    context             JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "rule_id": "uuid",
    --   "trigger_value": 135,
    --   "threshold": 120,
    --   "unit": "kmh",
    --   "latitude": 41.8781,
    --   "longitude": -87.6298,
    --   "related_entity_type": "geofence",
    --   "related_entity_id": "uuid"
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_org ON alert(organisation_id, occurred_at DESC);
CREATE INDEX idx_alert_unack ON alert(organisation_id, occurred_at DESC) WHERE acknowledged = false;
```

### Audit Log

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    user_id             UUID,
    action              VARCHAR(50) NOT NULL,
    resource_type       VARCHAR(100) NOT NULL,
    resource_id         UUID,
    changes             JSONB,
    context             JSONB,       -- ip_address, user_agent, etc.
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_audit_org ON audit_log(organisation_id, occurred_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Example Queries

### Find all diesel trucks with CARB compliance issues

```sql
SELECT id, name, vin, specs->>'make' AS make, specs->>'model' AS model
FROM vehicle
WHERE organisation_id = '(org-uuid)'
  AND vehicle_type = 'truck'
  AND fuel_type = 'diesel'
  AND (compliance_data->>'carb_compliant')::boolean = false;
```

### Find vehicles with specific custom field values

```sql
SELECT id, name, custom_fields->>'department' AS department
FROM vehicle
WHERE organisation_id = '(org-uuid)'
  AND custom_fields @> '{"cost_centre": "CC-4420"}';
```

### Calculate work order costs across all line items (JSONB array)

```sql
SELECT
    wo.id,
    wo.work_order_number,
    wo.title,
    jsonb_array_length(wo.line_items) AS item_count,
    (SELECT sum((item->>'cost')::numeric) FROM jsonb_array_elements(wo.line_items) AS item) AS total_cost
FROM work_order wo
WHERE wo.vehicle_id = '(vehicle-uuid)'
  AND wo.status = 'completed'
ORDER BY wo.updated_at DESC;
```

### Query telemetry with varied sensor data

```sql
SELECT
    recorded_at,
    speed_kmh,
    telemetry->>'engine_rpm' AS rpm,
    telemetry->>'coolant_temp_c' AS coolant_temp,
    telemetry->>'fuel_level_pct' AS fuel_level
FROM vehicle_position
WHERE vehicle_id = '(vehicle-uuid)'
  AND recorded_at >= now() - INTERVAL '1 hour'
  AND telemetry ? 'engine_rpm'
ORDER BY recorded_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Auth | 2 | organisation, app_user (roles as array, no separate role/permission tables) |
| Fleet & Vehicles | 3 | fleet, vehicle, telematics_device |
| Drivers | 1 | driver (licence, employment, qualifications in JSONB) |
| GPS & Trips | 2 | vehicle_position (partitioned), trip |
| Geofencing | 2 | geofence, geofence_event |
| Safety | 1 | safety_event |
| Maintenance | 2 | work_order (line items in JSONB), service_schedule |
| Inspections | 1 | inspection (items in JSONB array) |
| Fuel | 1 | fuel_entry |
| ELD / HOS | 1 | hos_log (edit history in JSONB) |
| Alerts | 1 | alert |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **~18** | Significantly fewer tables than normalised; flexibility in JSONB |

---

## Key Design Decisions

1. **JSONB for variable-structure data, relational for query-critical fields** — fields that appear in WHERE, ORDER BY, GROUP BY, and JOIN clauses (vehicle_id, status, vehicle_type, fuel_type, occurred_at) are relational columns. Fields that are displayed, varied, or customer-specific go in JSONB. This ensures query performance while maintaining flexibility.

2. **GIN indexes on JSONB** — PostgreSQL GIN indexes support `@>` (containment), `?` (key existence), and `?&` (all keys exist) operators on JSONB columns. This enables fast searches like "all vehicles where specs contains make = Freightliner" without scanning the full table.

3. **Work order line items as JSONB array** — instead of a separate `work_order_line_item` table, line items are stored as a JSONB array within the work order. This simplifies the API (one GET returns the complete work order), reduces join complexity, and matches how the data is typically consumed. The trade-off is that per-part reporting requires JSONB unpacking (`jsonb_array_elements`).

4. **Inspection items as JSONB array** — same pattern as work orders. A vehicle inspection is a single document with an array of checked items. This matches the mobile app workflow (driver fills in one form) and avoids a many-row insert per inspection.

5. **HOS edit history in JSONB** — FMCSA requires preserving original HOS entries when edits are made. Rather than a separate edit history table, edits are appended to an `edit_history` array within the `eld_data` JSONB column. This keeps the complete edit chain with the log entry for compliance review.

6. **Roles as PostgreSQL array** — instead of role/permission junction tables, user roles are stored as a `TEXT[]` array on the user record. For most fleet management deployments, the role set is small and well-defined (admin, fleet_manager, driver, mechanic, dispatcher). Array containment queries (`roles @> ARRAY['admin']`) are fast with GIN indexes.

7. **Organisation settings in JSONB** — distance units, fuel units, currency, enabled features, and carrier numbers are stored in a single JSONB `settings` column. This avoids a settings table with one row per setting and makes it trivial to add new org-level configuration without migrations.

8. **Vehicle specs vary by type** — a diesel truck has GVWR, axle count, DEF tank capacity, and sleeper cab info. An electric van has battery capacity, charging port type, and cargo volume. A trailer has length, door type, and reefer unit info. JSONB `specs` accommodates all of these without null-heavy wide tables or entity-attribute-value patterns.

9. **JSON Schema validation at application layer** — each JSONB column has a published JSON Schema that the application validates on write. This provides type safety without database-level constraints, keeping the database schema simple while ensuring data quality in the application.

10. **Partition strategy for high-volume tables only** — only `vehicle_position` and `audit_log` are partitioned by time range. The hybrid approach keeps table count low while handling the two tables that grow fastest.
