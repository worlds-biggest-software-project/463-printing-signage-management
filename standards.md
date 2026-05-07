# Standards & API Reference

> Project: Printing & Signage Management · Generated: 2026-05-07

---

## Industry Standards & Specifications

### ISO Standards

**ISO 12647 — Graphic Technology: Process Control for Colour Printing**
- URL: https://www.iso.org/standard/57833.html (Part 2: Offset lithography)
- Family of standards defining colour reproduction requirements and process control parameters for printing processes. Parts cover offset (12647-2), newspaper printing (12647-3), publication gravure (12647-4), screen printing (12647-5), and flexography (12647-6). Relevant to any MIS that must validate substrate and ink specifications or issue quality-control sign-off at job completion.

**ISO 15930 (PDF/X) — Prepress Digital Data Exchange Using PDF**
- URL: https://pdfa.org/resource/iso-15930-pdfx/
- Defines the PDF/X family of file formats for print-ready artwork exchange. Key versions:
  - PDF/X-4 (ISO 15930-7:2010): Complete exchange using PDF 1.6, supports colour-managed CMYK/RGB/spot colour with transparency. The current industry default for artwork submission.
  - PDF/X-6 (ISO 15930-9:2020): Successor based on PDF 2.0; adds black-point compensation, digital signatures, and N-channel profiles.
- A printing and signage MIS should validate uploaded artwork against PDF/X-4 or PDF/X-6 compliance as part of the automated prepress workflow.

**ISO 28178 — XML Schema for Print Product Metadata**
- Provides a standardised XML structure for describing print products and job metadata in supply-chain and MIS contexts.

---

### CIP4 / JDF Standards (Print Industry Specific)

**JDF — Job Definition Format (CIP4)**
- URL: https://www.cip4.org/print-automation/jdf
- Specification: https://www.cip4.org/files/cip4/documents/JDF%20Specification%201.8%20www.pdf
- GitHub: https://github.com/cip4/JDF-Specification
- XML-based format defining the entire print production process from order entry to final delivery logistics. JDF describes job intent, resources (substrates, inks, machines), and process steps. JMF (Job Messaging Format) is the companion protocol for real-time machine-to-MIS status communication (start, stop, counters, error states).
- Managed by CIP4 (International Cooperation for the Integration of Processes in Prepress, Press and Postpress). Open consortium; specification is freely available.
- Any MIS connecting to production presses, cutters, laminators, or digital front ends should use JDF/JMF for bidirectional integration.

**XJDF — Exchange Job Definition Format (CIP4)**
- URL: https://www.cip4.org/print-automation/jdf
- Simplified successor to JDF published in 2018. XJDF is designed as a pure information-interchange interface; it is automatically convertible to/from JDF. Both formats are deployed in parallel in the industry.
- Recommended for new integrations — lighter-weight than JDF while retaining full compatibility.

---

### Colour Management Standards

**ICC Profiles — International Color Consortium (ICC.1)**
- URL: https://www.color.org/iccprofile.xalter
- ICC profiles (ICC.1 v4.4 / ICCv4) characterise the colour behaviour of input and output devices. They define the mapping between a device's native colour space and a profile connection space (CIELAB or CIEXYZ), enabling consistent colour reproduction across different presses, substrates, and proofing devices.
- Used in print MIS workflows to validate artwork colour spaces, attach output ICC profiles to job tickets, and generate colour-managed proofs. The emerging iccMAX (ICC.2 / ICCv5) adds spectral and material connection spaces.

---

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines HTTP methods (GET, POST, PUT, DELETE, PATCH) and status codes. Foundational for any REST API exposed by a print MIS for integration with Web2Print storefronts, accounting systems, and third-party logistics providers.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Defines the OAuth 2.0 framework for delegated authorisation. Required for secure API access when the MIS integrates with third-party services (accounting, shipping, payment gateways) or exposes a customer-facing portal with SSO.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Defines JWTs for compact, URL-safe claims representation. Used alongside OAuth 2.0 for stateless authentication in MIS APIs and customer portal sessions.

**RFC 7807 — Problem Details for HTTP APIs**
- URL: https://datatracker.ietf.org/doc/html/rfc7807
- Defines a standard JSON error response format for HTTP APIs, improving API developer experience and consistency when the MIS returns error states.

**RFC 5321 — Simple Mail Transfer Protocol (SMTP)**
- URL: https://datatracker.ietf.org/doc/html/rfc5321
- Used for transactional email (proof approval requests, job milestone notifications, invoice delivery). Integration with SMTP or transactional email APIs (Twilio SendGrid, AWS SES) is required.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://swagger.io/specification/
- The world standard for describing REST APIs. All MIS API endpoints should be documented using OAS 3.1 to enable auto-generated SDKs, interactive documentation (Swagger UI / Redoc), and integration testing. OAS 3.1 aligns with JSON Schema 2020-12.

**JSON Schema 2020-12**
- URL: https://json-schema.org/specification.html
- Defines the schema language used within OAS 3.1 for request and response body validation. Used to validate job ticket JSON structures, artwork metadata, and inventory records.

**iCalendar (RFC 5545)**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- Standard format for calendar and scheduling data. Relevant for exporting delivery schedules and installation calendars to external calendar applications (Google Calendar, Outlook).

---

### Security & Compliance Standards

**OWASP Top 10 (2021)**
- URL: https://owasp.org/www-project-top-ten/
- Defines the ten most critical web application security risks (injection, broken authentication, SSRF, etc.). Relevant to the customer portal, artwork file upload endpoint, and any public-facing API.

**NIST SP 800-63B — Digital Identity Guidelines: Authentication**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- Defines password and authenticator assurance levels. Applicable to admin login, customer portal authentication, and API key management.

**GDPR (EU Regulation 2016/679)**
- URL: https://gdpr-info.eu/
- Requires lawful basis for processing customer personal data, data residency controls, breach notification within 72 hours, and right-to-erasure workflows. Artwork files frequently contain commercially sensitive customer data; the MIS must support per-customer data retention policies and audit logs.

**PCI DSS v4.0**
- URL: https://www.pcisecuritystandards.org/
- Applicable when the MIS processes card payments natively. In practice most platforms delegate card processing to a PCI-compliant payment gateway (Stripe, CardConnect); the MIS must ensure it never stores raw card numbers.

---

### MCP Server Specifications

MCP (Model Context Protocol) is potentially relevant for an AI-native print MIS. An MCP server exposing job data, estimation templates, and production schedules would allow AI agents (estimation assistants, scheduling optimisers, customer-service bots) to query and act on MIS data using a standardised tool interface.

- MCP specification: https://modelcontextprotocol.io/
- Candidate MCP tools for a print MIS server:
  - `get_job_status(job_id)` — retrieve live production status
  - `create_estimate(job_description)` — trigger AI-assisted estimation
  - `get_inventory_levels(substrate_id)` — query stock before scheduling
  - `schedule_job(job_id, machine_id, start_time)` — insert a job into the Gantt schedule
  - `approve_proof(job_id, version)` — record customer approval

---

## Similar Products — Developer Documentation & APIs

### PrintPLANR API

- **Description:** Cloud print MIS and Web2Print platform serving printing, signage, and promotional product businesses.
- **API Documentation:** https://www.printplanr.com/blog/printplanr-api-is-the-answer-to-connect-with-any-third-party-software-solution-for-your-print-industry/
- **SDKs/Libraries:** Not publicly documented; REST-based with custom integration support
- **Developer Guide:** Available via vendor on request
- **Standards:** REST/JSON
- **Authentication:** API key (details not publicly disclosed)

---

### EFI PrintSmith Vision / ePS PACE

- **Description:** Enterprise MIS/ERP for commercial, digital, wide-format, and specialty printing with JDF/JMF press integration.
- **API Documentation:** https://printepssw.com/ (documentation available to licensed customers)
- **SDKs/Libraries:** Not publicly available
- **Developer Guide:** Customer support portal
- **Standards:** JDF/JMF (CIP4), REST API, EFI Fiery integration
- **Authentication:** OAuth 2.0 / API key (enterprise tier)

---

### Avanti Slingshot

- **Description:** Enterprise print MIS with AI-driven scheduling, Web2Print integration, and deep JDF/JMF equipment connectivity; owned by Ricoh.
- **API Documentation:** https://github.com/Avanti-Software/api-docs (Stoplight documentation)
- **SDKs/Libraries:** Not publicly available; REST-based
- **Developer Guide:** https://avantisystems.com (support portal, login required)
- **Standards:** JDF/JMF, REST/JSON, OpenAPI
- **Authentication:** OAuth 2.0

---

### Printful API

- **Description:** Print-on-demand fulfilment platform with comprehensive REST API for product creation, order submission, and shipping tracking.
- **API Documentation:** https://developers.printful.com/docs/
- **API Documentation v2 (beta):** https://developers.printful.com/docs/v2-beta/
- **SDKs/Libraries:** Official SDK links available from the developer portal; community SDKs for Node.js, Python, PHP on GitHub
- **Developer Guide:** https://developers.printful.com/docs/ (getting started, authentication, webhooks)
- **Standards:** REST/JSON, OpenAPI 3.0
- **Authentication:** OAuth 2.0 (multi-account apps), API key (single-account)

---

### Printify API

- **Description:** Print-on-demand platform API for managing products, submitting orders, and receiving fulfilment webhooks across a global supplier network.
- **API Documentation:** https://developers.printify.com/
- **SDKs/Libraries:** Community SDKs available (GitHub: nasa8x/printify-api for Node.js)
- **Developer Guide:** https://developers.printify.com/ (authentication, products, orders, webhooks)
- **Standards:** REST/JSON
- **Authentication:** Personal Access Token (single account) or OAuth 2.0 (multi-account platform)

---

### Ordant

- **Description:** Cloud-based modular MIS for print, sign, screen printing, and apparel with automation and prepress modules.
- **API Documentation:** https://ordant.com/ (integration documentation via Connected Apps)
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** Available via vendor
- **Standards:** REST/JSON; QuickBooks API integration
- **Authentication:** API key

---

### InfoFlo Print

- **Description:** All-in-one print MIS and Web2Print with sign estimating, customer portal, and QuickBooks/Shippo/Twilio integrations.
- **API Documentation:** https://infofloprint.com/core-features/ (integration documentation available to subscribers)
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** https://infofloprint.com/docs/
- **Standards:** REST/JSON; QuickBooks Online API; Shippo API; Twilio API
- **Authentication:** API key / OAuth 2.0 (per integration)

---

### EFI Fiery DFE (Digital Front End)

- **Description:** The industry's most widely deployed digital front end (RIP) for production printers; CIP4-certified JDF DFE for press-to-MIS automation.
- **API Documentation:** Available to EFI partners and OEM customers; Fiery developer program at https://www.efi.com/products/fiery-servers-and-software/fiery-developer-program/
- **SDKs/Libraries:** Fiery API SDK (partner programme)
- **Developer Guide:** EFI Fiery Developer Program
- **Standards:** JDF/JMF (CIP4 certified), REST API (Fiery FS200 Pro+), OpenAPI
- **Authentication:** API key / OAuth 2.0

---

### Printlogic

- **Description:** Print MIS with public API for custom integrations with accounting platforms, logistics systems, and storefronts.
- **API Documentation:** https://www.printlogicsystem.com/print-mis-api.php
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** https://www.printlogicsystem.com/print-mis-api.php
- **Standards:** REST/JSON
- **Authentication:** API key

---

## Notes

- **JDF vs. XJDF:** The industry is gradually migrating from JDF to the lighter XJDF format. New integrations with press equipment should prefer XJDF where supported; JDF fallback is required for legacy machines.
- **Web2Print ↔ MIS price parity:** A recurring architectural gap is divergence between storefront pricing (often static rules) and MIS-calculated production pricing (substrate, quantity, finishing). Any new MIS should use a single shared pricing engine consumed by both the Web2Print storefront and the production estimator.
- **Mobile and PWA standards:** Shop-floor scanning (QR/barcode) and delivery proof-of-delivery are increasingly handled on smartphones. The W3C Web App Manifest and Service Worker APIs (Progressive Web Apps) allow mobile-capable features without a native app store distribution model.
- **Webhook standards:** For event-driven integration (order placed, proof approved, job dispatched), platforms should implement webhooks following the Webhook standard best practices (HTTPS POST, HMAC-SHA256 signature, idempotency keys). No single formal RFC governs webhooks, but the IETF Webhooks Community Group publishes non-normative guidance.
- **Emerging: AI/MCP integration:** No current production MIS exposes an MCP server. This is a significant opportunity for an AI-native open-source MIS to differentiate — giving LLM agents structured access to job data, estimation templates, and scheduling APIs.
