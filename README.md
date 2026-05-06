# Cleaning Services Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native operations platform for residential and commercial cleaning businesses — unifying scheduling, cleaner mobile workflows, quality inspection, and client portals in a single stack.

The Cleaning Services Platform is a candidate open-source system for the highly fragmented cleaning industry, where over 90% of commercial operators run with fewer than ten employees. It targets cleaning business owners, operations managers, and franchise operators who today must stitch together residential-focused tools (ZenMaid, Housecall Pro), commercial-only platforms (Swept, Janitorial Manager), inspection-only apps (OrangeQC), and general field service systems (Jobber, ServiceM8) to cover a single workflow.

---

## Why Cleaning Services Platform?

- **No incumbent spans residential and commercial.** ZenMaid is residential-only; Swept and Janitorial Manager are commercial-only; mixed operators have to run two systems.
- **Quality inspection is bolted on or absent.** Jobber, Housecall Pro, and GorillaDesk lack native quality checklists and photo-based proof-of-service; OrangeQC handles inspection but provides no scheduling, billing, or client acquisition.
- **APIs are gated or missing.** Housecall Pro restricts API access to its highest-tier MAX plan; ZenMaid, Booking Koala, GorillaDesk, Janitorial Manager, OrangeQC, and Fixlify publish no public API at all.
- **Per-seat pricing punishes growth.** ZenMaid charges $4/seat on top of base; Jobber bills per user; this taxes cleaning businesses precisely as they hire more cleaners.
- **Compliance tracking is unmet.** No reviewed platform tracks LEED Green Cleaning, EPA Safer Choice, CIMS, or OSHA SDS requirements that increasingly appear in institutional contracts.

---

## Key Features

### Scheduling & Dispatch

- Recurring job scheduling with a drag-and-drop calendar and multi-view (weekly grid, daily list, team availability)
- AI-assisted route optimisation across multiple cleaners using proximity, job duration, and traffic
- Skill- and history-aware assignment for matching cleaners to job requirements
- Mobile cleaner app with shift instructions and offline support

### Client Acquisition & Portal

- Embeddable self-service booking form with online payment at booking
- Client portal for viewing appointments, approving quotes, and paying invoices
- Automated SMS and email reminders, post-service summaries, and review requests
- Quote-to-proposal pipeline with online client approval

### Quality Inspection & Proof of Service

- Customisable inspection templates per location with photo capture and annotation
- GPS-tagged, timestamped check-in/check-out and inspection records
- Client-facing PDF inspection reports delivered automatically on a recurring schedule
- Photo-based before/after job documentation tied to each visit

### Workforce & Operations

- GPS clock-in/clock-out with geofencing for site-level accountability
- Multi-language cleaner app interface (English and Spanish at minimum)
- Cleaner retention dashboard surfacing workload and turnover-risk patterns
- Equipment, asset, and chemical inventory tracking

### Billing & Compliance

- Recurring invoicing with saved-card charging and automated payment reminders
- QuickBooks Online and Stripe native integrations
- Green-product compliance tracking against LEED, EPA Safer Choice, and CIMS-GB
- OSHA Safety Data Sheet (SDS) management for chemical inventories

---

## AI-Native Advantage

Unlike incumbents that bolt AI onto a legacy scheduling core, the platform treats intelligence as a first-class layer: ML-based route optimisation replaces manual dispatch, computer vision on submitted photos flags missed areas before clients notice them, and a dynamic pricing engine adjusts quotes from property size, frequency, market rates, and historical job-duration data. An AI phone and chat agent captures leads outside business hours, while anomaly detection on shift and payroll patterns surfaces cleaners at risk of turnover so operators can intervene early.

---

## Tech Stack & Deployment

The platform is intended to ship as a self-hostable open-source stack with an optional managed cloud offering. An API-first design — public REST/GraphQL endpoints with webhooks — lets third-party apps compose a best-of-breed cleaning stack rather than locking operators into a single vendor. Mobile clients target iOS and Android with parity, addressing a gap where ServiceM8 favours iOS and several incumbents treat Android as secondary. Standards alignment includes ISSA CIMS, LEED Green Cleaning, OSHA HazCom (29 CFR 1910.1200), EPA Safer Choice, and HIPAA-aware data handling for healthcare cleaning contexts.

---

## Market Context

The global commercial cleaning services market exceeds $400 billion in revenue, and the US residential cleaning market is estimated at over $10 billion (research.md). Incumbent pricing ranges from $19–$49/month for owner-operator tools (ZenMaid, ServiceM8, Housecall Pro, GorillaDesk) to $100–$500/month for mid-market platforms and custom enterprise contracts, with payment-processing fees of 2.9% + 30¢ scrutinised by buyers. Primary buyers are residential cleaning business owners, commercial janitorial operations managers, franchise operators, hospitality housekeeping directors, and facilities managers outsourcing cleaning contracts.

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
