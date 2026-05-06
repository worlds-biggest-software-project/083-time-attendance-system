# Time & Attendance System

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source time and attendance platform with self-hosted biometrics, automated compliance, and natural-language scheduling.

Time & Attendance System is a workforce time-capture, scheduling, and compliance platform built for organisations that need accurate clock-in data, FLSA- and EU Working Time Directive-compliant overtime calculation, and full data sovereignty over biometric records. It is designed for SMBs and mid-market employers who cannot justify the $25–$41 per-employee/month cost of UKG or Ceridian Dayforce, but require more compliance depth than free tools like Homebase or Jibble can offer.

---

## Why Time & Attendance System?

- **Incumbents are expensive and slow to deploy.** UKG Ready and UKG Pro charge $25–$41 per employee per month with 6–12 month implementation timelines, putting the market leader out of reach for the SMB and mid-market segments.
- **The only credible OSS option is dated.** TimeTrex (AGPL-3.0) is the sole open-source workforce management platform with biometric features, but its UI is described as functional but dated and its AGPL copyleft creates friction for organisations integrating it into hosted products.
- **Free tiers lack compliance depth.** Homebase and Jibble serve micro-businesses well but provide minimal FMLA tracking, no DCAA support, and limited overtime rule configuration — leaving the 5–500 employee segment underserved.
- **Biometric data sovereignty is unmet.** Biometric data is sensitive under GDPR Article 9, Illinois BIPA, Texas CUBI, and Washington My Health MY Data Act. No SaaS vendor can fully satisfy regulated-industry requirements for on-premises biometric processing.
- **Scheduling UX is largely form-based.** No incumbent currently accepts plain-language scheduling instructions, forcing managers to navigate calendar grids and drop-down menus for routine shift work.

---

## Key Features

### Time Capture & Verification

- Clock-in via web browser, mobile app, and software-based kiosk on existing tablet hardware
- GPS geofencing to restrict clock-in to authorised work locations
- Facial recognition on commodity Android/iOS devices using open-source embedding models, with on-device processing only
- Manager timesheet approval workflow with full audit trail (change history, timestamps, approver identity)
- Employee self-service portal: view schedule, request time off, review and dispute own timesheets

### Compliance & Pay Rules

- FLSA-compliant overtime calculation with configurable daily, weekly, and California double-time rules
- FMLA and state-level leave tracking with protected absence coding
- EU Working Time Directive rules: 48-hour maximum, rest period enforcement
- Real-time compliance alerting before violations occur (approaching overtime, missed rest periods, geofencing anomalies)
- DCAA-compliant timekeeping mode for U.S. government contractors with contemporaneous recording and supervisor approval

### Scheduling

- Shift scheduling with availability management, conflict detection, and coverage rules
- AI-based demand forecasting using historical attendance patterns
- Natural language schedule requests for managers ("Cover Tuesday night shift with 3 certified staff, avoid overtime")
- Shift swap workflows with optional manager approval

### Payroll Integration

- CSV export and direct integration with QuickBooks and Xero at minimum
- Standard payroll connector formats compatible with ADP, Gusto, and Paychex workflows
- iCalendar (RFC 5545) export for employee calendar apps

### Data Sovereignty & Deployment

- Self-hosted deployment option with biometric data processed entirely on-premises
- Open-source biometric libraries with disclosed model identity for auditability
- No vendor data access required for biometric or attendance records

---

## AI-Native Advantage

AI-optimised shift scheduling can balance demand forecasting, employee preferences, labour-law constraints, and cost targets simultaneously — research suggests this can reduce scheduling time from hours to minutes and cut overtime costs by 15–25 percent. Anomaly detection flags buddy-punching patterns, geofencing violations, and approaching overtime thresholds in real time, turning reactive compliance into proactive risk management. Natural language schedule management eliminates the need for grid-based scheduling UIs and reduces the training burden on shift supervisors. Predictive absence forecasting anticipates high-absence periods from historical patterns so coverage can be adjusted before gaps appear.

---

## Tech Stack & Deployment

The platform targets a self-hosted-first deployment model for biometric data sovereignty, with a hosted option for organisations preferring managed infrastructure. Facial recognition runs on commodity Android/iOS hardware using OpenCV-based open-source embedding models (e.g. FaceNet, DeepFace), eliminating the $500–$2,000 per-terminal cost of proprietary biometric clocks. Integrations follow open standards: iCalendar (RFC 5545) for schedule export, HR Open Standards for HRIS data exchange, and ISO/IEC 19794-4 for biometric data interchange. A REST API and standard CSV/XML payroll exports support QuickBooks, Xero, and connector-based integration with ADP, Gusto, Paychex, and Ceridian.

---

## Market Context

The Time & Attendance Systems market is estimated at $3.6B in 2026 and projected to reach $6.95B by 2035 (Kings Research); broader segmentations including hardware project $7.9B by 2026 (Market Research Future). UKG holds approximately 34.8 percent market share (AppsRunTheWorld, 2025), with pricing of $25–$41 per employee per month leaving a clear gap for the SMB and mid-market. Primary buyers are VPs of HR and HR Directors at 200–5,000-employee retail and manufacturing companies, payroll managers focused on FLSA/FMLA audit defence, and operations managers needing mobile clock-in and real-time attendance visibility.

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
