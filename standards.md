# Standards & API Reference

> Project: Fleet Management Platform · Generated: 2026-05-03

---

## Industry Standards & Specifications

### Regulatory Mandates (US / Canada)

**FMCSA ELD Mandate — 49 CFR Part 395**
- URL: https://www.ecfr.gov/current/title-49/subtitle-B/chapter-III/subchapter-B/part-395
- Requires electronic logging devices in commercial motor vehicles to record Hours of Service data. Any fleet platform targeting US commercial trucking must integrate with FMCSA-certified ELD hardware and produce FMCSA-compliant HOS logs.

**FMCSA Hours of Service Rules — 49 CFR Part 395**
- URL: https://www.fmcsa.dot.gov/regulations/hours-service/summary-hours-service-regulations
- Defines maximum driving times, required rest periods, and 30-minute break rules for commercial drivers. Fleet software must track, alert on, and report HOS compliance automatically.

**IFTA — International Fuel Tax Agreement**
- URL: https://www.iftach.org/
- Fuel-tax reporting standard for interstate commercial carriers. Fleet software must compute miles driven per jurisdiction from GPS data and generate IFTA-compliant quarterly reports.

**Transport Canada ELD Mandate — SOR/2019-165**
- URL: https://tc.canada.ca/en/road-transportation/truck-bus-transportation/hours-service/electronic-logging-device-mandate
- Canadian federal ELD requirement for commercial vehicles crossing provincial and national borders; substantially similar to the US FMCSA mandate.

---

### ISO Standards

**ISO 39001:2012 — Road Traffic Safety Management Systems**
- URL: https://www.iso.org/standard/44958.html
- Specifies requirements for an RTS management system to help organisations reduce deaths and serious injuries related to road traffic. Relevant for fleet operators seeking certification and for safety-module design within fleet management software.

**ISO 55000:2014 — Asset Management**
- URL: https://www.iso.org/standard/55088.html
- Framework for managing the lifecycle of physical assets including vehicles, emphasising balancing cost, risk, and performance. Relevant to fleet asset lifecycle management, depreciation tracking, and retirement planning modules.

**ISO 23725:2024 — Autonomous System and Fleet Management System Interoperability**
- URL: https://www.iso.org/standard/76769.html
- Defines interfaces between fleet management systems (FMS) and autonomous haulage systems (AHS), covering communication protocols, message structures, telemetry signals, map sharing, and task assignments. Relevant for fleets integrating autonomous or semi-autonomous vehicles.

**ISO 15143-3:2020 (AEMP 2.0) — Worksite Equipment Data Web Service**
- URL: https://www.iso.org/standard/67556.html
- Formalises the AEMP Telematics Standard defining how construction and earthmoving equipment data is exposed over a web service interface. Key for mixed fleets that include off-road or construction machinery.

---

### Vehicle Communication Protocols

**SAE J1939 — CAN Bus Protocol for Heavy-Duty Vehicles**
- URL: https://www.sae.org/standards/content/j1939/
- Controller Area Network (CAN) based protocol for heavy-duty trucks, buses, and industrial vehicles. Provides engine speed, fuel consumption, coolant temperature, odometer, fault codes, and other parameters. The foundation of telematics data collection from commercial vehicles. Sub-standards include J1939-21 (data link layer), J1939-71 (vehicle application layer), and J1939-81 (network management).

**OBD-II — On-Board Diagnostics (SAE J1979 / ISO 15031)**
- URL: https://www.sae.org/standards/content/j1979/
- Standardised diagnostic port and protocol for passenger vehicles and light trucks. Exposes Diagnostic Trouble Codes (DTCs), real-time sensor data, and emissions compliance data. Essential for telematics hardware that plugs into the OBD-II port for fleet data collection.

**FMS Standard (Fleet Management System Interface)**
- URL: https://www.fms-standard.com/
- Developed in 2002 by Daimler, MAN, Scania, Volvo, DAF, and IVECO to enable manufacturer-independent telematics applications. Defines the vehicle gateway interface exposing speed, fuel consumption, engine hours, vehicle weight, tachograph data, and more over CAN. Widely adopted by European commercial vehicle manufacturers.

**ISO 15765-2 — UDS (Unified Diagnostic Services) over CAN**
- URL: https://www.iso.org/standard/66574.html
- Diagnostic communication protocol for modern vehicle ECUs, used in conjunction with OBD-II and J1939 for advanced fault code reading and ECU programming. Relevant for telematics devices that perform remote diagnostics.

---

### Data & API Standards

**OpenAPI 3.x (OAS 3)**
- URL: https://spec.openapis.org/oas/latest.html
- The de-facto standard for documenting REST APIs. All major fleet management platforms (Samsara, Geotab, Fleetio, Traccar) publish or should publish OpenAPI specifications. A new fleet management platform should publish an OAS 3.1 spec for all its API endpoints.

**JSON Schema (draft 2020-12)**
- URL: https://json-schema.org/
- Standard for describing and validating JSON data structures. Used for validating API request/response payloads and for defining fleet data models (vehicle records, trip events, maintenance records).

**GeoJSON (RFC 7946)**
- URL: https://datatracker.ietf.org/doc/html/rfc7946
- Standard for encoding geographic data structures in JSON. Essential for representing geofences, trip paths, depot locations, and real-time vehicle positions in fleet management APIs.

**GTFS (General Transit Feed Specification)**
- URL: https://gtfs.org/
- Open format for public transit schedules and route data, relevant for fleet management platforms serving transit agencies. Supported by Google Maps and OpenTripPlanner.

**WebSocket Protocol (RFC 6455)**
- URL: https://datatracker.ietf.org/doc/html/rfc6455
- Standard for full-duplex communication over a single TCP connection. Relevant for real-time vehicle position streaming to web dashboards and mobile apps (used by Traccar, FleetBase, and modern fleet tracking systems).

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Standard authorisation framework for API access delegation. Required for fleet management APIs that allow third-party integrations and Marketplace applications to access fleet data on behalf of users.

**OpenID Connect 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0 for user authentication. Relevant for fleet platforms supporting SSO with enterprise identity providers (Azure AD, Okta, Google Workspace).

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/www-project-api-security/
- Authoritative reference for API security best practices. Critical for a fleet management platform handling location data, driver PII, and safety-critical vehicle commands.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr.eu/
- EU regulation governing the collection and processing of personal data. Vehicle location data correlating to an identifiable driver constitutes personal data under GDPR. Fleet platforms must implement lawful basis for processing, data minimisation, configurable retention periods, driver privacy windows, and privacy impact assessments. The Data (Use and Access) Act 2025 (UK) introduced updates to recognised legitimate interests.

**CCPA — California Consumer Privacy Act**
- URL: https://oag.ca.gov/privacy/ccpa
- California privacy law with similar implications to GDPR for fleet operators with California-based drivers. Data subject access requests and opt-out rights must be supported.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is relevant to an AI-native fleet management platform for exposing fleet data and operations to LLM-based assistants and agents.

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/
- Open protocol for connecting AI assistants to external data sources and tools. A fleet management MCP server could expose resources (vehicles, drivers, trips, maintenance records) and tools (create work order, set geofence, run report) to any MCP-compatible AI client, enabling natural language fleet management without a custom chat UI.

---

## Similar Products — Developer Documentation & APIs

### Samsara
- **Description:** IoT-connected fleet management platform with GPS tracking, AI dashcams, ELD compliance, and safety analytics for commercial fleets.
- **API Documentation:** https://developers.samsara.com/docs/rest-api-overview
- **SDKs/Libraries:** JavaScript, Python SDKs listed at developers.samsara.com; App Marketplace SDKs available
- **Developer Guide:** https://www.samsara.com/support/developers
- **Standards:** REST/JSON; Webhooks for event-driven integrations
- **Authentication:** OAuth 2.0 bearer token

### Geotab (MyGeotab SDK)
- **Description:** Global fleet telematics platform with open SDK, 5M+ connected vehicles, and a 250+ app Marketplace ecosystem.
- **API Documentation:** https://geotab.github.io/sdk/software/api/reference/
- **SDKs/Libraries:** JavaScript (MyGeotab SDK); .NET, Java wrappers available via community
- **Developer Guide:** https://geotab.github.io/sdk/
- **Standards:** JSON-RPC over HTTPS; REST supplement; OpenAPI exposure via Analytics Lab API Explorer
- **Authentication:** Session token / API key

### Fleetio
- **Description:** Fleet maintenance management platform with public REST API, 50+ webhooks, and integrations for telematics providers and fuel card systems.
- **API Documentation:** https://developer.fleetio.com/
- **SDKs/Libraries:** REST/JSON; no official language SDKs published; community wrappers exist
- **Developer Guide:** https://www.fleetio.com/features/developer-api
- **Standards:** REST/JSON; OpenAPI documentation published
- **Authentication:** API Key (per-account token)

### Verizon Connect
- **Description:** Enterprise fleet tracking, route optimisation, ELD compliance, and dispatch management from Verizon's carrier-grade network.
- **API Documentation:** https://api-help.verizonconnect.com/
- **SDKs/Libraries:** REST API toolkit; no official published SDKs
- **Developer Guide:** https://www.verizonconnect.com/services/api-integration/
- **Standards:** REST/JSON
- **Authentication:** API Key / OAuth 2.0

### Motive (formerly KeepTruckin)
- **Description:** Fleet management, ELD compliance, AI dashcams, and integrated spend management for trucking and commercial transport.
- **API Documentation:** https://developer.gomotive.com/
- **SDKs/Libraries:** REST API; no official published SDKs
- **Developer Guide:** https://developer.gomotive.com/docs
- **Standards:** REST/JSON; Webhooks
- **Authentication:** OAuth 2.0

### Traccar
- **Description:** Open-source GPS tracking server supporting 2,000+ device models with REST API, WebSocket, and OpenAPI specification.
- **API Documentation:** https://www.traccar.org/api-reference/
- **SDKs/Libraries:** OpenAPI spec at https://www.traccar.org/api-reference/openapi.yaml; community-generated client SDKs in multiple languages
- **Developer Guide:** https://www.traccar.org/documentation/
- **Standards:** REST/JSON; OpenAPI 3.x; WebSocket for real-time streaming
- **Authentication:** Session cookie / API token (Basic Auth supported)
- **Licence:** Apache 2.0 (open source)

### FleetBase
- **Description:** Open-source logistics and fleet operations platform with API-first architecture, WebSocket support, and webhook automation.
- **API Documentation:** https://fleetbase.io/docs
- **SDKs/Libraries:** REST API; JavaScript SDK; community SDKs
- **Developer Guide:** https://fleetbase.io/docs/getting-started
- **Standards:** REST/JSON; WebSocket; Webhooks; event-driven architecture
- **Authentication:** API Key
- **Licence:** MIT (open source)

### Google Maps Platform (Fleet & Routing APIs)
- **Description:** Route Optimization API and Maps Platform providing routing, geocoding, and real-time traffic data used by many fleet management platforms as an underlying infrastructure component.
- **API Documentation:** https://developers.google.com/maps/documentation/route-optimization
- **SDKs/Libraries:** JavaScript, Python, Java, Go, Node.js, .NET client libraries
- **Developer Guide:** https://developers.google.com/maps/documentation
- **Standards:** REST/JSON; OpenAPI
- **Authentication:** API Key

### Here Technologies (Fleet & Routing)
- **Description:** Location platform providing routing, geocoding, fleet management APIs, and real-time traffic as an alternative to Google Maps, with no-lock-in licensing.
- **API Documentation:** https://developer.here.com/documentation
- **SDKs/Libraries:** JavaScript, Python, Java, Swift, Kotlin SDKs
- **Developer Guide:** https://developer.here.com/tutorials
- **Standards:** REST/JSON; OpenAPI
- **Authentication:** API Key / OAuth 2.0

### OpenStreetMap / Overpass API
- **Description:** Free, open-source geographic database used as the map tile and routing data source by open-source fleet management tools (Traccar, OpenGTS). The Overpass API enables querying OSM data programmatically.
- **API Documentation:** https://overpass-api.de/api/
- **SDKs/Libraries:** osmtogeojson (JavaScript); overpy (Python); various community tools
- **Developer Guide:** https://wiki.openstreetmap.org/wiki/Overpass_API
- **Standards:** Overpass QL; GeoJSON output; OSM XML
- **Authentication:** None required for public read access

---

## Notes

- **ELD Certification complexity:** US FMCSA ELD certification is required for hardware, not software, but software must interface correctly with certified devices. Third-party ELD device partnerships or integration with Samsara/Geotab hardware is the practical path for a new platform rather than pursuing independent hardware certification.

- **GDPR complexity in multi-jurisdiction deployments:** Fleet tracking platforms operating across EU, UK, US, and Canada must handle overlapping privacy frameworks with different retention periods, lawful bases, and data subject rights. Privacy-by-design architecture with configurable driver privacy windows (disabling tracking during personal use hours) is emerging as a standard feature requirement.

- **FMS Standard adoption:** The FMS standard is widely adopted by European truck manufacturers (Daimler, Volvo, Scania, MAN, DAF, IVECO) and provides a vehicle-side hardware-independent data interface. Supporting FMS alongside SAE J1939 and OBD-II covers the vast majority of commercial vehicle types globally.

- **MCP as a fleet management integration pattern:** The emergence of the Model Context Protocol as an interoperability standard for AI agents creates an opportunity to expose fleet management capabilities as MCP tools and resources, enabling any MCP-compatible AI assistant or agent to interact with fleet data without a custom integration layer.

- **OpenStreetMap as infrastructure:** Building on OpenStreetMap and open routing engines (OSRM, Valhalla, OpenRouteService) rather than Google Maps or HERE avoids per-call API costs that scale proportionally with fleet size, which is a significant operating-cost advantage for an open-source fleet platform.
