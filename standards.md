# Standards & API Reference

> Project: Cleaning Services Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- The foundational quality management standard. Cleaning software must support structured documentation of service workflows, scheduling, and performance measurement aligned with the quality processes ISO 9001 requires of cleaning organisations seeking certification.

**ISO 14001:2015 — Environmental Management Systems**
- URL: https://www.iso.org/standard/60857.html
- Sets requirements for managing environmental impacts. Relevant to cleaning companies that track cleaning product usage, waste disposal, and chemical inventory — a platform supporting green cleaning compliance must accommodate environmental data collection aligned with ISO 14001.

**ISO 45001:2018 — Occupational Health and Safety Management**
- URL: https://www.iso.org/standard/63787.html
- Global standard for workplace safety management. Cleaning companies handle hazardous chemicals and physical environments; software should support incident logging, hazard documentation, and safety-briefing records as evidence of ISO 45001 compliance.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- Required for any SaaS platform handling personal data (cleaner schedules, client addresses, health-facility cleaning logs). Governs data classification, access control, and breach response — essential for enterprise and healthcare-sector customers.

**ISO/IEC 20000-1:2018 — IT Service Management**
- URL: https://www.iso.org/standard/70636.html
- Relevant for software providers delivering cleaning management as a managed service. Covers service delivery processes, SLAs, and continuous improvement — applicable when the platform is operated as a white-label managed service for large facilities.

---

### W3C & IETF Standards

**RFC 5545 — iCalendar (Internet Calendaring and Scheduling Core Object Specification)**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- Defines the `.ics` format for representing calendaring events, recurring schedules, and to-dos. A cleaning platform exporting or importing job schedules to calendar apps (Google Calendar, Outlook, Apple Calendar) must conform to RFC 5545 for interoperability.

**RFC 5546 — iTIP (iCalendar Transport-Independent Interoperability Protocol)**
- URL: https://datatracker.ietf.org/doc/html/rfc5546
- Defines how scheduling messages (meeting requests, updates, cancellations) are communicated between calendar systems. Relevant for client appointment confirmations and cleaner shift notifications that integrate with calendar clients.

**RFC 4791 — CalDAV (Calendaring Extensions to WebDAV)**
- URL: https://datatracker.ietf.org/doc/html/rfc4791
- Enables server-side calendar storage and sharing via HTTP. Relevant if the platform offers a CalDAV endpoint for clients or cleaners to sync schedules directly to their calendar applications.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines standard HTTP methods, status codes, and content negotiation. All REST API endpoints in the platform must conform to this specification for predictable, standards-compliant client behaviour.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard for delegated API access. All third-party integrations (QuickBooks, Stripe, Google) and the platform's own public API must implement OAuth 2.0 for secure token-based authentication.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0 — supports user login via Google, Apple, or enterprise SSO. Required for GDPR and CCPA alignment and for enterprise customers with single sign-on requirements.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- Machine-readable standard for describing REST APIs. All public platform endpoints should be documented with an OAS 3.1 specification, enabling auto-generated SDKs, interactive documentation, and compatibility testing.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Vocabulary for annotating and validating JSON documents. Use for request/response validation on all API endpoints (job, client, invoice, inspection data models).

**GS1 Global Location Number (GLN)**
- URL: https://www.gs1.org/standards/id-keys/gln
- Used in commercial and healthcare cleaning contexts to identify facilities and rooms uniquely. Enterprise and hospital cleaning contracts increasingly use GLN for site identification — relevant for multi-location commercial cleaning contracts.

**WS-Calendar (OASIS)**
- URL: https://docs.oasis-open.org/ws-calendar/ws-calendar/v1.0/
- OASIS standard for passing schedule and interval information between services. Relevant if the platform exposes scheduling data to enterprise facility management or building management systems.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) + PKCE (RFC 7636)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- PKCE (Proof Key for Code Exchange) extends OAuth 2.0 for mobile and single-page app security. Required for the cleaner mobile app and any public-facing client booking flows using OAuth.

**OWASP Application Security Verification Standard (ASVS)**
- URL: https://owasp.org/www-project-application-security-verification-standard/
- Framework for web application security requirements. The platform should meet ASVS Level 2 at minimum, covering authentication, session management, access control, and data validation — particularly important given customer PII and home address data.

**OWASP Top 10**
- URL: https://owasp.org/www-project-top-ten/
- The ten most critical web application security risks. All API endpoints, booking forms, and client portals must be audited against OWASP Top 10 before production deployment.

**NIST SP 800-63B — Digital Identity Guidelines (Authentication)**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- US federal guidance on password policies, MFA, and credential recovery. Relevant for enterprise and government-facility cleaning contracts that mandate NIST-compliant authentication.

**GDPR (EU) 2016/679 — General Data Protection Regulation**
- URL: https://gdpr-info.eu/
- Governs personal data handling for EU clients, cleaners, and any EU-based operations. The platform must implement data minimisation, right-to-erasure, consent management, and data residency controls.

**CCPA — California Consumer Privacy Act**
- URL: https://oag.ca.gov/privacy/ccpa
- US (California) data privacy law covering consumer rights over personal data. Applies to any California-based customers or cleaners on the platform — requires opt-out of data sale, disclosure of data collected, and deletion on request.

**HIPAA (US) — Health Insurance Portability and Accountability Act**
- URL: https://www.hhs.gov/hipaa/
- Applies to cleaning service providers operating in hospitals, clinics, and healthcare facilities. The platform must support HIPAA-compliant data handling for client-side cleaning records, room-level cleaning logs, and PHI-adjacent location data.

---

### Domain-Specific Standards

**ISSA CIMS — Cleaning Industry Management Standard**
- URL: https://cims.issa.com/
- The global benchmark for operational excellence in commercial cleaning organisations. Software supporting CIMS-certified operators must enable documentation of quality systems, customer focus procedures, and environmental stewardship per CIMS requirements. CIMS certification requires 100% of mandatory elements and 60% of recommended elements per section.

**ISSA CIMS-GB — Green Building Addendum**
- URL: https://cims.issa.com/cims-standard-overview/
- Extension of CIMS covering green cleaning practices. Cleaning software supporting LEED-certified buildings must track product compliance with EPA Safer Choice, LEED green cleaning requirements, and CIMS-GB criteria.

**OSHA HazCom Standard (29 CFR 1910.1200)**
- URL: https://www.osha.gov/hazcom
- Requires Safety Data Sheets (SDS) for all cleaning chemicals in the workplace. A platform serving commercial cleaning operators must support SDS document management, chemical inventory tracking, and worker hazard communication as required by HazCom.

**EPA Safer Choice Programme**
- URL: https://www.epa.gov/saferchoice
- Voluntary certification for cleaning products meeting environmental and safety standards. Increasingly required in institutional and government cleaning contracts — software must be able to flag products that do not carry EPA Safer Choice certification.

**LEED v4.1 — Cleaning and Maintenance Credit (EQp3)**
- URL: https://www.usgbc.org/credits/healthcare/v4/eqp3
- LEED building certification includes requirements for cleaning products and procedures. Software used by building-service contractors in LEED-certified facilities must support green product logging and cleaning protocol documentation.

---

## Similar Products — Developer Documentation & APIs

### Jobber
- **Description:** General field service management platform widely used by cleaning businesses for scheduling, invoicing, client portal, and recurring job management.
- **API Documentation:** https://developer.getjobber.com/docs/
- **SDKs/Libraries:** Ruby on Rails API Template (https://github.com/GetJobber/Jobber-AppTemplate-RailsAPI); React App Template available via Developer Center
- **Developer Guide:** https://developer.getjobber.com/docs/building_your_app/app_authorization/
- **Standards:** GraphQL API (application/json); OAuth 2.0 for app authorisation; webhook at-least-once delivery with idempotency handling
- **Authentication:** OAuth 2.0 (Authorization Code Flow)

### Housecall Pro
- **Description:** Home-services FSM used by 45,000+ professionals for scheduling, dispatch, marketing automation, and online booking.
- **API Documentation:** https://docs.housecallpro.com/docs/housecall-public-api
- **SDKs/Libraries:** No official SDK; Zapier, Make, and Pabbly Connect no-code connectors available
- **Developer Guide:** https://help.housecallpro.com/en/articles/8505035-api-overview
- **Standards:** REST/JSON; webhook event subscriptions with signing secret
- **Authentication:** API key (MAX plan only); OAuth 2.0 not documented

### ServiceM8
- **Description:** iOS-first job management platform for small cleaning and trade businesses with rich quoting, checklists, and asset management.
- **API Documentation:** https://developer.servicem8.com/
- **SDKs/Libraries:** No official SDK; REST endpoints consumable from any HTTP client
- **Developer Guide:** https://developer.servicem8.com/docs/
- **Standards:** REST/JSON; OpenAPI-compatible endpoints; webhook event delivery
- **Authentication:** OAuth 2.0

### Stripe (Payments)
- **Description:** Payment processing platform used by most cleaning software for card payments, recurring billing, and marketplace payouts (Stripe Connect).
- **API Documentation:** https://docs.stripe.com/
- **SDKs/Libraries:** Official SDKs for Python, Ruby, Node.js, Java, Go, PHP, .NET (https://docs.stripe.com/libraries)
- **Developer Guide:** https://docs.stripe.com/connect/marketplace (for multi-vendor/marketplace use cases)
- **Standards:** REST/JSON; OpenAPI 3.0 specification available; webhooks with HMAC-SHA256 signing; idempotency keys
- **Authentication:** API Key (Bearer token); OAuth 2.0 for Connect account onboarding

### QuickBooks Online (Intuit)
- **Description:** Accounting platform integrated by virtually all cleaning software for invoicing, payroll, and financial reporting synchronisation.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/develop
- **SDKs/Libraries:** Official SDKs for Java, PHP, Python, Ruby, Node.js (https://developer.intuit.com/app/developer/qbo/docs/develop/sdks-and-libraries)
- **Developer Guide:** https://developer.intuit.com/app/developer/qbo/docs/get-started
- **Standards:** REST/JSON; webhook event notifications (real-time data change alerts); OpenID Connect for user authentication
- **Authentication:** OAuth 2.0 (Authorization Code flow); refresh token management required

### Google Maps Platform (Routes API / Route Optimization API)
- **Description:** Mapping and routing services used for cleaner dispatch, travel time estimation, and route optimisation across daily job stops.
- **API Documentation:** https://developers.google.com/maps/documentation/routes
- **SDKs/Libraries:** Google Maps JavaScript SDK, Python client library, Node.js client library (https://developers.google.com/maps/documentation/javascript)
- **Developer Guide:** https://developers.google.com/maps/documentation/route-optimization/overview
- **Standards:** REST/JSON; gRPC supported for high-throughput route matrix calls
- **Authentication:** API Key; service account for server-side calls

### Twilio (SMS & Voice)
- **Description:** Programmable communications platform used by cleaning software (including Fixlify's AI phone agent) for SMS reminders, appointment notifications, and AI voice interactions.
- **API Documentation:** https://www.twilio.com/docs
- **SDKs/Libraries:** Official SDKs for Python, Node.js, Ruby, Java, C#, PHP, Go (https://www.twilio.com/docs/libraries)
- **Developer Guide:** https://www.twilio.com/docs/messaging/quickstart
- **Standards:** REST/JSON; webhook callbacks for inbound messages and call events; TwiML (XML) for call flow logic
- **Authentication:** API Key + Secret; account SID/auth token

### SendGrid / Twilio Email API
- **Description:** Transactional email service used for booking confirmations, invoices, inspection reports, and automated marketing campaigns in cleaning platforms.
- **API Documentation:** https://docs.sendgrid.com/api-reference
- **SDKs/Libraries:** Official libraries for Python, Node.js, Ruby, Java, C#, PHP, Go
- **Developer Guide:** https://docs.sendgrid.com/for-developers/sending-email/api-getting-started
- **Standards:** REST/JSON; SMTP relay option; webhook event tracking; DKIM/SPF/DMARC email authentication standards
- **Authentication:** API Key (Bearer token)

---

## Notes

**Emerging standards to monitor:**
- **MCP (Model Context Protocol)** — Anthropic's open standard for AI agents to interface with tools and data. As AI scheduling assistants, inspection agents, and phone bots become core platform features, MCP provides a standard interface for connecting AI models to job data, scheduling engines, and client records. See https://modelcontextprotocol.io/
- **AsyncAPI 2.x / 3.0** — Complements OpenAPI for event-driven and webhook architectures. As the platform adds real-time events (GPS check-in, inspection submitted, payment received), AsyncAPI documents the async message contracts. See https://www.asyncapi.com/
- **OWASP Mobile Application Security Verification Standard (MASVS)** — As the cleaner mobile app handles location data, photos, and authentication, MASVS provides the security baseline for mobile app hardening. See https://mas.owasp.org/MASVS/

**Areas with limited standardisation:**
- No published open standard exists for cleaning-specific data models (jobs, inspection results, chemical inventories) — any open-source platform has an opportunity to define and publish a JSON Schema-based canonical data model for the industry.
- CIMS compliance tracking is well-defined as a certification framework but lacks a digital data interchange format — software vendors currently implement compliance reporting in proprietary ways.
