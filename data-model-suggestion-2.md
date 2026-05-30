# Data Model Suggestion 2: Event-Sourced / Audit-First with Time-Series Telemetry

> Project: Fleet Management Platform · Created: 2026-05-22

## Philosophy

This model treats every state change in the fleet — a vehicle moving, a driver changing duty status, a work order progressing, a geofence being crossed — as an immutable event appended to an event store. The event store is the single source of truth. Materialised read models (CQRS pattern) are projected from the event stream to serve operational dashboards, reports, and API queries. Vehicle telemetry (positions, sensor readings, diagnostics) flows into TimescaleDB hypertables optimised for time-series ingestion and range queries.

This architecture is inspired by systems where full audit trails are non-negotiable: financial ledgers, healthcare records, and specifically FMCSA ELD compliance where every edit to a driver's HOS log must be recorded and the original entry preserved. Event sourcing naturally satisfies the FMCSA requirement that "the ELD records the driver's duty status and all changes, including the original and the edit, along with the driver's annotation." The CQRS read-side pattern is used by platforms like Microsoft's Azure IoT reference architecture for automotive telemetry and by EventStoreDB-based fleet tracking systems.

This approach is best when: regulatory audit trails are a primary requirement (FMCSA ELD, IFTA), the platform must answer temporal queries ("what was the fleet state at 3pm yesterday?"), telemetry ingestion rates are high (thousands of position reports per second across the fleet), and AI/ML analytics will process historical event streams for predictive maintenance and anomaly detection.

**Best for:** Compliance-first deployments where immutable audit trails, temporal queries, and high-throughput telemetry ingestion are paramount.

**Trade-offs:**
- (+) Complete, immutable audit trail — every state change is preserved forever
- (+) Temporal queries are natural — replay events to reconstruct state at any point in time
- (+) High-throughput write path — append-only event ingestion scales linearly
- (+) TimescaleDB hypertables handle vehicle telemetry at scale with compression and retention policies
- (+) Event streams are ideal input for ML pipelines and anomaly detection
- (-) Increased storage requirements — events are never deleted, only archived
- (-) Eventual consistency between write model (event store) and read models (projections)
- (-) More complex application code — event handlers, projectors, and snapshot management
- (-) Read model rebuilds can be slow for large event histories without snapshots
- (-) Steeper learning curve for teams unfamiliar with event sourcing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FMCSA 49 CFR Part 395 | Every HOS status change and edit is an immutable event with original + edit preserved, directly satisfying ELD audit requirements |
| IFTA | Jurisdiction-crossing events are first-class events; IFTA reports are projections from the trip_event stream |
| SAE J1939 / OBD-II | Sensor readings and fault codes flow as telemetry events into TimescaleDB hypertables with standard SPN/FMI identifiers |
| GeoJSON / RFC 7946 | Geofence definitions stored as GeoJSON; boundary crossing detected and emitted as geofence events |
| ISO 3166-1/3166-2 | Jurisdiction references in trip events use ISO codes |
| ISO 39001 | Safety event stream supports Road Traffic Safety Management reporting by aggregating safety events over time |
| OpenAPI 3.x | Read models expose REST API endpoints; event ingestion uses separate write API |
| OCSF | Event schema follows Open Cybersecurity Schema Framework patterns for structured, typed event logging |

---

## Architecture Overview

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│   Write API     │────▶│   Event Store    │────▶│   Event Projectors  │
│  (Commands)     │     │  (Append-only)   │     │                     │
└─────────────────┘     └──────────────────┘     └──────┬──────────────┘
                                                        │
                              ┌──────────────────────────┤
                              ▼                          ▼
                   ┌──────────────────┐      ┌──────────────────────┐
                   │   Read Models    │      │  TimescaleDB         │
                   │  (Projections)   │      │  (Telemetry Series)  │
                   └──────────────────┘      └──────────────────────┘
                              │                          │
                              ▼                          ▼
                   ┌──────────────────┐      ┌──────────────────────┐
                   │    Read API      │      │   Analytics / ML     │
                   │   (Queries)      │      │   Pipeline           │
                   └──────────────────┘      └──────────────────────┘
```

---

## Event Store

```sql
-- The central event store. ALL state changes are appended here.
CREATE TABLE event_store (
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id           UUID NOT NULL,          -- aggregate root ID (vehicle_id, driver_id, work_order_id, etc.)
    stream_type         VARCHAR(50) NOT NULL,    -- 'vehicle', 'driver', 'work_order', 'trip', 'fleet'
    event_type          VARCHAR(100) NOT NULL,   -- e.g., 'VehicleCreated', 'PositionRecorded', 'DutyStatusChanged'
    event_version       INTEGER NOT NULL,        -- monotonically increasing per stream
    occurred_at         TIMESTAMPTZ NOT NULL,    -- when the event happened in the real world
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when we recorded it
    actor_id            UUID,                    -- user or system that caused the event
    actor_type          VARCHAR(30),             -- 'user', 'driver', 'system', 'device', 'integration'
    correlation_id      UUID,                    -- links related events across aggregates
    causation_id        UUID,                    -- the event that caused this event
    payload             JSONB NOT NULL,          -- event-specific data
    metadata            JSONB,                   -- context: ip_address, device_info, etc.
    UNIQUE(stream_id, event_version)
) PARTITION BY RANGE (recorded_at);

-- Monthly partitions for the event store
-- CREATE TABLE event_store_2026_01 PARTITION OF event_store
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type, recorded_at DESC);
CREATE INDEX idx_event_stream_type ON event_store(stream_type, recorded_at DESC);
CREATE INDEX idx_event_correlation ON event_store(correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_event_occurred ON event_store(occurred_at DESC);
```

### Event Type Catalogue

```sql
-- Registry of all known event types with their JSON schema
CREATE TABLE event_type_registry (
    event_type          VARCHAR(100) PRIMARY KEY,
    stream_type         VARCHAR(50) NOT NULL,
    description         TEXT NOT NULL,
    payload_schema      JSONB NOT NULL,          -- JSON Schema for the payload
    version             INTEGER NOT NULL DEFAULT 1,
    deprecated          BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types:
-- VehicleRegistered, VehicleUpdated, VehicleRetired, VehicleAssignedToFleet
-- DriverOnboarded, DriverLicenceUpdated, DriverTerminated
-- PositionRecorded, TripStarted, TripEnded, JurisdictionCrossed
-- DutyStatusChanged, HosLogEdited, HosLogCertified, HosViolationDetected
-- WorkOrderCreated, WorkOrderAssigned, WorkOrderStarted, WorkOrderCompleted
-- FuelPurchased, FuelAnomalyDetected
-- GeofenceEntered, GeofenceExited, SpeedLimitExceeded
-- SafetyEventDetected, SafetyEventReviewed
-- InspectionSubmitted, InspectionDefectFound
-- DiagnosticFaultDetected, DiagnosticFaultCleared
-- MaintenanceDue, MaintenanceOverdue
```

### Event Payload Examples

```json
// VehicleRegistered
{
    "vin": "1FUJGLDR5DLBP8234",
    "name": "TRUCK-042",
    "vehicle_type": "truck",
    "make": "Freightliner",
    "model": "Cascadia",
    "year": 2024,
    "fuel_type": "diesel",
    "tank_capacity_litres": 450,
    "licence_plate": "ABC-1234",
    "fleet_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479"
}

// DutyStatusChanged (FMCSA ELD-compliant)
{
    "previous_status": "driving",
    "new_status": "on_duty_not_driving",
    "latitude": 41.8781,
    "longitude": -87.6298,
    "location_description": "Chicago, IL",
    "odometer_km": 234567.8,
    "engine_hours": 12345.6,
    "origin": "auto",
    "annotation": null,
    "cmv_power_unit": "TRUCK-042",
    "trailer_numbers": ["TRAIL-101"],
    "shipping_doc": "BOL-2026-4521"
}

// HosLogEdited (preserves original per FMCSA requirement)
{
    "original_event_id": "a1b2c3d4-...",
    "original_status": "off_duty",
    "edited_status": "sleeper_berth",
    "edit_reason": "Incorrect status selected",
    "original_start": "2026-05-21T22:00:00Z",
    "edited_start": "2026-05-21T22:00:00Z",
    "driver_annotation": "Was in sleeper berth, not off duty"
}

// WorkOrderProgressChanged
{
    "previous_status": "open",
    "new_status": "in_progress",
    "assigned_to": "uuid-of-mechanic",
    "vehicle_odometer_km": 145678.2,
    "notes": "Starting brake pad replacement"
}
```

---

## TimescaleDB Telemetry Hypertables

```sql
-- High-frequency vehicle position data as a TimescaleDB hypertable
CREATE TABLE telemetry_position (
    time                TIMESTAMPTZ NOT NULL,
    vehicle_id          UUID NOT NULL,
    device_id           UUID NOT NULL,
    latitude            NUMERIC(10,7) NOT NULL,
    longitude           NUMERIC(10,7) NOT NULL,
    altitude_m          NUMERIC(8,2),
    speed_kmh           NUMERIC(6,2),
    heading             NUMERIC(5,2),
    odometer_km         NUMERIC(12,2),
    engine_hours        NUMERIC(12,2),
    ignition_on         BOOLEAN,
    satellites          SMALLINT,
    hdop                NUMERIC(4,2),
    location            GEOGRAPHY(POINT, 4326)
);

-- Convert to hypertable with 1-day chunks
SELECT create_hypertable('telemetry_position', 'time',
    chunk_time_interval => INTERVAL '1 day');

-- Space partitioning by vehicle_id for parallel ingestion
SELECT add_dimension('telemetry_position', 'vehicle_id', number_partitions => 16);

CREATE INDEX idx_tpos_vehicle_time ON telemetry_position(vehicle_id, time DESC);
CREATE INDEX idx_tpos_location ON telemetry_position USING GIST(location);

-- Compression policy: compress chunks older than 7 days
SELECT add_compression_policy('telemetry_position', INTERVAL '7 days');

-- Retention policy: drop raw data older than 2 years (keep continuous aggregates)
SELECT add_retention_policy('telemetry_position', INTERVAL '2 years');
```

```sql
-- Vehicle sensor readings (engine data, fuel, temperatures)
CREATE TABLE telemetry_sensor (
    time                TIMESTAMPTZ NOT NULL,
    vehicle_id          UUID NOT NULL,
    device_id           UUID NOT NULL,
    parameter_name      VARCHAR(100) NOT NULL,  -- e.g., 'engine_rpm', 'coolant_temp_c', 'fuel_level_pct'
    parameter_source    VARCHAR(10) NOT NULL,    -- 'obd2', 'j1939', 'fms', 'iot'
    value_numeric       NUMERIC(15,4),
    value_text          VARCHAR(255),
    unit                VARCHAR(20)              -- 'rpm', 'celsius', 'percent', 'kpa', 'litres'
);

SELECT create_hypertable('telemetry_sensor', 'time',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_tsensor_vehicle_param ON telemetry_sensor(vehicle_id, parameter_name, time DESC);

SELECT add_compression_policy('telemetry_sensor', INTERVAL '7 days');
SELECT add_retention_policy('telemetry_sensor', INTERVAL '1 year');
```

```sql
-- Diagnostic fault codes as time-series events
CREATE TABLE telemetry_fault (
    time                TIMESTAMPTZ NOT NULL,
    vehicle_id          UUID NOT NULL,
    device_id           UUID NOT NULL,
    protocol            VARCHAR(10) NOT NULL,    -- 'obd2', 'j1939'
    code                VARCHAR(20) NOT NULL,
    spn                 INTEGER,                 -- J1939 Suspect Parameter Number
    fmi                 INTEGER,                 -- J1939 Failure Mode Identifier
    description         TEXT,
    severity            VARCHAR(20),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    odometer_km         NUMERIC(12,2),
    engine_hours        NUMERIC(12,2)
);

SELECT create_hypertable('telemetry_fault', 'time',
    chunk_time_interval => INTERVAL '1 week');

CREATE INDEX idx_tfault_vehicle ON telemetry_fault(vehicle_id, time DESC);
CREATE INDEX idx_tfault_code ON telemetry_fault(code, time DESC);
CREATE INDEX idx_tfault_active ON telemetry_fault(vehicle_id, time DESC) WHERE is_active = true;

SELECT add_compression_policy('telemetry_fault', INTERVAL '30 days');
```

### Continuous Aggregates for Dashboards

```sql
-- Hourly vehicle activity summary (materialised from telemetry_position)
CREATE MATERIALIZED VIEW vehicle_hourly_stats
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    vehicle_id,
    count(*) AS position_count,
    avg(speed_kmh) AS avg_speed_kmh,
    max(speed_kmh) AS max_speed_kmh,
    max(odometer_km) - min(odometer_km) AS distance_km,
    sum(CASE WHEN speed_kmh > 0 THEN 1 ELSE 0 END) AS moving_count,
    sum(CASE WHEN speed_kmh = 0 AND ignition_on = true THEN 1 ELSE 0 END) AS idle_count
FROM telemetry_position
GROUP BY bucket, vehicle_id;

-- Refresh policy: update every 15 minutes
SELECT add_continuous_aggregate_policy('vehicle_hourly_stats',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '15 minutes');
```

```sql
-- Daily sensor statistics for predictive maintenance ML features
CREATE MATERIALIZED VIEW sensor_daily_stats
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    vehicle_id,
    parameter_name,
    avg(value_numeric) AS avg_value,
    min(value_numeric) AS min_value,
    max(value_numeric) AS max_value,
    stddev(value_numeric) AS stddev_value,
    count(*) AS reading_count
FROM telemetry_sensor
WHERE value_numeric IS NOT NULL
GROUP BY bucket, vehicle_id, parameter_name;

SELECT add_continuous_aggregate_policy('sensor_daily_stats',
    start_offset => INTERVAL '2 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');
```

---

## Read Model Projections (Materialised Views from Events)

```sql
-- Current vehicle state projected from VehicleRegistered, VehicleUpdated, VehicleRetired events
CREATE TABLE rm_vehicle (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    fleet_id            UUID,
    name                VARCHAR(100) NOT NULL,
    vin                 VARCHAR(17),
    licence_plate       VARCHAR(20),
    make                VARCHAR(100),
    model               VARCHAR(100),
    year                SMALLINT,
    vehicle_type        VARCHAR(50),
    fuel_type           VARCHAR(30),
    status              VARCHAR(30) NOT NULL,
    current_driver_id   UUID,
    current_odometer_km NUMERIC(12,2),
    current_engine_hours NUMERIC(12,2),
    last_position_lat   NUMERIC(10,7),
    last_position_lng   NUMERIC(10,7),
    last_position_at    TIMESTAMPTZ,
    last_event_version  INTEGER NOT NULL,      -- for optimistic concurrency
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_vehicle_org ON rm_vehicle(organisation_id);
CREATE INDEX idx_rm_vehicle_fleet ON rm_vehicle(fleet_id);
```

```sql
-- Current driver state projected from driver events
CREATE TABLE rm_driver (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    user_id             UUID,
    employee_number     VARCHAR(50),
    first_name          VARCHAR(100),
    last_name           VARCHAR(100),
    licence_number      VARCHAR(50),
    licence_class       VARCHAR(20),
    licence_expiry      DATE,
    medical_card_expiry DATE,
    status              VARCHAR(30) NOT NULL,
    current_vehicle_id  UUID,
    current_duty_status VARCHAR(20),
    duty_status_since   TIMESTAMPTZ,
    safety_score        NUMERIC(5,2),
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_driver_org ON rm_driver(organisation_id);
```

```sql
-- Current work order state projected from work order events
CREATE TABLE rm_work_order (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    vehicle_id          UUID NOT NULL,
    work_order_number   VARCHAR(50),
    status              VARCHAR(30) NOT NULL,
    priority            VARCHAR(20),
    work_order_type     VARCHAR(30),
    title               VARCHAR(255),
    description         TEXT,
    assigned_to         UUID,
    scheduled_date      DATE,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    total_cost          NUMERIC(10,2),
    line_items          JSONB,             -- denormalised for read performance
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_wo_org ON rm_work_order(organisation_id);
CREATE INDEX idx_rm_wo_vehicle ON rm_work_order(vehicle_id);
CREATE INDEX idx_rm_wo_status ON rm_work_order(status);
```

```sql
-- HOS daily summary projected from DutyStatusChanged events
CREATE TABLE rm_hos_daily (
    driver_id           UUID NOT NULL,
    log_date            DATE NOT NULL,
    vehicle_id          UUID,
    total_driving_min   INTEGER NOT NULL DEFAULT 0,
    total_on_duty_min   INTEGER NOT NULL DEFAULT 0,
    total_sleeper_min   INTEGER NOT NULL DEFAULT 0,
    total_off_duty_min  INTEGER NOT NULL DEFAULT 0,
    remaining_drive_min INTEGER,           -- 11-hour limit minus driving
    remaining_window_min INTEGER,          -- 14-hour window remaining
    violations          JSONB,             -- array of violation summaries
    is_certified        BOOLEAN NOT NULL DEFAULT false,
    certified_at        TIMESTAMPTZ,
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (driver_id, log_date)
);
```

```sql
-- IFTA quarterly projection from JurisdictionCrossed and FuelPurchased events
CREATE TABLE rm_ifta_quarterly (
    organisation_id     UUID NOT NULL,
    quarter             VARCHAR(7) NOT NULL,   -- '2026-Q1'
    vehicle_id          UUID NOT NULL,
    jurisdiction_code   VARCHAR(10) NOT NULL,  -- ISO 3166-2
    total_distance_km   NUMERIC(12,2) NOT NULL DEFAULT 0,
    fuel_purchased_l    NUMERIC(10,2) NOT NULL DEFAULT 0,
    fuel_consumed_l     NUMERIC(10,2) NOT NULL DEFAULT 0,
    tax_rate            NUMERIC(8,4),
    tax_due             NUMERIC(10,2),
    last_event_id       UUID,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organisation_id, quarter, vehicle_id, jurisdiction_code)
);
```

---

## Snapshot Store (for Event Replay Optimisation)

```sql
-- Periodic snapshots to avoid replaying full event history
CREATE TABLE event_snapshot (
    stream_id           UUID NOT NULL,
    stream_type         VARCHAR(50) NOT NULL,
    snapshot_version    INTEGER NOT NULL,       -- event_version at snapshot time
    state               JSONB NOT NULL,         -- serialised aggregate state
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- Keep only the latest N snapshots per stream
CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, snapshot_version DESC);
```

---

## Supporting Reference Tables

```sql
-- These are NOT event-sourced — they are slow-changing reference data
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    parent_org_id   UUID REFERENCES organisation(id),
    dot_number      VARCHAR(20),
    ifta_account    VARCHAR(30),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    UNIQUE(organisation_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE jurisdiction (
    country_code        CHAR(2) NOT NULL,
    subdivision_code    VARCHAR(6) NOT NULL,
    name                VARCHAR(100) NOT NULL,
    ifta_member         BOOLEAN NOT NULL DEFAULT false,
    fuel_tax_rate       NUMERIC(8,4),
    PRIMARY KEY (country_code, subdivision_code)
);

CREATE TABLE geofence (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    name                VARCHAR(255) NOT NULL,
    boundary            GEOGRAPHY(POLYGON, 4326),
    centre_point        GEOGRAPHY(POINT, 4326),
    radius_m            NUMERIC(10,2),
    speed_limit_kmh     NUMERIC(6,2),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_geofence_boundary ON geofence USING GIST(boundary);
```

---

## Example Queries

### Reconstruct vehicle state at a specific point in time

```sql
-- Replay all events for a vehicle up to a specific timestamp
SELECT payload
FROM event_store
WHERE stream_id = '(vehicle-uuid)'
  AND stream_type = 'vehicle'
  AND occurred_at <= '2026-05-15T14:00:00Z'
ORDER BY event_version ASC;
```

### Query HOS compliance from the event stream

```sql
-- All duty status changes for a driver on a specific day
SELECT
    event_id,
    occurred_at,
    payload->>'previous_status' AS from_status,
    payload->>'new_status' AS to_status,
    payload->>'latitude' AS lat,
    payload->>'longitude' AS lng,
    payload->>'origin' AS source,
    actor_id
FROM event_store
WHERE stream_type = 'driver'
  AND stream_id = '(driver-uuid)'
  AND event_type = 'DutyStatusChanged'
  AND occurred_at >= '2026-05-21T00:00:00Z'
  AND occurred_at < '2026-05-22T00:00:00Z'
ORDER BY occurred_at;
```

### Time-series query: Average speed per hour for a vehicle

```sql
SELECT
    bucket,
    avg_speed_kmh,
    max_speed_kmh,
    distance_km
FROM vehicle_hourly_stats
WHERE vehicle_id = '(vehicle-uuid)'
  AND bucket >= now() - INTERVAL '24 hours'
ORDER BY bucket;
```

### Predictive maintenance: Sensor trend analysis

```sql
-- Detect coolant temperature trending upward over the past 30 days
SELECT
    bucket,
    avg_value,
    stddev_value,
    max_value
FROM sensor_daily_stats
WHERE vehicle_id = '(vehicle-uuid)'
  AND parameter_name = 'coolant_temp_c'
  AND bucket >= now() - INTERVAL '30 days'
ORDER BY bucket;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | event_store (partitioned), event_type_registry, event_snapshot |
| Telemetry Hypertables | 3 | telemetry_position, telemetry_sensor, telemetry_fault |
| Continuous Aggregates | 2 | vehicle_hourly_stats, sensor_daily_stats |
| Read Model Projections | 5 | rm_vehicle, rm_driver, rm_work_order, rm_hos_daily, rm_ifta_quarterly |
| Reference / Auth | 6 | organisation, app_user, role, user_role, jurisdiction, geofence |
| **Total** | **~19** | Plus TimescaleDB chunks and event store partitions |

---

## Key Design Decisions

1. **Single event store table** — all domain events go into one partitioned table rather than per-aggregate tables. This simplifies infrastructure, enables cross-aggregate queries (correlation_id), and keeps the projection logic in event handlers rather than the storage layer. Partitioning by `recorded_at` provides time-based data management.

2. **CQRS with explicit read models** — the `rm_*` (read model) tables are projections maintained by event handler code. They are fully rebuildable from the event store. This separation allows read models to be denormalised for specific query patterns without compromising the write-side integrity.

3. **TimescaleDB for telemetry, not the event store** — high-frequency telemetry (GPS positions at 1-30 second intervals, sensor readings) flows into purpose-built hypertables with compression and retention policies. These are treated as raw data rather than domain events. The event store captures semantically meaningful events (TripStarted, SpeedLimitExceeded) derived from the telemetry stream.

4. **Event versioning and schema evolution** — `event_type_registry` stores the JSON Schema for each event type with a version number. When event schemas evolve, the old schema is preserved and event handlers use upcasters to transform old events to the current shape during replay.

5. **Correlation and causation IDs** — `correlation_id` links events across aggregates (a single user action might create events in vehicle, driver, and work_order streams). `causation_id` tracks which event triggered a downstream event, enabling full event genealogy for debugging and audit.

6. **Snapshots for replay performance** — for aggregates with long event histories (vehicles with years of data), periodic snapshots serialise the current state. Replay starts from the latest snapshot rather than event zero, keeping reconstruction fast.

7. **Continuous aggregates for dashboards** — rather than querying raw telemetry for dashboard KPIs, TimescaleDB continuous aggregates pre-compute hourly and daily statistics. The dashboard reads from the aggregate, which is incrementally updated as new data arrives.

8. **FMCSA compliance is natural** — every HOS edit produces a new `HosLogEdited` event while the original `DutyStatusChanged` event remains immutable. The driver's annotation is captured in the edit event. This directly satisfies 49 CFR Part 395 requirements without any special audit table — the event store IS the audit trail.

9. **ML pipeline integration** — the telemetry hypertables and continuous aggregates produce exactly the feature vectors needed for predictive maintenance ML: daily min/max/avg/stddev of sensor parameters, fault code frequency, and operating pattern statistics. No separate ETL is needed.

10. **Separate reference data** — organisation, user, role, jurisdiction, and geofence tables are traditional CRUD — they change rarely and do not benefit from event sourcing. Mixing event-sourced and CRUD tables in the same database is a pragmatic choice that avoids over-engineering.
