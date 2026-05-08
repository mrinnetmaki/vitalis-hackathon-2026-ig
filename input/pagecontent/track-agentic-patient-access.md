# Agentic Patient Access

**Track lead: Jens Villadsen**

## Overview

This track explores whether AI agents can enable patient access to health data portals that do not natively support FHIR. The approach is inspired by the Danish [Dhroxy](https://github.com/orgs/c3po-initiative/repositories) project, developed as part of the [c3po initiative](https://github.com/orgs/c3po-initiative/repositories), which uses AI agents to provide FHIR-based access to existing patient portals.

Mikael Rinnetmaki is interested in exploring whether this approach could be replicated in Finland with the [Kanta patient portal](https://www.kanta.fi/).

## Goals

- Reverse-engineer the HTTP/API traffic of a patient-facing health portal that does not expose a FHIR API
- Build an AI-assisted proxy (or FHIR facade) that translates the portal's internal data into standard FHIR R4 resources
- Document the approach so others can replicate it for other portals

## Candidate Swedish Systems

The table below lists ten Swedish healthcare portals and central systems that are not (yet) accessible via FHIR. Any of these are valid targets for the track. Participants are also welcome to bring their own favourite portal from any country.

| # | System | Operator | Patient access | Authentication | FHIR target resources |
|---|--------|----------|----------------|----------------|-----------------------|
| 1 | [1177.se](https://www.1177.se) internal APIs (journal, inbox, appointments) | Inera | **Direct** — BankID or Freja eID+ login at e-tjanster.1177.se | BankID redirect → session cookie; every request sends `Cookie: <session>` + `X-Requested-With: XMLHttpRequest` + `Accept: application/json` | `Patient`, `Appointment`, `Communication` |
| 2 | [Nationell Patientöversikt (NPÖ)](https://www.inera.se/tjanster/alla-tjanster-a-o/npo---nationell-patientoversikt/) | Inera | **Staff only** — requires SITHS card; no patient portal | SITHS function certificate mTLS (authentication is at TLS layer, no `Authorization` header); RIV-TA SOAP with `wsa:To` = target HSA-id | `Patient`, `Encounter`, `Condition`, `Observation` |
| 3 | [1177 Tidbokning](https://inera.atlassian.net/wiki/spaces/OITB/overview) (appointment booking) | Inera | **Indirect** — patients book via 1177.se; backend is system-to-system | Same as NPÖ: SITHS function certificate mTLS + RIV-TA SOAP; service domain `riv:crm:scheduling` | `Schedule`, `Slot`, `Appointment` |
| 4 | [Elektronisk Remiss](https://www.inera.se/tjanster/alla-tjanster-a-o/elektronisk-remiss/) (e-referral) | Inera | **Staff only** — provider-to-provider; patients see status via 1177 | Same as NPÖ: SITHS function certificate mTLS + RIV-TA SOAP; domain `clinicalprocess:activity:request` | `ServiceRequest`, `Task` |
| 5 | [MittVaccin](https://www.mittvaccin.se) (vaccination records) | Cambio | **Direct** — BankID or Freja eID+ login at mittvaccin.se | BankID redirect → session cookie | `Immunization` |
| 6 | [1177 Intyg](https://intyg.1177.se) / [Webcert](https://www.inera.se/tjanster/alla-tjanster-a-o/intygstjanster/webcert/) (medical certificates) | Inera | **Direct** for patients at intyg.1177.se (BankID); staff authoring via Webcert | 1177 Intyg: BankID redirect → session cookie; Webcert (staff): SITHS eID → SAML assertion → `Authorization: Bearer <token>` (RFC 7522 exchange) | `DocumentReference`, `Composition` |
| 7 | [LabPortalen](https://www.labportalen.se) (lab results) | InfoSolutions | **Direct** — patients log in with SITHS card or SMS OTP | SSO redirect from journal system: `?sys=<SYSTEM_GUID>&UserIntegrationKey=<USER_GUID>&PID=<personnummer>`; SYSTEM_GUID is a per-vendor shared secret assigned by InfoSolutions | `DiagnosticReport`, `Observation` |
| 8 | [Tandvårdsportalen](https://www.forsakringskassan.se/tandvarden/e-tjanster-for-tandvarden/tandvardsportalen) (dental subsidies) | Försäkringskassan | **Indirect** — patients see dental subsidy data via [Mina sidor](https://www.forsakringskassan.se/mina-sidor) (BankID) | Provider portal: SOAP + function certificate from Svensk e-identitet or Expisoft (not SITHS); no public REST/FHIR API | `Coverage`, `Claim` |
| 9 | [1177 Högkostnadsskydd / e-Frikort](https://www.inera.se/tjanster/alla-tjanster-a-o/1177-hogkostnadsskydd/) (cost protection) | Inera / Regions | **Indirect** — patients view via 1177.se (BankID); data fetched on demand from regional systems | Patient side: same BankID session cookie pattern as 1177.se; backend: SITHS function certificate mTLS + RIV-TA SOAP over NTP | `Coverage`, `ExplanationOfBenefit` |
| 10 | [Patientregistret / Cancerregistret](https://www.socialstyrelsen.se/statistik-och-data/register/) (national health registers) | Socialstyrelsen | **No portal** — individual GDPR access only via written application; provider reporting via [Filip portal](https://filip.socialstyrelsen.se) (BankID) or SFTP | Filip portal: BankID → session cookie; bulk reporting: SFTP with credentials from Socialstyrelsen; no REST or FHIR API | `Encounter`, `Condition`, `Procedure` |

> **Note:** Two systems are notably *not* on this list because they already use FHIR:
> - [Nationella Läkemedelslistan (NLL)](https://www.ehalsomyndigheten.se/yrkesverksam/ngs-tjansten/) — FHIR R4 + OAuth2
> - [Nationella Vaccinationsregistret (NVR)](https://www.folkhalsomyndigheten.se/smittskydd-beredskap/vaccinationer/nationella-vaccinationsregistret/) — migrated to a FHIR-based API (NVR 2.0) in March 2026; provider-facing OAuth2 `client_credentials` flow; patient access still via MittVaccin (row 5)

Most of these systems also have Android apps available on Google Play. These apps are valid targets for decompilation and static analysis (e.g. using [jadx](https://github.com/skylot/jadx) or [apktool](https://apktool.org/)) and can reveal API endpoints, request formats, and authentication flows that are not documented anywhere publicly.

## Prerequisites

Attendees need to bring two things:

1. **A target system** — either pick one from the table above or bring your own favourite health portal that is not already on FHIR
2. **An AI tool** to which you have a license (Cursor, Claude Code, Copilot, or similar)

Everything else should be possible to do at the hackathon.

## Tasks

1. **Explore** — Log into your target portal with a test or personal account and capture the HTTP traffic using one of the methods below. Identify the key API calls that return health data.
   - **Browser DevTools — Copy as cURL** — The quickest way to capture a single request. Open the Network tab, right-click any request, and choose **Copy → Copy as cURL** (see [full tutorial](https://www.scrapingbee.com/tutorials/how-to-extract-curl-requests-from-chrome/)). The result is a self-contained shell command you can paste directly into a terminal or hand to an AI tool:
     ```bash
     curl 'https://e-tjanster.1177.se/api/core/overview/events/appointment-events' \
       -H 'Accept: application/json' \
       -H 'X-Requested-With: XMLHttpRequest' \
       -H 'Cookie: SESSION=abc123; XSRF-TOKEN=xyz'
     ```
   - **Browser DevTools — Export HAR** — Right-click anywhere in the Network tab request list and choose **Save all as HAR with content**, or use the download icon. A HAR file captures the full session (all requests, all responses, all headers) in one JSON file — better than individual cURL commands when you want an AI to reason across many requests at once.
   - **Proxy** — Route traffic through [mitmproxy](https://mitmproxy.org/) or Charles Proxy for richer inspection and scripting.
   - **Android app decompilation** — If the portal has an Android app, download the APK and decompile it with [jadx](https://github.com/skylot/jadx) or [apktool](https://apktool.org/) to extract hardcoded endpoints and request schemas without needing to intercept live traffic.

   > **Tip:** Before feeding a HAR file to an AI tool, scrub or replace real patient values (names, personnummer, dates of birth) with synthetic equivalents — the structure and field names are what matter for mapping, not the actual data. [har-sanitizer](https://github.com/cloudflare/har-sanitizer) (Cloudflare) can help automate this step. Alternatively, [chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp) lets an AI agent connect directly to Chrome DevTools and observe network traffic in real time, skipping the manual export step entirely.
2. **Map** — For each data type returned (appointments, lab results, medications, …) identify the closest FHIR R4 resource and draft a simple mapping table.
3. **Build** — Use an AI coding agent to scaffold a lightweight proxy server (any language/framework) that authenticates against the target portal and re-exposes the data as FHIR R4 resources.
4. **Validate** — Run the FHIR responses through the [FHIR validator](https://validator.fhir.org) or a local validator to check conformance.
5. **Document** — Write a short README that explains the authentication flow, the endpoint mapping, and any known limitations.
6. **API Documentation** - Consider using https://github.com/mikkelkrogsholm/api-mapper

## Expected Outcomes

- A GitHub repository (or a branch in an existing repo) containing a working or partially working FHIR proxy/facade for at least one non-FHIR Swedish health system
- A mapping document (could be a simple markdown table) showing how the target system's data model maps to FHIR R4 resources
- A short demo or write-up suitable for contributing back to the [c3po initiative](https://github.com/orgs/c3po-initiative/repositories)

## Resources

- [c3po initiative repositories](https://github.com/orgs/c3po-initiative/repositories) — reference implementations for Denmark (Dhroxy) and Sweden (inroxy)
- [inroxy](https://github.com/c3po-initiative/inroxy) — community effort to build a FHIR proxy for 1177.se (already in progress — a natural starting point for the 1177 track)
- [RIV-TA service contracts](https://rivta.se/domains/index.html) — SOAP-based service contract specifications for Inera/NTP services
- [Inera developer wiki](https://inera.atlassian.net/wiki/spaces/OITB/overview) — documentation for Inera's integration services
- [FHIR Validator](https://validator.fhir.org) — validate FHIR resources online
- [chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp) — MCP server that gives an AI agent direct access to Chrome DevTools (network tab, console, DOM) — useful for live traffic inspection without a manual HAR export


### Example starting prompt

```
# FHIR Facade Implementation — Plan Mode

You are a senior healthcare integration engineer. Your task is to **analyze provided artifacts and produce a detailed implementation plan** for a FHIR R4 facade layer before writing any code.

## Inputs provided
- One or more **HAR files** (browser network captures) that show the real HTTP traffic of the underlying proprietary API
- An **OpenAPI specification** describing the underlying API's endpoints, schemas, and authentication

## Phase 1 — Analysis (do this first, output findings before planning)

### HAR file analysis
For each HAR file:
1. Identify all unique API endpoints called (method + path)
2. Extract request headers, authentication patterns (tokens, cookies, API keys)
3. Document request/response payload shapes with example values
4. Note pagination patterns, error response formats, and any undocumented behaviors not in the OpenAPI spec
5. Flag any discrepancies between actual HAR traffic and what the OpenAPI spec documents

### OpenAPI spec analysis
1. List all available endpoints and operations
2. Identify data models and their fields
3. Note authentication/authorization schemes
4. Identify any endpoints that appear unused in the HAR files

## Phase 2 — FHIR Mapping Plan

Map the underlying API's resources to FHIR R4 resources. For each mapping, specify:

| Underlying resource (from HAR/OpenAPI) | FHIR R4 Resource | Mapping notes / gaps |
|----------------------------------------|------------------|----------------------|
| ...                                    | ...              | ...                  |

For each FHIR resource you plan to expose:
- Which underlying API calls are needed to populate it?
- What data transformations are required?
- What FHIR fields cannot be populated from available data (mark as `extension` or omit)?
- What FHIR search parameters can be supported?

## Phase 3 — Architecture Plan

Propose a concrete architecture covering:

1. **Tech stack** — recommend a language/framework appropriate for a FHIR facade (e.g. Node.js + HAPI FHIR, Python + Flask, Java Spring Boot, etc.) with justification
2. **Layer breakdown**
   - FHIR REST layer (handles incoming FHIR requests, validates, routes)
   - Mapping/translation layer (FHIR ↔ proprietary model)
   - Upstream client layer (calls the underlying API, handles auth + retries)
3. **Authentication strategy** — how the facade will authenticate to the upstream API (based on HAR evidence) and how it will authenticate its own callers
4. **Caching strategy** — where caching makes sense given the observed traffic patterns
5. **Error handling** — how upstream errors map to FHIR OperationOutcome responses
6. **Testing strategy** — unit, integration, and contract tests against FHIR conformance

## Phase 4 — Implementation Roadmap

Break the work into milestones:
- Milestone 1: Project scaffold + upstream API client (auth working, raw calls passing)
- Milestone 2: First FHIR resource end-to-end (suggest which one is simplest)
- Milestone 3: Remaining FHIR resources
- Milestone 4: Search parameter support
- Milestone 5: Validation, error handling, conformance statement (`/metadata`)

For each milestone list the files/modules that will be created.

## Constraints & decisions to surface
Before finishing the plan, explicitly flag:
- Any ambiguities in the HAR/OpenAPI data that require a human decision
- FHIR elements that are mandatory but unavailable from the source system
- Security concerns observed in the HAR files (e.g. tokens in URLs, missing TLS, overly broad scopes)
- Licensing or terms-of-service considerations if the underlying API is a third-party SaaS

---

**Do not write implementation code yet.** Deliver only the analysis and plan. Wait for approval before proceeding to code generation.
```
