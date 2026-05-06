# Cleaning Services Platform — Feature & Functionality Survey

> Candidate #240 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| ZenMaid | Residential maid-service scheduling & operations | Commercial SaaS — from $19/month + $4/seat | https://get.zenmaid.com/ |
| Swept | Commercial cleaning workforce management | Commercial SaaS — contact for quote | https://sweptworks.com/ |
| Jobber | General field service management | Commercial SaaS — from $29/user/month | https://www.getjobber.com/ |
| Housecall Pro | Home-services FSM with marketing tools | Commercial SaaS — from $59/month | https://www.housecallpro.com/ |
| Booking Koala | Client booking & management for cleaning | Commercial SaaS — from $27/month | https://www.bookingkoala.com/ |
| GorillaDesk | Field service (pest, lawn, cleaning) | Commercial SaaS — from $49/month | https://gorilladesk.com/ |
| ServiceM8 | Job management — iOS-first for small businesses | Commercial SaaS — from $29/month | https://www.servicem8.com/ |
| Fixlify | AI-powered FSM for small cleaning businesses | Commercial SaaS — Free → Pro $49/mo → Business $99/mo | https://fixlify.app/ |
| Janitorial Manager | Commercial cleaning operations & inspections | Commercial SaaS — contact for quote | https://www.janitorialmanager.com/ |
| OrangeQC | Commercial cleaning quality control & inspections | Commercial SaaS — contact for quote | https://www.orangeqc.com/ |

---

## Feature Analysis by Solution

### ZenMaid

**Core features**
- Drag-and-drop scheduling calendar with multiple views (weekly grid, daily list, team availability)
- Client self-service booking form embeddable on business website
- Automated email and SMS reminders to clients and cleaners
- Invoicing with auto-generate-on-job-completion and QuickBooks sync
- GPS clock-in/clock-out for cleaners
- Customisable reporting (hours worked, revenue, booking sources)
- Recurring job scheduling and management
- Client management and contact database

**Differentiating features**
- Built exclusively for residential maid-service businesses — every workflow assumes a domestic cleaning context
- Booking forms embed directly into any website, converting visitors to bookings without a phone call
- Automation-first: reminders, follow-ups, and review requests fire automatically without manual intervention

**UX patterns**
- Simple, opinionated setup — limited configuration overhead suits first-time software buyers
- Calendar-centric interface with jobs visible at a glance
- Mobile app supports cleaners but iOS/Android apps are secondary to the admin web portal

**Integration points**
- QuickBooks Online (accounting and payroll export)
- Zapier (event triggers and actions for bookings, invoices, clients, appointments)
- Embedded booking forms via iframe or link

**Known gaps**
- No commercial cleaning features — no site-level contract tracking, no multi-crew zone coordination
- No route optimisation for dispatching multiple cleaners across a city
- No quality inspection or photo submission workflow
- QuickBooks integration flagged as incomplete as of early 2026
- No geofencing — cleaners can clock in from anywhere
- Limited AI or intelligent automation beyond templated messaging

**Licence / IP notes**
- Proprietary SaaS; no open-source components; no public API documented

---

### Swept

**Core features**
- Shift scheduling and daily task assignment for commercial cleaning sites
- Cleaner mobile app with shift instructions in 100+ languages (auto-translated)
- Quality control inspections with customisable templates (Exceeds/Meets/Doesn't Meet ratings)
- Photo capture and annotation on inspections
- PDF inspection reports shareable with clients
- GPS check-in/check-out at each location
- In-app messaging between cleaners, supervisors, and managers
- Cleaner retention analytics (customers retain cleaners 91% better than industry average per Swept claims)

**Differentiating features**
- 100+ language translation in the cleaner app — critical for immigrant-heavy cleaning workforces
- Reusable inspection templates per location with photo evidence and client-facing PDF reports
- Purpose-built for commercial cleaning, not adapted from residential or general FSM

**UX patterns**
- Cleaner-facing app designed for minimal training — icon-based navigation
- Admin interface focused on location/contract management rather than individual job booking
- Progressive disclosure: inspectors see only what they need per location

**Integration points**
- Zapier for workflow automations
- Limited direct API — primarily a closed platform

**Known gaps**
- Not suited for residential or mixed residential/commercial operations
- Limited quoting, estimating, or new-client acquisition workflows
- No online booking or instant-quote capability
- Financial/billing features are basic compared to general FSM competitors

**Licence / IP notes**
- Proprietary SaaS; limited public API access

---

### Jobber

**Core features**
- Job scheduling, quoting, and invoicing for recurring and one-off work
- Client Hub — online portal for clients to approve quotes, view appointments, pay invoices
- Automated payment reminders and recurring billing with saved cards
- Mobile app (iOS and Android) for field crews
- QuickBooks Online and Stripe integration
- 1,500+ Zapier-connected app automations
- AI-powered scheduling assistant (launched 2026) with skill matching, location, and historical performance data
- Offline mode for field crews (launched January 2026)
- Time tracking, job forms, and work order management

**Differentiating features**
- Mature, richly featured platform used across many field service industries — large ecosystem
- Client Hub provides a polished self-service experience for end clients
- AI scheduling assistant (2026) reduces scheduling conflicts with optimal assignment suggestions

**UX patterns**
- Higher complexity than cleaning-only tools — suited for businesses with diverse service types
- Strong recurring job and subscription-billing workflows
- Role-based access: admin, field worker, client — each sees an appropriate interface

**Integration points**
- QuickBooks Online (native, two-way sync)
- Stripe (payment processing)
- Zapier (1,500+ connected apps)
- GraphQL API with webhooks (OAuth 2.0) — developer.getjobber.com
- Ruby on Rails and React app templates for third-party developers

**Known gaps**
- Quality inspection and photo-based proof-of-service are not native — rely on form add-ons
- Not purpose-built for cleaning — lacks cleaning-specific nomenclature and workflow defaults
- Per-user pricing becomes expensive quickly for larger teams

**Licence / IP notes**
- Proprietary SaaS; public GraphQL API available; no open-source core

---

### Housecall Pro

**Core features**
- Online booking via website, Google, and social platforms
- Scheduling, dispatch, and job management
- Automated SMS and email campaigns (marketing and follow-up)
- Review management — auto-request reviews on Google and Facebook, centralised response
- Invoicing with online payment links (card, ACH, digital wallets)
- GPS technician tracking
- QuickBooks, Google, and Thumbtack integrations
- AI-powered email composition for client outreach

**Differentiating features**
- Strongest marketing automation in the category — review management and campaign tools go beyond scheduling
- Booking directly from Google search results (Google Business Profile integration)
- Used by 45,000+ home service professionals — large user community

**UX patterns**
- Consumer-like onboarding, polished interface, strong mobile experience
- Built for owner-operators who want marketing and operations in one place
- Automations are set-and-forget once configured

**Integration points**
- QuickBooks, Google, Thumbtack (native)
- Zapier, Make, Pabbly Connect
- REST API + webhooks (MAX plan only) — docs.housecallpro.com

**Known gaps**
- API and webhooks locked behind the highest pricing tier (MAX plan only)
- Limited depth for commercial cleaning (no site-level contracts, no CIMS compliance tracking)
- General FSM; lacks cleaning-specific quality management features
- No multi-language cleaner support

**Licence / IP notes**
- Proprietary SaaS; API available on MAX plan only; no open-source components

---

### Booking Koala

**Core features**
- Customisable online booking forms and website builder
- Customer-facing dashboard for managing appointments, referrals, and gift cards
- Provider portal for cleaners to view schedules, payments, tips, and ratings
- Built-in coupon and referral management
- Multi-location support
- SMS and email reminders
- Revenue and payroll reporting
- Cleaner mobile app for viewing jobs

**Differentiating features**
- Full website builder included — operators can build a complete business presence from inside the platform
- Customer referral and gift card features built in at all price points ($27/month entry)
- Provider-facing dashboards give cleaners visibility into their own earnings and performance

**UX patterns**
- Feature-rich but complex initial setup — steeper learning curve than ZenMaid
- Consumer-facing booking flow is polished and conversion-optimised
- Designed for cleaning businesses that want control over branding and client acquisition

**Integration points**
- Zapier integrations
- Limited API documentation publicly available

**Known gaps**
- Operational depth (dispatch, route optimisation, quality inspections) is lighter than competitors
- Commercial cleaning features are minimal
- Setup complexity creates friction for first-time buyers

**Licence / IP notes**
- Proprietary SaaS; no public API; no open-source components

---

### GorillaDesk

**Core features**
- Job scheduling and recurring-route management
- Route optimisation for reducing mileage across daily stops
- Customer portal for viewing quotes, appointments, and paying invoices
- Mobile app for field crews with job details, eSignatures, and on-site payments
- Recurring invoicing and automatic payment reminders
- 20+ built-in reports for profitability analysis
- Estimate creation and approval workflow

**Differentiating features**
- Cross-vertical — serves pest control, lawn care, pool, and cleaning in a single platform
- Route optimisation native to the platform rather than an add-on
- Affordable entry price ($49/month) relative to feature depth

**UX patterns**
- Operations-first interface — scheduling and route maps are primary views
- Customer portal reduces inbound calls for appointment and payment queries

**Integration points**
- QuickBooks, Stripe
- Zapier

**Known gaps**
- Not cleaning-specific — no quality checklists, photo inspection, or CIMS compliance
- Commercial cleaning contract management is absent
- No multi-language cleaner support
- Limited marketing automation

**Licence / IP notes**
- Proprietary SaaS; no public API; no open-source core

---

### ServiceM8

**Core features**
- Job cards with notes, photos, and video attached per job
- Quote-to-invoice pipeline with online client approval
- Proposals builder with rich text, images, optional/fixed materials
- Instant quote and online booking for clients
- Quality checklists for job requirements
- iOS and iPad-first field experience
- Asset management for tracked equipment
- Automated client communications (email/SMS)

**Differentiating features**
- iOS-first architecture — best-in-class iPad/iPhone experience for field workers
- Rich proposal builder with drag-and-drop, sections, and optional line items
- Strong for small businesses (under 30 staff) due to simplicity and iOS app quality

**UX patterns**
- Mobile-native design — the iOS app is the primary work surface, not the web portal
- Setup is fast; minimal configuration required to go live
- Designed for owner-operators working in the field themselves

**Integration points**
- QuickBooks, Xero (accounting)
- Stripe, Square (payments)
- Zapier
- REST API with OAuth 2.0 — servicem8.com/api

**Known gaps**
- Limited scalability beyond ~30 field workers
- No commercial cleaning contract management or CIMS compliance
- Android app is secondary — poor fit for Android-using cleaning teams
- No AI features or intelligent scheduling

**Licence / IP notes**
- Proprietary SaaS; REST API available; no open-source components

---

### Fixlify

**Core features**
- Recurring client scheduling and team management
- Automated billing and invoicing
- AI phone answering — captures and books new client enquiries outside business hours
- Client portal for bookings and communication
- Job and crew tracking
- Marketing automation features

**Differentiating features**
- AI phone agent that books new cleaning jobs at any hour — converts leads that would otherwise go to voicemail (reported 3x conversion improvement)
- Free tier available — lowest friction entry into AI-powered cleaning software

**UX patterns**
- Targeted at 1–30 employee cleaning businesses — onboarding is streamlined for that scale
- AI features are surfaced prominently rather than buried in settings

**Integration points**
- Standard payment and accounting integrations (details limited in public documentation)
- No public API documented

**Known gaps**
- Limited commercial cleaning depth
- Quality inspection and photo proof-of-service not documented
- Route optimisation not featured
- Newer entrant — feature set less mature than Jobber or Housecall Pro

**Licence / IP notes**
- Proprietary SaaS; no public API documented; newer platform with limited third-party reviews

---

### Janitorial Manager

**Core features**
- Janitorial inspection app with customisable templates per location
- Multiple grading types with image capture and cloud storage
- Work order management from inspections and client requests
- Employee timekeeping with multiple clock-in methods and location-specific reminders
- Scan4Clean QR code system — cleaners scan a room to see tasks; building occupants scan to see last cleaned time
- Equipment tracking and lifecycle management
- JM Connect mobile app for field team management
- Site-specific scheduling and shift reminders

**Differentiating features**
- Scan4Clean is a unique QR-based proof-of-service and transparency feature — building occupants can verify cleaning status
- Purpose-built for commercial/institutional cleaning with multi-site and multi-building management
- Equipment asset management integrated into the cleaning operations workflow

**UX patterns**
- Operations-manager-first interface — dashboards show real-time site activity
- Inspection templates are location-specific, reducing setup time for large portfolios
- Mobile app (JM Connect) is the cleaner-facing tool

**Integration points**
- Limited third-party integrations documented publicly
- Focus is on closed-loop operations within the platform

**Known gaps**
- Weak client acquisition, quoting, and billing compared to residential-focused tools
- Limited online booking or self-service for end clients
- No route optimisation for dispatching across a city
- API access not publicly documented

**Licence / IP notes**
- Proprietary SaaS; no public API documented

---

### OrangeQC

**Core features**
- Customisable mobile inspection forms (text fields, drop-downs, numerical inputs, signatures, photos with annotation)
- Offline inspection with sync when connectivity returns
- GPS-tagged, timestamped inspection records
- Automatic scoring with highest/lowest facility benchmarking
- Scheduled recurring inspections (daily, weekly, monthly, custom)
- Automated client and stakeholder report delivery (daily/weekly/monthly)
- Real-time email alerts for inspections and tickets
- Service checklists and employee visit tracking

**Differentiating features**
- Purpose-built for quality audit workflows — the deepest inspection and reporting toolkit in the category
- Automatic report delivery to client stakeholders builds transparency and client trust
- Offline capability with GPS/timestamp ensures field validity even in no-signal environments

**UX patterns**
- Inspector-first UX — forms are designed to be completed quickly on a phone or tablet in the field
- Management dashboards show quality performance across all sites at a glance
- Client-facing reporting reduces need for manual account management calls

**Integration points**
- Limited third-party integrations
- No public API documented

**Known gaps**
- Focused solely on inspections and quality — not a full FSM or scheduling platform
- No scheduling, dispatch, invoicing, or client acquisition features
- Requires integration with a separate scheduling/billing platform for complete operations

**Licence / IP notes**
- Proprietary SaaS; no public API documented

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Job scheduling with recurring appointment support
- Client-facing booking or communication interface (portal, form, or app)
- Invoicing and payment collection (card, ACH, or saved cards)
- Basic mobile app for field workers (iOS minimum; Android increasingly expected)
- QuickBooks or accounting software integration
- Automated SMS/email reminders for clients and cleaners

### Differentiating Features
- AI-powered scheduling with proximity, skill, and history optimisation (Jobber 2026, FieldCamp)
- Multi-language cleaner apps for non-English-speaking workforces (Swept — 100+ languages)
- AI phone answering for 24/7 lead capture and booking (Fixlify)
- GPS-tagged photo inspection with automated client report delivery (OrangeQC, Swept)
- QR-code proof-of-service transparency for building occupants (Janitorial Manager Scan4Clean)
- Dynamic quote-to-proposal pipeline with client online approval (ServiceM8)
- Built-in review management and Google booking integration (Housecall Pro)
- Stripe Connect-style multi-vendor payout capability for franchise or marketplace models

### Underserved Areas / Opportunities
- Intelligent route optimisation across a fleet of cleaners — most platforms leave dispatch manual
- Photo-based AI quality inspection — no platform analyses photos automatically to detect missed areas
- Real-time cleaner location sharing with clients (beyond simple GPS clock-in)
- Green cleaning product tracking for LEED, CIMS-GB, or EPA Safer Choice compliance
- Integrated SDS/chemical inventory management for OSHA compliance
- Franchise and multi-brand management under a single operator account
- Cleaner retention analytics — pattern detection for turnover risk proactive intervention
- Dynamic pricing engine integrating property size, frequency, and market data
- Cross-platform inspection + scheduling + billing in a single open-source stack
- API-first open platform allowing third-party apps to compose a best-of-breed stack

### AI-Augmentation Candidates
- Dispatch and route optimisation — replacing manual daily scheduling with ML-optimised routing
- Quality inspection — computer vision on before/after photos to detect incomplete cleaning
- Lead capture and booking — AI phone/chat agent converting inbound enquiries outside business hours
- Cleaner retention prediction — anomaly detection on shift patterns, hours, and payroll data
- Client communication drafting — personalised appointment reminders, post-service summaries, review requests
- Dynamic pricing — adjusting quotes in real time based on size, frequency, market, and historical duration
- Chemical and compliance tracking — flagging non-compliant product usage against LEED or EPA Safer Choice requirements

---

## Legal & IP Summary

No patents or copyright concerns were identified with the feature set. All reviewed products are proprietary SaaS offerings with no open-source cores. The Scan4Clean QR concept (Janitorial Manager) may represent a registered trademark but is not patent-protected in the publicly available literature. The multi-language translation feature in Swept leverages standard translation APIs (Google Translate or DeepL) — these are freely available. No licensing, patent, or IP barriers were identified for implementing any of the described features in a new open-source platform.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Recurring job scheduling with drag-and-drop calendar and mobile cleaner app
- Client self-service booking form with online payment at booking
- Automated SMS/email reminders and post-service follow-up
- Invoicing with recurring billing and saved-card charging
- Photo-based job documentation (before/after) with timestamped GPS check-in/out
- Basic quality checklist per job or location type

**Should-have (v1.1)**
- AI-assisted route optimisation for dispatching multiple cleaners
- Customisable inspection templates with client-facing PDF report delivery
- Multi-language cleaner app interface (at minimum English + Spanish)
- QuickBooks Online and Stripe native integrations
- Review request automation (Google, Facebook)
- Cleaner retention dashboard with workload and pattern analytics

**Nice-to-have (backlog)**
- AI phone agent for 24/7 new client lead capture and booking
- Computer-vision quality check on submitted photos (flag missed areas)
- Green product compliance tracker (LEED, EPA Safer Choice, CIMS-GB)
- Dynamic pricing engine with market-rate and historical-duration inputs
- Franchise / multi-brand management with consolidated reporting
- Public REST/GraphQL API with webhooks for third-party app integrations
