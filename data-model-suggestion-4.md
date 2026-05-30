# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Fleet Management Platform · Created: 2026-05-22

## Philosophy

This model adds a property graph layer on top of relational operational tables to capture the complex, evolving relationships that define fleet operations: which driver drives which vehicle on what route for which customer, which vehicles belong to which sub-fleets under which divisions, which maintenance vendor services which vehicle types in which regions, which telematics device is installed in which vehicle and reports to which gateway. These relationships change frequently, involve multiple degrees of connection, and are often queried by traversal ("find all drivers who have driven vehicles that had brake failures in the past 90 days").

The approach uses PostgreSQL with the Apache AGE extension (or a separate Neo4j instance) for the graph layer, combined with standard relational tables for operational CRUD, time-series telemetry, and compliance data. The graph stores entities as nodes with properties and relationships as typed, directed edges with properties (including temporal validity). The relational tables remain the system of record for transactional data; the graph is a queryable projection optimised for relationship-heavy queries.

Real-world precedents include logistics platforms that model supply chain networks as graphs (delivery routes, hub connections, carrier relationships), social network-style organisational hierarchies, and knowledge graphs used in predictive maintenance to connect fault codes → components → failure modes → parts → vendors. Neo4j is used in fleet management by companies modelling complex asset ownership chains and multi-modal transport networks.

**Best for:** Large fleet operations with complex organisational hierarchies, multi-vendor maintenance networks, driver-vehicle-route relationship analysis, and AI-powered anomaly detection that benefits from graph traversal patterns.

**Trade-offs:**
- (+) Relationship queries are natural and fast — "find all X connected to Y through Z" without multi-table joins
- (+) Organisational hierarchies (company → division → fleet → sub-fleet → vehicle) traversed in single queries
- (+) Fraud and anomaly detection benefits from graph pattern matching
- (+) Maintenance knowledge graph connects symptoms → causes → parts → vendors for intelligent recommendations
- (+) Temporal edges track relationship history naturally
- (-) Additional technology (Apache AGE or Neo4j) adds operational complexity
- (-) Graph and relational stores must be kept in sync — requires event-driven synchronisation
- (-) Graph query languages (Cypher, openCypher) require team learning investment
- (-) Not all queries benefit from graph — simple CRUD and time-series are better served by relational/time-series stores
- (-) Fewer mature ORM/framework integrations for graph databases

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1/3166-2 | Jurisdiction nodes in the graph with ISO codes; edges connect vehicles and drivers to their operating jurisdictions |
| SAE J1939 / OBD-II | Fault code nodes linked to component nodes and vehicle nodes; enables graph-based root cause analysis |
| FMCSA 49 CFR Part 395 | HOS compliance data in relational tables; driver-vehicle-carrier relationships in graph for compliance network queries |
| IFTA | Jurisdiction traversal computed from trip-jurisdiction edges in the graph |
| GeoJSON / RFC 7946 | Geofence boundaries in PostGIS; geofence-vehicle relationship events as graph edges |
| ISO 55000 | Asset lifecycle relationships (vehicle → owner → lease → disposal) modelled as temporal graph edges |
| ISO 39001 | Safety event analysis via graph: driver → event → vehicle → component → maintenance_action paths |

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│                   Application Layer                   │
│         REST API  ·  GraphQL  ·  WebSocket           │
└──────────┬───────────────────────────┬───────────────┘
           │                           │
           ▼                           ▼
┌──────────────────┐       ┌──────────────────────────┐
│   PostgreSQL     │       │   Graph Store            │
│   (Relational)   │◄─────▶│   (Apache AGE / Neo4j)   │
│                  │ sync  │                          │
│ • Vehicles       │       │ • Organisation hierarchy │
│ • Drivers        │       │ • Vehicle-Driver assign  │
│ • Work orders    │       │ • Route networks         │
│ • Positions      │       │ • Maintenance knowledge  │
│ • HOS logs       │       │ • Vendor relationships   │
│ • Fuel entries   │       │ • Fault-Component links  │
│ • Inspections    │       │ • Fleet topology         │
└──────────────────┘       └──────────────────────────┘
```

---

## Graph Layer: Node and Edge Definitions

### Node Types

```cypher
// Organisation hierarchy nodes
(:Organisation {id: UUID, name: String, slug: String, org_type: String, dot_number: String})
(:Fleet {id: UUID, name: String, description: String})

// Asset nodes
(:Vehicle {id: UUID, name: String, vin: String, vehicle_type: String, fuel_type: String, 
           status: String, year: Int, make: String, model: String})
(:TelematicsDevice {id: UUID, serial_number: String, device_type: String, manufacturer: String})
(:Trailer {id: UUID, name: String, trailer_type: String, length_m: Float})

// People nodes
(:Driver {id: UUID, employee_number: String, first_name: String, last_name: String, 
          licence_class: String, status: String})
(:Mechanic {id: UUID, first_name: String, last_name: String, specialities: [String]})
(:User {id: UUID, email: String, first_name: String, last_name: String})

// Location nodes
(:Location {id: UUID, name: String, location_type: String, latitude: Float, longitude: Float,
            address: String})
(:Jurisdiction {code: String, name: String, country: String, ifta_member: Boolean})
(:Geofence {id: UUID, name: String, geofence_type: String})

// Maintenance knowledge graph nodes
(:Component {id: UUID, name: String, category: String, system: String})
  // e.g., "brake_pad_front", "oil_filter", "alternator", "turbocharger"
(:FaultCode {code: String, protocol: String, spn: Int, fmi: Int, description: String})
(:FailureMode {id: UUID, name: String, description: String, severity: String})
(:Part {id: UUID, part_number: String, name: String, manufacturer: String})
(:ServiceTask {id: UUID, name: String, category: String, estimated_hours: Float})

// Vendor nodes
(:Vendor {id: UUID, name: String, vendor_type: String, rating: Float})

// Route and customer nodes
(:Route {id: UUID, name: String, distance_km: Float})
(:Customer {id: UUID, name: String})
```

### Edge Types (Relationships)

```cypher
// Organisational hierarchy
(:Organisation)-[:HAS_CHILD_ORG]->(:Organisation)
(:Organisation)-[:OPERATES_FLEET]->(:Fleet)
(:Fleet)-[:CONTAINS_VEHICLE {since: DateTime, until: DateTime}]->(:Vehicle)
(:Organisation)-[:EMPLOYS {since: DateTime, until: DateTime, role: String}]->(:Driver)
(:Organisation)-[:EMPLOYS {since: DateTime, until: DateTime, role: String}]->(:Mechanic)
(:Organisation)-[:CONTRACTS_WITH {since: DateTime, contract_type: String}]->(:Vendor)

// Vehicle relationships
(:Driver)-[:DRIVES {since: DateTime, until: DateTime, is_primary: Boolean}]->(:Vehicle)
(:Vehicle)-[:HAS_DEVICE {installed_at: DateTime, removed_at: DateTime}]->(:TelematicsDevice)
(:Vehicle)-[:TOWS {since: DateTime, until: DateTime}]->(:Trailer)
(:Vehicle)-[:LOCATED_AT {since: DateTime}]->(:Location)
(:Vehicle)-[:OPERATES_IN {since: DateTime}]->(:Jurisdiction)
(:Vehicle)-[:HAS_COMPONENT {installed_at: DateTime, replaced_at: DateTime}]->(:Component)

// Maintenance knowledge graph edges
(:FaultCode)-[:INDICATES]->(:FailureMode)
(:FailureMode)-[:AFFECTS]->(:Component)
(:Component)-[:REQUIRES_PART]->(:Part)
(:ServiceTask)-[:ADDRESSES]->(:FailureMode)
(:Vendor)-[:SUPPLIES {lead_time_days: Int, unit_cost: Float}]->(:Part)
(:Vendor)-[:SERVICES {service_area_km: Float}]->(:Component)
(:Mechanic)-[:QUALIFIED_FOR]->(:ServiceTask)
(:Mechanic)-[:CERTIFIED_ON]->(:Component)

// Route and operations
(:Route)-[:STARTS_AT]->(:Location)
(:Route)-[:ENDS_AT]->(:Location)
(:Route)-[:PASSES_THROUGH]->(:Jurisdiction)
(:Route)-[:PASSES_THROUGH]->(:Geofence)
(:Vehicle)-[:TRAVELLED {trip_id: UUID, date: Date, distance_km: Float}]->(:Route)
(:Customer)-[:SERVICED_VIA]->(:Route)
(:Customer)-[:USES_VEHICLE {contract_id: String}]->(:Vehicle)

// Safety edges
(:Driver)-[:HAD_SAFETY_EVENT {event_type: String, severity: String, date: DateTime}]->(:Vehicle)
(:Vehicle)-[:ENTERED {at: DateTime}]->(:Geofence)
(:Vehicle)-[:EXITED {at: DateTime}]->(:Geofence)

// Maintenance edges
(:Vehicle)-[:SERVICED_BY {work_order_id: UUID, date: Date}]->(:Vendor)
(:Vehicle)-[:HAD_FAULT {detected_at: DateTime, cleared_at: DateTime}]->(:FaultCode)
```

---

## Relational Tables (Operational CRUD)

The relational layer handles transactional data, time-series telemetry, and compliance records. These tables are similar to Model 1 (normalised) but leaner, since relationship data lives in the graph.

### Core Entity Tables

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    dot_number      VARCHAR(20),
    ifta_account    VARCHAR(30),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
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
    roles           TEXT[] NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE TABLE vehicle (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(100) NOT NULL,
    vin             VARCHAR(17),
    vehicle_type    VARCHAR(50) NOT NULL,
    fuel_type       VARCHAR(30) NOT NULL DEFAULT 'diesel',
    make            VARCHAR(100),
    model           VARCHAR(100),
    year            SMALLINT,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    odometer_km     NUMERIC(12,2),
    engine_hours    NUMERIC(12,2),
    specs           JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vehicle_org ON vehicle(organisation_id);
CREATE INDEX idx_vehicle_vin ON vehicle(vin);
CREATE INDEX idx_vehicle_status ON vehicle(status);

CREATE TABLE driver (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES app_user(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    employee_number VARCHAR(50),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    licence_number  VARCHAR(50),
    licence_class   VARCHAR(20),
    licence_expiry  DATE,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    qualifications  JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_driver_org ON driver(organisation_id);

CREATE TABLE vendor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    vendor_type     VARCHAR(50),
    contact_info    JSONB NOT NULL DEFAULT '{}',
    service_areas   JSONB NOT NULL DEFAULT '[]',
    specialities    TEXT[],
    rating          NUMERIC(3,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vendor_org ON vendor(organisation_id);
```

### Maintenance Knowledge Base (Relational + Graph)

```sql
-- Components that vehicles contain (relational for CRUD, mirrored to graph)
CREATE TABLE component (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100) NOT NULL,    -- 'engine', 'brakes', 'electrical', 'drivetrain'
    system          VARCHAR(100),             -- 'cooling', 'exhaust', 'fuel', 'steering'
    description     TEXT,
    typical_lifespan_km     NUMERIC(10,2),
    typical_lifespan_hours  NUMERIC(10,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Known fault codes linked to failure modes
CREATE TABLE fault_code (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(20) NOT NULL,
    protocol        VARCHAR(10) NOT NULL,
    spn             INTEGER,
    fmi             INTEGER,
    description     TEXT NOT NULL,
    severity        VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(code, protocol)
);

CREATE TABLE failure_mode (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    severity        VARCHAR(20) NOT NULL,
    symptoms        TEXT[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- These junction tables are ALSO represented as graph edges for traversal queries
CREATE TABLE fault_indicates_failure (
    fault_code_id   UUID NOT NULL REFERENCES fault_code(id),
    failure_mode_id UUID NOT NULL REFERENCES failure_mode(id),
    confidence      NUMERIC(3,2) NOT NULL DEFAULT 1.0,  -- probability this fault indicates this failure
    PRIMARY KEY (fault_code_id, failure_mode_id)
);

CREATE TABLE failure_affects_component (
    failure_mode_id UUID NOT NULL REFERENCES failure_mode(id),
    component_id    UUID NOT NULL REFERENCES component(id),
    PRIMARY KEY (failure_mode_id, component_id)
);

CREATE TABLE part (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID REFERENCES organisation(id),
    part_number     VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    manufacturer    VARCHAR(100),
    unit_cost       NUMERIC(10,2),
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    reorder_point   INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE component_requires_part (
    component_id    UUID NOT NULL REFERENCES component(id),
    part_id         UUID NOT NULL REFERENCES part(id),
    quantity        INTEGER NOT NULL DEFAULT 1,
    PRIMARY KEY (component_id, part_id)
);
```

### Operational Tables

```sql
CREATE TABLE vehicle_position (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id      UUID NOT NULL REFERENCES vehicle(id),
    recorded_at     TIMESTAMPTZ NOT NULL,
    latitude        NUMERIC(10,7) NOT NULL,
    longitude       NUMERIC(10,7) NOT NULL,
    speed_kmh       NUMERIC(6,2),
    heading         NUMERIC(5,2),
    odometer_km     NUMERIC(12,2),
    engine_hours    NUMERIC(12,2),
    ignition_on     BOOLEAN,
    location        GEOGRAPHY(POINT, 4326),
    telemetry       JSONB NOT NULL DEFAULT '{}'
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_pos_vehicle_time ON vehicle_position(vehicle_id, recorded_at DESC);
CREATE INDEX idx_pos_location ON vehicle_position USING GIST(location);
```

```sql
CREATE TABLE trip (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id      UUID NOT NULL REFERENCES vehicle(id),
    driver_id       UUID REFERENCES driver(id),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ,
    distance_km     NUMERIC(10,2),
    duration_seconds INTEGER,
    fuel_consumed_l NUMERIC(8,2),
    route_summary   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trip_vehicle ON trip(vehicle_id, start_time DESC);
CREATE INDEX idx_trip_driver ON trip(driver_id, start_time DESC);
```

```sql
CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    vehicle_id      UUID NOT NULL REFERENCES vehicle(id),
    work_order_number VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    work_order_type VARCHAR(30) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    assigned_to     UUID REFERENCES app_user(id),
    vendor_id       UUID REFERENCES vendor(id),
    scheduled_date  DATE,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    total_cost      NUMERIC(10,2) DEFAULT 0,
    line_items      JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_org ON work_order(organisation_id);
CREATE INDEX idx_wo_vehicle ON work_order(vehicle_id);
CREATE INDEX idx_wo_status ON work_order(status);
```

```sql
CREATE TABLE vehicle_fault_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id      UUID NOT NULL REFERENCES vehicle(id),
    fault_code_id   UUID NOT NULL REFERENCES fault_code(id),
    detected_at     TIMESTAMPTZ NOT NULL,
    cleared_at      TIMESTAMPTZ,
    odometer_km     NUMERIC(12,2),
    engine_hours    NUMERIC(12,2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vfe_vehicle ON vehicle_fault_event(vehicle_id, detected_at DESC);
CREATE INDEX idx_vfe_fault ON vehicle_fault_event(fault_code_id);
CREATE INDEX idx_vfe_active ON vehicle_fault_event(vehicle_id) WHERE is_active = true;
```

```sql
CREATE TABLE safety_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id      UUID NOT NULL REFERENCES vehicle(id),
    driver_id       UUID REFERENCES driver(id),
    event_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL,
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    event_data      JSONB NOT NULL DEFAULT '{}',
    review_status   VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_safety_vehicle ON safety_event(vehicle_id, occurred_at DESC);
CREATE INDEX idx_safety_driver ON safety_event(driver_id, occurred_at DESC);
```

```sql
CREATE TABLE fuel_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id      UUID NOT NULL REFERENCES vehicle(id),
    driver_id       UUID REFERENCES driver(id),
    transaction_date TIMESTAMPTZ NOT NULL,
    fuel_type       VARCHAR(30) NOT NULL,
    quantity_litres NUMERIC(8,2) NOT NULL,
    total_cost      NUMERIC(10,2),
    odometer_km     NUMERIC(12,2),
    jurisdiction_code VARCHAR(10),
    transaction_data JSONB NOT NULL DEFAULT '{}',
    is_anomalous    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fuel_vehicle ON fuel_entry(vehicle_id, transaction_date DESC);
```

```sql
CREATE TABLE hos_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    driver_id       UUID NOT NULL REFERENCES driver(id),
    vehicle_id      UUID NOT NULL REFERENCES vehicle(id),
    log_date        DATE NOT NULL,
    duty_status     VARCHAR(20) NOT NULL,
    status_start    TIMESTAMPTZ NOT NULL,
    status_end      TIMESTAMPTZ,
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    origin          VARCHAR(20) NOT NULL DEFAULT 'auto',
    eld_data        JSONB NOT NULL DEFAULT '{}',
    certified       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(driver_id, log_date, status_start)
);

CREATE INDEX idx_hos_driver ON hos_log(driver_id, log_date DESC);
```

```sql
CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id      UUID NOT NULL REFERENCES vehicle(id),
    driver_id       UUID NOT NULL REFERENCES driver(id),
    inspection_type VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'passed',
    inspected_at    TIMESTAMPTZ NOT NULL,
    items           JSONB NOT NULL,
    odometer_km     NUMERIC(12,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_vehicle ON inspection(vehicle_id, inspected_at DESC);
```

```sql
CREATE TABLE geofence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    boundary        GEOGRAPHY(POLYGON, 4326),
    centre_point    GEOGRAPHY(POINT, 4326),
    radius_m        NUMERIC(10,2),
    properties      JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_geofence_org ON geofence(organisation_id);
CREATE INDEX idx_geofence_boundary ON geofence USING GIST(boundary);
```

```sql
CREATE TABLE alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    vehicle_id      UUID REFERENCES vehicle(id),
    driver_id       UUID REFERENCES driver(id),
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL,
    acknowledged    BOOLEAN NOT NULL DEFAULT false,
    context         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_org ON alert(organisation_id, occurred_at DESC);
```

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     UUID,
    changes         JSONB,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_audit_org ON audit_log(organisation_id, occurred_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Graph Query Examples (Cypher)

### Traverse organisation hierarchy to find all vehicles

```cypher
// Find all vehicles across an organisation and its sub-organisations
MATCH (org:Organisation {id: 'org-uuid'})-[:HAS_CHILD_ORG*0..5]->(sub)
      -[:OPERATES_FLEET]->(fleet)-[:CONTAINS_VEHICLE]->(v:Vehicle)
WHERE v.status = 'active'
RETURN sub.name AS division, fleet.name AS fleet, v.name AS vehicle, v.vin
ORDER BY division, fleet, vehicle
```

### Find drivers qualified to service a failed component

```cypher
// Vehicle has fault code P0301 — find the root cause and qualified mechanics
MATCH (v:Vehicle {id: 'vehicle-uuid'})-[:HAS_FAULT]->(fc:FaultCode {code: 'P0301'})
      -[:INDICATES]->(fm:FailureMode)-[:AFFECTS]->(comp:Component),
      (st:ServiceTask)-[:ADDRESSES]->(fm),
      (mech:Mechanic)-[:QUALIFIED_FOR]->(st)
RETURN fc.description, fm.name AS failure, comp.name AS component,
       st.name AS required_service, mech.first_name + ' ' + mech.last_name AS mechanic
```

### Find optimal vendor for a repair

```cypher
// Find vendors that supply the needed part AND service the component type,
// ranked by cost and proximity
MATCH (v:Vehicle {id: 'vehicle-uuid'})-[:HAS_FAULT]->(fc:FaultCode)
      -[:INDICATES]->(fm:FailureMode)-[:AFFECTS]->(comp:Component)
      -[:REQUIRES_PART]->(part:Part),
      (vendor:Vendor)-[:SUPPLIES {unit_cost: cost}]->(part),
      (vendor)-[:SERVICES]->(comp)
RETURN vendor.name, part.part_number, cost, vendor.rating
ORDER BY cost ASC, vendor.rating DESC
```

### Identify drivers affected by a component recall

```cypher
// Find all drivers who have driven vehicles with a specific component
MATCH (comp:Component {name: 'turbocharger_model_x'})
      <-[:HAS_COMPONENT]-(v:Vehicle)
      <-[:DRIVES]-(d:Driver)
WHERE d.status = 'active'
RETURN DISTINCT d.first_name, d.last_name, d.employee_number,
       v.name, v.vin
```

### Multi-hop safety analysis

```cypher
// Find all vehicles driven by drivers who had critical safety events
// in the past 90 days, including other vehicles those drivers currently drive
MATCH (d:Driver)-[se:HAD_SAFETY_EVENT]->(v1:Vehicle)
WHERE se.severity = 'critical' AND se.date > datetime() - duration('P90D')
MATCH (d)-[dr:DRIVES]->(v2:Vehicle)
WHERE dr.until IS NULL  // currently active assignment
RETURN d.first_name, d.last_name,
       v1.name AS incident_vehicle, se.event_type, se.date,
       collect(DISTINCT v2.name) AS current_vehicles
```

### IFTA jurisdiction traversal

```cypher
// Summarise distance per jurisdiction for a vehicle's trips
MATCH (v:Vehicle {id: 'vehicle-uuid'})-[t:TRAVELLED]->(r:Route)
      -[:PASSES_THROUGH]->(j:Jurisdiction)
WHERE t.date >= date('2026-04-01') AND t.date < date('2026-07-01')
RETURN j.code, j.name, sum(t.distance_km) AS total_km
ORDER BY total_km DESC
```

### Fleet topology visualisation

```cypher
// Export the full fleet graph for visualisation
MATCH path = (org:Organisation {id: 'org-uuid'})-[:HAS_CHILD_ORG*0..3]->()
              -[:OPERATES_FLEET]->()-[:CONTAINS_VEHICLE]->()
RETURN path
```

---

## Graph Synchronisation Strategy

The relational database is the write-side system of record. Graph updates are triggered by application events:

```
┌─────────────┐    INSERT/UPDATE    ┌──────────────────┐
│ Application │ ──────────────────▶ │   PostgreSQL     │
│   Service   │                     │   (Relational)   │
└──────┬──────┘                     └──────────────────┘
       │
       │  Event published
       ▼
┌──────────────┐    Graph mutation   ┌──────────────────┐
│  Event Bus   │ ──────────────────▶ │   Graph Store    │
│ (async)      │                     │ (AGE / Neo4j)    │
└──────────────┘                     └──────────────────┘
```

Key sync events:
- Vehicle created/updated → upsert Vehicle node + Organisation edge
- Driver assigned to vehicle → create/update DRIVES edge with timestamps
- Work order completed → create SERVICED_BY edge between Vehicle and Vendor
- Fault code detected → create HAS_FAULT edge; link to FaultCode node
- Trip completed → create TRAVELLED edge with Route and Jurisdiction connections

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Entities | 4 | organisation, app_user, vehicle, driver |
| Maintenance Knowledge | 6 | component, fault_code, failure_mode, fault_indicates_failure, failure_affects_component, component_requires_part |
| Parts & Vendors | 2 | part, vendor |
| Operations | 5 | vehicle_position (partitioned), trip, work_order, vehicle_fault_event, safety_event |
| Compliance | 2 | hos_log, fuel_entry |
| Location & Alerts | 4 | geofence, inspection, alert, audit_log (partitioned) |
| **Relational Total** | **~23** | Plus graph store with ~15 node types and ~25 edge types |
| Graph Nodes | ~15 types | Organisation, Fleet, Vehicle, Driver, Mechanic, Component, FaultCode, FailureMode, Part, ServiceTask, Vendor, Location, Jurisdiction, Route, Customer |
| Graph Edges | ~25 types | Organisational, operational, maintenance knowledge, and spatial relationships |

---

## Key Design Decisions

1. **Graph for relationships, relational for records** — the graph excels at traversing connections (who drives what, which vendor supplies which part for which component). The relational store excels at transactional CRUD, time-series queries, and compliance record keeping. Each technology handles what it does best.

2. **Maintenance knowledge graph** — the chain FaultCode → FailureMode → Component → Part → Vendor is the highest-value graph structure. It enables intelligent maintenance recommendations: "Vehicle TRUCK-042 has fault code P0301 (cylinder 1 misfire). This indicates ignition coil failure affecting the ignition system. Required part: IC-4420 (ignition coil). Nearest vendor with stock: Joe's Truck Service (3.2 km away, $85/unit, 4.8-star rating)."

3. **Temporal edges** — graph edges include `since` and `until` timestamps. This means the graph stores relationship history: "Driver Jane Smith drove Vehicle TRUCK-042 from 2024-03-15 to 2025-12-01, then switched to VAN-008." Temporal edges enable historical analysis without querying a separate assignment history table.

4. **Asynchronous graph sync** — changes to the relational store trigger events that asynchronously update the graph. This means the graph may lag the relational store by milliseconds to seconds, which is acceptable for analytical queries. The relational store is always the source of truth for operational transactions.

5. **Apache AGE as the default graph option** — Apache AGE is a PostgreSQL extension that adds openCypher graph query capability within PostgreSQL itself. This avoids running a separate Neo4j instance. For teams that need Neo4j's more mature graph features (visual browser, advanced algorithms), a standalone Neo4j instance connected via the event bus is the alternative.

6. **Confidence-weighted fault-failure links** — the `fault_indicates_failure` junction table includes a `confidence` score (0.0-1.0). A fault code like P0300 (random misfire) could indicate multiple failure modes with different probabilities. The confidence score, refined over time by ML on actual repair outcomes, enables probabilistic diagnosis recommendations.

7. **Graph-powered anomaly detection** — graph pattern matching can detect anomalies that are invisible in relational queries: "Driver X only has safety events when driving Vehicle Y" (potential vehicle issue, not driver issue), "Vendor Z's repairs on Component A fail within 30 days more often than other vendors" (quality issue), "Vehicles that visited Location Q develop fault code P0455 within 2 weeks" (environmental contamination).

8. **Dual query paths** — simple lookups (get vehicle by ID, list work orders by status) hit the relational database directly. Complex relationship queries (organisation hierarchy traversal, maintenance recommendation chains, multi-hop safety analysis) hit the graph. The API layer routes queries to the appropriate store.

9. **Route as a graph concept** — routes are modelled as graph nodes connected to start/end locations and traversed jurisdictions. This enables queries like "which routes cross three or more IFTA jurisdictions?" or "which customers are served by routes that pass through geofence X?" without complex relational joins.

10. **Graph enables natural language AI queries** — the maintenance knowledge graph provides structured context for an AI assistant. When a user asks "Why does truck 42 keep having engine problems?", the AI can traverse the graph: Vehicle → recent faults → failure modes → components → past work orders → vendors → parts used, constructing a narrative from the graph path rather than joining 8 relational tables.
