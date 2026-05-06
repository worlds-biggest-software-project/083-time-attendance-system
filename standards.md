# Standards & API Reference

> Project: Time & Attendance System · Generated: 2026-05-06

## Industry Standards & Specifications

### Regulatory & Labour Law Frameworks

**Fair Labor Standards Act (FLSA) — 29 CFR Part 778**
- URL: https://www.dol.gov/agencies/whd/overtime
- Governs U.S. overtime calculation (1.5x for hours over 40/week), minimum wage, and exempt/non-exempt employee classification. A T&A system operating in the U.S. must implement FLSA-compliant overtime logic with audit-ready records. FLSA compliance is a functional requirement, not an IP concern — the calculation rules are fully specified in federal regulation.

**Family and Medical Leave Act (FMLA)**
- URL: https://www.dol.gov/agencies/whd/fmla
- Requires tracking of intermittent leave, protected absence coding, and leave eligibility rules (12 weeks unpaid for qualifying reasons). T&A systems must integrate FMLA leave tracking with attendance occurrence logic to avoid inadvertent adverse action against protected absences.

**EU Working Time Directive — Directive 2003/88/EC**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=celex:32003L0088
- Mandates a maximum 48-hour working week, minimum daily and weekly rest periods, and paid annual leave across EU member states. A May 2019 European Court of Justice ruling (Case C-55/18) requires employers to implement objective, reliable, and accessible systems to record employees' daily working hours. A compliant T&A system must enforce rest period rules, alert on directive violations, and store records for the statutory retention period under each member state's implementing legislation.

**Defense Contract Audit Agency (DCAA) Timekeeping Requirements**
- URL: https://www.dcaa.mil
- Reference guide: https://hubstaff.com/time-tracking/dcaa-timekeeping-requirements
- Requires contemporaneous time recording by project and labour category, supervisor approval workflows, digital audit trails with immutable change history, and record retention of 3–6 years. Governed by FAR, DFARS, and the DCAA Contract Audit Manual (CAM). Relevant for any T&A system targeting U.S. government contractors.

---

### Biometric Data Standards

**ISO/IEC 19794-2:2011 — Biometric Data Interchange Formats — Finger Minutiae Data**
- URL: https://www.iso.org/standard/50864.html
- Specifies the data format for representation of fingerprints using minutiae for use in automated fingerprint recognition. Relevant when integrating with fingerprint-based hardware time clocks or when implementing biometric data portability between systems.

**ISO/IEC 19794-4:2011 — Biometric Data Interchange Formats — Finger Image Data**
- URL: https://www.iso.org/standard/50866.html
- Specifies a data record interchange format for storing, recording, and transmitting finger and palm image data. Applicable to systems that store raw fingerprint images rather than derived minutiae for re-matching.

**ISO/IEC 19794-5:2011 — Biometric Data Interchange Formats — Face Image Data**
- URL: https://www.iso.org/standard/50867.html
- Defines a standard scheme for codifying face image data within a CBEFF-compliant data structure for use in facial recognition systems. Directly relevant to software-based facial clock-in implementations where image data must be stored or shared with third-party matching services.

**ISO/IEC 19795-10:2024 — Biometric Performance Testing — Demographic Variation**
- URL: https://www.iso.org/standard/81223.html
- Specifies how to quantify biometric system performance variation across demographic groups (age, sex, ethnicity). Any T&A system using facial recognition must demonstrate equitable performance across workforce demographics to satisfy both technical and regulatory fairness requirements under GDPR and EEOC guidelines.

---

### Privacy & Biometric Compliance Legislation

**GDPR — Article 9 (Special Categories of Personal Data)**
- URL: https://gdpr.eu/article-9-processing-special-categories-of-personal-data/
- Classifies biometric data (facial geometry, fingerprints) as sensitive personal data requiring explicit consent, data minimisation, and a lawful basis for processing. On-premises or self-hosted biometric processing is the most defensible architecture under GDPR for T&A systems. Data retention periods must be defined and enforced; records must be deletable upon employee departure.

**Illinois Biometric Information Privacy Act (BIPA) — 740 ILCS 14**
- URL: https://law.justia.com/codes/illinois/chapter-740/act-740-ilcs-14/
- 2024 amendments limit damages from per-scan to per-person violations (max $1,000 per negligent violation, $5,000 per intentional/reckless violation). Requires written (including electronic) consent before collecting biometric identifiers, disclosure of data purpose and retention period, and a documented destruction schedule. Applies to any employer using fingerprint or facial recognition time clocks with Illinois-based employees.

**California Consumer Privacy Act / CPRA (CCPA)**
- URL: https://oag.ca.gov/privacy/ccpa
- Treats biometric data as sensitive personal information with opt-out rights, data access rights, and deletion rights. T&A systems serving California-based workforces must implement consent workflows and data subject request handling for biometric records.

---

### Calendaring & Scheduling Standards

**RFC 5545 — Internet Calendaring and Scheduling Core Object Specification (iCalendar)**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- Standard data format for representing and exchanging calendar events, to-dos, and free/busy information, independent of any calendar service or protocol. Widely used for shift schedule export to employee calendar applications (Google Calendar, Apple Calendar, Outlook). Should be a native export format in any T&A scheduling module.

**RFC 5546 — iCalendar Transport-Independent Interoperability Protocol (iTIP)**
- URL: https://datatracker.ietf.org/doc/html/rfc5546
- Defines how iCalendar objects are used to request, reply to, add, cancel, refresh, counter, and decline scheduling requests. Relevant for two-way shift swap and scheduling request workflows between a T&A system and employee calendar clients.

---

### Data Model & API Specifications

**HR Open Standards — HR-JSON Timecard Schema**
- URL: https://www.hropenstandards.org/standards
- Documentation: https://www.hropenstandards.org/documentationinformation
- The only open, vendor-neutral JSON schema standard specifically designed for HR data interchange, including a Timecard schema for time entry data exchange. Provides field definitions for time entries, labour categories, project codes, and approval status. Schemas are available for free download. Implementing the HR-JSON Timecard schema for payroll export enables interoperability with any compliant payroll system without proprietary mapping.

**OpenAPI Specification 3.1 / 3.2**
- URL: https://spec.openapis.org/oas/v3.2.0.html
- Machine-readable, vendor-neutral standard for describing RESTful APIs in JSON or YAML. A T&A system's public API should be described using an OpenAPI 3.x specification to enable auto-generated SDKs, API documentation, and integration testing. The design-first approach (writing the OpenAPI spec before implementation) is the accepted best practice.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Standard for validating the structure of JSON data. Used in conjunction with OpenAPI to define request/response data models. T&A data objects (time entries, employees, shifts, accruals) should be defined as JSON Schema documents for validation, documentation, and code generation.

---

### Authentication & Security Standards

**OAuth 2.0 — RFC 6749 / RFC 9700 (Security Best Current Practice)**
- URL: https://datatracker.ietf.org/doc/rfc9700/
- Industry-standard protocol for delegated API authorisation. A T&A API must implement OAuth 2.0 for third-party integrations (payroll systems, HRIS platforms, BI tools). RFC 9700 defines current security best practices, including PKCE enforcement, token binding, and restricting redirect URIs.

**OpenID Connect 1.0**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0 providing standardised user identity claims. Required for SSO integration with corporate identity providers (Azure AD / Entra ID, Okta, Google Workspace). T&A systems deployed in enterprise environments must support OIDC-based SSO.

**WebAuthn / FIDO2**
- W3C WebAuthn: https://www.w3.org/TR/webauthn/
- FIDO Alliance CTAP: https://fidoalliance.org/specifications/
- Standard for browser-based and native passwordless authentication using public key cryptography and device-bound biometrics (Touch ID, Face ID, Windows Hello). Applicable to T&A kiosk and web clock-in where employees authenticate without a shared PIN or password; eliminates credential-sharing (buddy-punching via PIN) without requiring proprietary biometric hardware.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- Cheatsheet: https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html
- Catalogue of the most critical API security risks. A T&A API handling sensitive biometric and payroll-adjacent data must address all OWASP API Top 10 risks, including broken object-level authorisation, broken authentication, excessive data exposure, and rate limiting. Particularly important for endpoints exposing timesheet data, biometric templates, and employee PII.

**NIST Special Publication 800-76-2 — Biometric Specifications for PIV**
- URL: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-76-2.pdf
- Defines biometric data quality and format requirements for U.S. government Personal Identity Verification. Relevant if a T&A system must interoperate with government facility access control or produce biometric records compatible with PIV credential systems.

---

### Payment & Payroll Integration Standards

**NACHA / ACH Network Rules**
- URL: https://www.nacha.org/content/compliance
- Governs electronic payroll direct deposit transactions over the ACH network, which processed 33.6 billion transactions worth $86.2 trillion in 2024. From 2026, Nacha requires standardised "PAYROLL" descriptor for prearranged payment and deposit (PPD) credit entries representing wages. A T&A system that initiates payroll payments (rather than just exporting time data) must achieve Nacha certification. Most T&A systems export to a separate payroll processor rather than initiating ACH payments directly.

---

## Similar Products — Developer Documentation & APIs

### UKG (Kronos)
- **Description:** Market-leading workforce management platform with the deepest compliance engine in the industry; covers T&A, scheduling, payroll integration, and HRIS for 70,000+ organisations.
- **API Documentation:** https://developer.ukg.com/wfm/reference/welcome-to-the-ukg-pro-workforce-management-api
- **Developer Hub:** https://developer.ukg.com/
- **SDKs/Libraries:** Available via UKG Developer Hub; REST and GraphQL APIs documented separately.
- **Developer Guide:** https://www.suretysystems.com/insights/ukg-api-overview-your-ultimate-guide-to-api-documentation/
- **Standards:** REST/JSON; GraphQL option available; OpenAPI documented internally.
- **Authentication:** Customer API key + User API key (dual-key model); OAuth 2.0 for partner integrations.

---

### ADP Workforce Now
- **Description:** Integrated payroll, time, and HR platform for mid-market; 39M+ users globally; hundreds of certified payroll carrier connections.
- **API Documentation:** https://developers.adp.com/
- **Marketplace API Catalog:** https://developers.adp.com/articles/guides/adp-workforce-now-api-catalog
- **SDKs/Libraries:** Available via ADP Developer Resources portal.
- **Developer Guide:** https://www.getknit.dev/blog/adp-api-integration-in-depth
- **Standards:** REST/JSON; SOAP-based web services also available for legacy integrations.
- **Authentication:** OAuth 2.0 (client credentials flow); requires ADP API Central subscription; Client ID + Client Secret issued via the API Central portal.

---

### Ceridian Dayforce
- **Description:** Continuous payroll + time + scheduling platform; real-time pay calculation on each clock event; strong retail and healthcare scheduling.
- **API Documentation:** https://help.dayforce.com/r/documents/Dayforce-RESTful-Web-Services-Developer-Guide
- **SDKs/Libraries:** Third-party connectors available via WSO2, Apideck, and Unified.to.
- **Developer Guide:** https://developers.getknit.dev/docs/ceridiandayforce-usecases
- **Standards:** REST/JSON; supports OData query syntax on some endpoints.
- **Authentication:** OAuth 2.0 token-based authentication; also supports username + password + API key for legacy integrations.

---

### Deputy
- **Description:** Scheduling-first workforce management with mobile clock-in, GPS geofencing, demand-based scheduling, and 50+ payroll integrations; 340,000+ workplaces.
- **API Documentation:** https://developer.deputy.com/
- **Resource API Overview:** https://developer.deputy.com/docs/resource-api-objects
- **SDKs/Libraries:** Community Python library available (github.com/tonyallan/deputy); no official SDK.
- **Developer Guide:** https://developer.deputy.com/docs/the-hello-world-of-deputy
- **Standards:** REST/JSON; webhooks for real-time event notifications; supports INFO requests (GET) for schema introspection.
- **Authentication:** OAuth 2.0 bearer tokens; personal access tokens for development.

---

### Replicon
- **Description:** Time & attendance plus project time tracking for professional services, government contractors, and mixed time-model organisations; DCAA-compliant timekeeping.
- **API Documentation:** https://www.replicon.com/help/using-the-api/
- **Getting Started:** https://www.replicon.com/help/getting-started-with-replicons-api/
- **API Concepts:** https://www.replicon.com/help/api-concepts/
- **SDKs/Libraries:** Workato connector available; custom integration via REST API.
- **Standards:** REST/JSON; endpoint structure organised by resource (Tasks, Projects, Users, Clients, Timesheets).
- **Authentication:** Basic authentication (username/password) and token-based authentication.

---

### Factorial HR
- **Description:** HRIS + time & attendance for SMBs; GDPR/CCPA-compliant biometric data handling; European market focus.
- **API Documentation:** https://apidoc.factorialhr.com/
- **Integrations Framework:** https://apidoc.factorialhr.com/docs/integrations-framework
- **SDKs/Libraries:** Third-party wrappers via Merge, Finch, and Apideck unified HR API platforms.
- **Developer Guide:** https://bindbee.dev/blog/factorial-api
- **Standards:** REST/JSON; versioned endpoints (e.g., 2024-10-01 version in URL path); OData query options supported.
- **Authentication:** OAuth 2.0; access tokens issued per application with user-delegated consent.

---

### Jibble
- **Description:** Free-tier time tracking and attendance for small teams; GPS geofencing, facial recognition via device camera, Slack/MS Teams clock-in.
- **API Documentation:** https://docs.api.jibble.io/
- **Help Guide:** https://www.jibble.io/help/using-jibbles-api-for-your-custom-needs
- **SDKs/Libraries:** Official SDKs for multiple languages available via API tracker; Zapier integration for no-code automation.
- **Standards:** REST/JSON; resource-based structure; OData query options; standard HTTP response codes.
- **Authentication:** Bearer token (personal access token generated from account settings).

---

### Homebase
- **Description:** Free scheduling and time tracking for small businesses (restaurant/retail); pre-built payroll integrations with Gusto, ADP, Paychex, QuickBooks.
- **API Documentation:** https://app.joinhomebase.com/api-docs
- **Overview:** https://www.joinhomebase.com/blog/api-data-internal-tools
- **SDKs/Libraries:** No official SDK; REST API with JSON; third-party via Apideck.
- **Standards:** REST/JSON.
- **Authentication:** API key authentication; OAuth 2.0 for partner integrations.

---

### TimeTrex (Open Source)
- **Description:** Only credible open-source workforce management platform; includes T&A, scheduling, payroll, HR, and AI-based facial recognition on commodity Android hardware. AGPL-3.0 Community Edition.
- **API Documentation:** https://help.timetrex.com/developers/corporate/packages/API.html
- **API Examples (Postman):** https://documenter.getpostman.com/view/10223102/SWTABegB
- **Developer Guide:** https://www.timetrex.com/workforce-management-api
- **SDKs/Libraries:** REST API with full 100% coverage of all UI operations; API tracing tool built into web UI (CTRL+ALT+SHIFT+F11); open-source codebase enables direct database integration.
- **Standards:** REST/JSON; 100% API coverage — all UI actions available via API; supports webhooks.
- **Authentication:** Username/password; API key; session token.

---

## Notes

**Emerging: MCP (Model Context Protocol) for AI-native T&A**
If the T&A system exposes an MCP server, AI assistants (Claude, Copilot) could directly query shift schedules, approve timesheets, and surface compliance alerts via natural language. The Model Context Protocol specification (https://modelcontextprotocol.io/) is an emerging open standard for AI tool integration that is highly relevant to an AI-native T&A system. No current T&A vendor publishes an MCP server; this represents a first-mover opportunity.

**Fragmented standards landscape**
Unlike payroll (NACHA) or healthcare (HL7 FHIR), the T&A space lacks a widely adopted open data exchange standard. HR Open Standards (HR-JSON Timecard) is the most relevant open standard but has limited adoption among commercial vendors. Most integrations rely on vendor-specific REST APIs with proprietary data models, increasing switching costs and integration complexity. An OSS T&A system built around HR-JSON Timecard for data export and OpenAPI 3.x for API description would have a meaningful interoperability advantage.

**Biometric legislation is accelerating**
As of 2026, Illinois BIPA is the most litigated biometric privacy law in the U.S., with Texas (CUBI), Washington (My Health MY Data Act), and several other states enacting similar statutes. Federal biometric privacy legislation is under active discussion in Congress. Any T&A system using biometric clock-in must implement: explicit consent collection, on-device or on-premises processing, defined retention and deletion schedules, and audit-ready consent records. Self-hosted biometric processing is the architecturally safest approach under all current and anticipated regulatory frameworks.
