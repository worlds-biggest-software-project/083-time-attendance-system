# Time & Attendance System — Feature & Functionality Survey

> Candidate #83 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| UKG (Kronos) | Commercial SaaS | Proprietary; subscription | https://www.ukg.com/solutions/time-and-attendance-software |
| Deputy | Commercial SaaS | Proprietary; per-user/month | https://www.deputy.com/ |
| Replicon | Commercial SaaS | Proprietary; per-user/month | https://www.replicon.com/ |
| Ceridian Dayforce | Commercial SaaS | Proprietary; custom enterprise | https://www.dayforce.com/ |
| Factorial | Commercial SaaS | Proprietary; per-user/month | https://factorialhr.com/ |
| Jibble | Commercial SaaS (freemium) | Proprietary; freemium | https://www.jibble.io/ |
| TimeTrex | Open Source / Commercial | AGPL-3.0 (Community); proprietary Enterprise | https://www.timetrex.com/ |
| Homebase | Commercial SaaS (freemium) | Proprietary; per-location/month | https://joinhomebase.com/ |

## Feature Analysis by Solution

### UKG (Kronos)

**Core features**
- Multi-method time capture: badge readers, biometric terminals (InTouch DX — fingerprint and facial recognition), mobile app, web browser, and kiosk modes
- Predictive scheduling: AI-assisted schedule generation based on demand forecasting, employee availability, and coverage rules
- Automated overtime and premium pay calculation: FLSA-compliant regular, overtime, and double-time calculations with configurable pay rules by jurisdiction
- Leave and absence management: FMLA tracking, intermittent leave coding, protected absence management, and attendance occurrence tracking
- Multi-location and multi-site support: location-specific pay rules, accrual policies, and scheduling templates
- Real-time compliance alerting: break violation warnings, approaching overtime thresholds, and attestation workflows

**Differentiating features**
- Deepest compliance engine in market: pre-built rules for FLSA, EU Working Time Directive, Canadian provincial law, and 100+ local jurisdictions
- Healthcare-specific scheduling: patient census-based staffing, credential-matched scheduling, and agency staff integration
- Manufacturing-specific features: job costing, gang scheduling, and shift differential automation
- UKG Dimensions (formerly Kronos Workforce Dimensions): AI-powered demand forecasting for scheduling optimisation

**UX patterns**
- Manager-facing scheduling grid with drag-and-drop shift assignment
- Employee self-service portal and mobile app for clock-in, shift swaps, and time-off requests
- Supervisor dashboard with real-time attendance visibility and exception-based alerts
- Complex configuration UI requiring specialist administrators; not self-service

**Integration points**
- Native integration with UKG Pro (HRIS) and UKG Ready (SMB HRIS)
- ADP, Workday, SAP SuccessFactors, Oracle HCM via certified connectors
- Payroll export in standard formats; 200+ payroll system integrations
- Hardware ecosystem: InTouch DX biometric terminals and badge readers

**Known gaps**
- Expensive: $25–$41/employee/month, making it inaccessible for small employers
- Complex implementation: 6–12 month typical deployment timeline
- AI scheduling is present but not the platform's primary strength compared to Deputy
- Limited natural language interaction; predominantly form-based UI

**Licence / IP notes**
- Fully proprietary; UKG formed via Ultimate Software + Kronos merger (2020)
- Biometric hardware IP held by UKG; proprietary demand forecasting algorithms
- No OSS components

---

### Deputy

**Core features**
- Shift scheduling: visual schedule builder with drag-and-drop, template-based shift creation, and auto-scheduling based on availability and role requirements
- Time and attendance: mobile clock-in with GPS geofencing, kiosk mode, and facial verification on clock-in
- Timesheet approvals: manager approval workflow with automatic flagging of unauthorised overtime and missed breaks
- Team communication: in-app messaging, shift handover notes, and announcement broadcasts
- Demand-based scheduling: integration with POS systems and sales data to schedule against expected volume
- Leave management: employee-initiated time-off requests integrated into the scheduling view

**Differentiating features**
- Best scheduling UX in market: deputy is scheduling-first, with a clean visual interface that frontline managers can use without training
- Fastest deployment: organisations typically go live in days, not months
- Award interpretation engine: Australian-specific pre-built rules for modern award compliance (deputy's heritage market)

**UX patterns**
- Mobile-first design: both manager scheduling and employee clock-in are optimised for smartphones
- Shift swap marketplace: employees can offer and pick up shifts via mobile app; manager approval optional
- Auto-scheduling assistant: AI suggests optimal schedule based on availability, qualifications, and demand

**Integration points**
- Payroll: Xero, QuickBooks, ADP, Gusto, and 50+ payroll systems via pre-built connectors
- POS: Square, Lightspeed, Shopify, Toast for demand-based scheduling
- HRIS: BambooHR, Workday (limited), and custom API
- iCalendar (RFC 5545) export for employee calendar apps

**Known gaps**
- Limited compliance depth for complex US jurisdictions: FMLA tracking, complex premium pay rules, and DCAA compliance are not fully supported
- Payroll integration is export-only; no native payroll processing
- No biometric hardware of its own; relies on facial recognition via existing device camera
- Enterprise scalability: suited to 20–2,000 employees; not designed for 10,000+ employee complexity

**Licence / IP notes**
- Proprietary SaaS; raised $81M Series C (2021); 340,000+ workplaces
- No OSS components; auto-scheduling algorithm is proprietary

---

### Replicon

**Core features**
- Flexible time models: project-based time tracking, shift-based attendance, and hybrid models in a single platform
- Policy-driven time rules: configurable overtime, premium pay, and accrual rules per department, location, and employee class
- Compliance documentation: audit-ready timesheets with approval timestamps, change history, and electronic signatures
- Project time allocation: time entries linked to project codes, activities, and cost centres for professional services billing
- Global time zone management: distributed team time tracking across multiple time zones with local-time display
- DCAA-compliant timekeeping: contemporaneous time recording with supervisor approval required for government contractors

**Differentiating features**
- Best fit for mixed time models: only platform that handles project billing, shift attendance, and salaried exception tracking equally well in one system
- Compliance documentation depth: full audit trail for every time entry change — designed for regulated industries and government contracting
- Polaris AI assistant: natural language time entry and query ("Log 3 hours to the Acme project yesterday")

**UX patterns**
- Web-first interface with mobile app; browser-based time entry familiar to office workers
- Manager exception view: only surfaces time entries that require attention
- Configurable approval workflow: multi-level approval with delegation and escalation rules

**Integration points**
- Payroll: ADP, Ceridian, Paychex, and custom exports
- ERP: SAP, Oracle, QuickBooks, NetSuite for project cost allocation
- HRIS: standard HR Open Standards API and custom connectors
- Project management: Jira, ServiceNow for project time cross-reference

**Known gaps**
- Scheduling depth is significantly behind Deputy and UKG; not designed for shift-intensive workplaces
- Mobile clock-in experience less polished than Deputy
- Biometric integration requires third-party hardware with custom integration

**Licence / IP notes**
- Proprietary SaaS; private company; no disclosed funding or valuation
- No OSS components

---

### TimeTrex (Open Source)

**Core features**
- Core time and attendance: web-based punch clock, mobile app, GPS tracking, and kiosk mode
- AI-powered facial recognition: prevents buddy punching using on-device facial recognition deployed on standard Android tablets and smartphones — no proprietary hardware required
- Scheduling: shift scheduling with conflict detection, availability management, and schedule templates
- Payroll engine: integrated payroll processing with tax calculations for US and Canada
- HR management: employee records, document management, performance reviews, and recruitment
- Overtime automation: configurable overtime rules with FLSA-compliant calculation logic
- Open-source codebase: full source code access for customisation and on-premises deployment

**Differentiating features**
- Only credible open-source workforce management platform with biometric features
- Software-defined biometrics: facial recognition on commodity hardware (Android tablet) eliminates $500–$2,000 per-terminal hardware cost of proprietary biometric clocks
- Self-hosted option: complete data sovereignty with no vendor data access required
- Integrated payroll: most T&A tools require a separate payroll system; TimeTrex processes payroll natively

**UX patterns**
- Functional but dated web interface; not consumer-grade
- Employee self-service portal for time-off requests, schedule viewing, and timesheet review
- Mobile app for clock-in with GPS and facial recognition

**Integration points**
- QuickBooks, Sage, and standard CSV/XML payroll exports
- REST API for custom integrations; open-source codebase enables direct database integration
- iCalendar export for schedule publishing

**Known gaps**
- UI significantly less polished than commercial alternatives; may create adoption friction
- Limited AI scheduling optimisation beyond conflict detection
- Small commercial ecosystem; limited partner network for implementation support
- Cloud-managed option exists but is not the primary product; self-hosting requires DevOps resources
- FLSA compliance rules are present but require careful manual configuration; not pre-certified

**Licence / IP notes**
- Community Edition: AGPL-3.0 licence — any modifications to server-side code that are deployed must be open-sourced
- Enterprise Edition: proprietary commercial licence (contact for pricing)
- AGPL copyleft requirement is relevant: organisations integrating TimeTrex into a hosted SaaS product must release their modifications under AGPL unless they purchase the Enterprise licence
- No known patent claims; facial recognition implementation uses open-source libraries (OpenCV-based)

---

### Jibble

**Core features**
- Time tracking: web, mobile, Slack, and MS Teams clock-in options
- GPS geofencing: location-based clock-in restriction to authorised work locations
- Facial recognition: on-device photo verification on clock-in to prevent buddy punching
- Project and activity tracking: time entries tagged to projects, clients, and activities
- Global time zone support: distributed team tracking across all time zones
- Basic attendance reports: hours worked, late arrivals, early departures, and absent days

**Differentiating features**
- Most generous free tier in market: unlimited users on the free plan with core features included
- Communication tool integration: clock-in via Slack or MS Teams without leaving the collaboration tool
- No hardware required: facial recognition via existing smartphone camera

**UX patterns**
- Clean mobile-first interface; minimal configuration required
- Employee-initiated clock-in with manager visibility dashboard
- Simple timesheet reports suitable for small teams

**Integration points**
- Slack, MS Teams for in-app clock-in
- Gusto, QuickBooks, and Xero for payroll export
- Zapier and REST API for custom integrations

**Known gaps**
- Limited scheduling functionality; scheduling is not a primary feature
- Compliance automation is minimal: no FMLA tracking, limited overtime rules, no DCAA support
- Not suitable for complex multi-site, multi-jurisdiction, or enterprise deployments
- Analytics depth is basic; no predictive or AI-driven insights beyond simple reports

**Licence / IP notes**
- Proprietary SaaS; free tier is a commercial freemium model
- No OSS components

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Clock-in methods: web, mobile, and kiosk at minimum; biometric optional
- FLSA-compliant overtime calculation (1.5x for hours over 40/week; configurable for other rules)
- Manager timesheet approval workflow with audit trail
- Employee self-service: view schedule, request time off, review own timesheets
- Payroll export in standard formats (CSV, direct connector to major payroll systems)
- Basic attendance reporting: hours worked, absences, late arrivals

### Differentiating Features
- AI-optimised shift scheduling using demand forecasting — UKG Dimensions, Deputy auto-scheduling
- Real-time compliance alerting before violations occur — UKG
- Biometric buddy-punching prevention on commodity hardware — TimeTrex, Jibble
- Project-linked time tracking for professional services billing — Replicon
- Award/collective agreement interpretation engine — Deputy (Australian market)
- DCAA-compliant timekeeping for government contractors — Replicon
- Continuous payroll recalculation on each clock event — Ceridian Dayforce

### Underserved Areas / Opportunities
- **Open-source with modern UX**: TimeTrex is the only OSS option but has a dated interface; a modern OSS T&A tool would immediately capture adoption
- **SMB compliance automation**: SMBs with 5–200 employees cannot afford UKG ($25–$41 PEPM) but face the same FLSA and state-level compliance obligations; no affordable tool covers this gap fully
- **Natural language schedule management**: no tool currently accepts plain-language scheduling instructions ("Cover Tuesday night shift with 3 certified staff, avoid overtime")
- **Biometric data sovereignty**: no SaaS vendor can fully satisfy regulated-industry requirements for on-premises biometric data processing; OSS self-hosted biometrics addresses this
- **Gig and hybrid workforce**: mixed employee/contractor time tracking in a single system is poorly served by current tools

### AI-Augmentation Candidates
- Demand-based schedule optimisation: AI generates schedules balancing coverage, cost, employee preferences, and compliance constraints simultaneously
- Proactive compliance alerting: flag approaching overtime, missed rest periods, and scheduling violations before they become violations
- Anomaly detection: identify buddy-punching patterns, geofencing violations, and unusual attendance patterns automatically
- Natural language schedule management for managers
- Predictive absence forecasting: anticipate high-absence periods from historical patterns to adjust coverage proactively

---

## Legal & IP Summary

- TimeTrex Community Edition is licensed under AGPL-3.0; any server-side modifications deployed as a hosted service must be released under AGPL unless the Enterprise commercial licence is purchased. This is a material constraint for any SaaS product built on top of TimeTrex.
- Biometric data (facial geometry, fingerprints) is classified as sensitive personal data under GDPR Article 9, Illinois BIPA, Texas CUBI, and Washington My Health MY Data Act; explicit consent, data minimisation, and on-premises processing requirements are stricter than for standard HR data.
- UKG and Deputy hold no publicly known patents on core scheduling or time-capture algorithms; these are established software patterns freely implementable.
- FLSA overtime calculation logic is specified in U.S. federal regulation (29 CFR Part 778) and is freely implementable; correct calculation is a compliance requirement, not an IP question.
- iCalendar (RFC 5545) and HR Open Standards data formats are open standards with no IP constraints.
- Facial recognition algorithms based on OpenCV and open-source embedding models (e.g., FaceNet, DeepFace) are freely available; any OSS T&A system should use open-source biometric libraries and disclose the specific model used for auditability.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Clock-in via web browser, mobile app, and software-based kiosk (using existing tablet hardware)
- GPS geofencing: restrict clock-in to authorised locations
- FLSA-compliant overtime calculation with configurable pay rules (daily, weekly, and California double-time rules)
- Manager timesheet approval workflow with full audit trail (change history with timestamps and approver identity)
- Employee self-service: view schedule, request time off, review and dispute own timesheets
- Payroll export: CSV and direct integration with QuickBooks and Xero at minimum
- Self-hosted deployment option for biometric data sovereignty

**Should-have (v1.1)**
- Facial recognition clock-in on commodity Android/iOS hardware (open-source embedding model; on-device processing only)
- FMLA and state-level leave tracking with protected absence coding
- Shift scheduling with availability management, conflict detection, and coverage rules
- EU Working Time Directive compliance rules (48-hour maximum, rest period enforcement)
- Natural language schedule requests for managers
- AI-based demand forecasting for scheduling using historical attendance patterns

**Nice-to-have (backlog)**
- DCAA-compliant timekeeping mode for government contractors
- Healthcare-specific scheduling: credential matching, patient census integration
- Project-linked time tracking for professional services billing
- Award interpretation engine for Australian modern awards and similar collective agreements
- Predictive absence forecasting dashboard
- Hardware integrations: standard NFC/RFID badge readers for organisations preferring hardware-based clock-in
