# Fleet Management Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source fleet management platform for vehicle tracking, maintenance, fuel management, and driver behaviour — without hardware lock-in or multi-year contracts.

Fleet Management Platform is a self-hostable system for organisations that operate vehicles — logistics carriers, service businesses, municipal fleets, and mixed industrial operators. It unifies real-time GPS tracking, maintenance workflows, compliance reporting, and AI-driven safety and predictive analytics into a single platform built on open standards.

---

## Why Fleet Management Platform?

- Incumbents like Samsara and Motive lock customers into proprietary hardware and multi-year contracts, with pricing rarely published and quotes ranging $27–$60 per vehicle per month.
- Verizon Connect customers report rigid contracts, glitchy mobile apps, and persistent customer service issues; Azuga mandates 36-month contracts — the longest in the industry.
- Fleetio offers strong maintenance UX but has no native GPS, ELD, or dashcam capability, forcing customers to stitch together multiple vendors.
- Existing open-source options (Traccar, FleetBase, OpenGTS) provide tracking primitives but lack maintenance workflows, ELD/HOS compliance, AI analytics, and modern UX.
- Only Samsara has launched a generative AI assistant for fleet operations; predictive maintenance across the market is still mostly rule-based threshold alerts rather than ML failure prediction.

---

## Key Features

### Tracking and Visibility

- Real-time GPS vehicle tracking with live map view using open standards and self-hostable map tiles
- Geofencing with configurable alerts for boundary crossings, speed thresholds, and excessive idle time
- Historical trip reporting and data export
- Asset tracking for trailers, equipment, and non-vehicle assets

### Maintenance and Lifecycle

- Preventive and corrective work orders with service scheduling
- Vehicle diagnostics with engine fault code (DTC) monitoring
- Cost tracking and total cost of ownership reporting per vehicle
- Parts inventory, vendor management, and recall/warranty tracking (planned tiers)

### Driver Safety and Compliance

- Driver behaviour monitoring with automated event logging for harsh braking, acceleration, and speeding
- Mobile driver app for inspection forms (DVIR), trip logging, and messaging
- ELD / HOS compliance module aligned with US FMCSA and Canada Transport requirements (v1.1)
- IFTA fuel-tax report generation (v1.1)

### Fuel, EV, and Sustainability

- Fuel management with fuel card integration, cost-per-mile tracking, and anomaly detection (v1.1)
- EV fleet management: battery state of charge, charging status, and range estimation (v1.1)
- ESG and emissions reporting with per-vehicle and per-route carbon footprint calculations (backlog)

### Platform and Integration

- REST API with webhook support for third-party integrations
- Role-based access control across fleet manager, driver, mechanic, and admin roles
- Multi-fleet and sub-fleet organisational hierarchy with permission scoping (v1.1)

---

## AI-Native Advantage

Most incumbents rely on rule-based threshold alerts and require navigating complex report builders to extract insights. This project builds AI in from the start: predictive maintenance using OBD-II and sensor data with ML failure models, real-time dashcam analytics for harsh-event and distraction detection, dynamic route optimisation that recalculates as traffic and job priorities change, and a natural language query interface for ad-hoc fleet reporting. Anomaly detection extends to fuel-card fraud, unauthorised vehicle use, and odometer manipulation.

---

## Tech Stack & Deployment

The platform is designed to be self-hostable, with a REST API and webhook event model as the primary integration surface. Mapping uses OpenStreetMap-compatible tiles. Hardware integration follows the precedent set by open-source platforms like Traccar, which supports 2,000+ GPS device models across 200+ protocols, allowing the platform to remain device-agnostic rather than locked to proprietary hardware. Mobile driver apps cover inspection, trip, and messaging workflows.

---

## Market Context

The fleet management software market is projected to surpass $30 billion in 2026, growing at approximately 16–18% CAGR (research.md). Per-vehicle pricing among incumbents ranges from $5–$60 per month, with hardware adding $100–$400 per vehicle upfront and enterprise contracts typically requiring multi-year commitments. Primary buyers are fleet managers, transportation directors, logistics and supply-chain managers, safety compliance officers, CFOs tracking fleet TCO, and municipal government fleet supervisors.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
