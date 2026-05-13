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
- One or more **HAR files** (browser network captures) showing real HTTP traffic from the underlying proprietary API
- An **OpenAPI specification** describing the underlying API's endpoints, schemas, and authentication

---

## Phase 1 — Analysis (complete before any planning)

### HAR file analysis
For each HAR file:
1. Identify all unique API endpoints called (method + path)
2. Extract request headers and authentication patterns (tokens, cookies, API keys)
   - **Note:** If the HAR contains login/auth flows (e.g. `/login`, `/oauth/token`, `/auth`), you may **skip detailed analysis of those flows** unless they reveal something structurally important about how session tokens are used downstream. Just note that auth exists and what credential type is passed on subsequent requests.
3. Document request/response payload shapes with representative field names and value types
4. Note pagination patterns, error response formats, and any behaviors not reflected in the OpenAPI spec
5. Flag discrepancies between actual HAR traffic and the OpenAPI spec

### OpenAPI spec analysis
1. List all available endpoints and operations
2. Identify data models and their fields
3. Note authentication/authorization schemes (summary only — implementation detail deferred)
4. Flag endpoints that appear unused or absent in the HAR captures

---

## Phase 2 — FHIR Resource Identification & Mapping

**Infer the appropriate FHIR R4 resources from the HAR and OpenAPI data.** Do not assume a fixed resource set — let the underlying data model drive what FHIR resources make sense to expose.

For each identified mapping:

| Underlying resource (HAR/OpenAPI) | FHIR R4 Resource | Confidence | Mapping notes / gaps |
|-----------------------------------|------------------|------------|----------------------|
| ...                               | ...              | High/Med/Low | ...                |

For each FHIR resource:
- Which underlying API calls are needed to populate it?
- What field-level transformations are required (renaming, type coercion, unit conversion, code system mapping)?
- Which mandatory FHIR fields cannot be populated from available data? (mark for `extension`, stub value, or omission with justification)
- Which FHIR search parameters (`_id`, `subject`, `date`, etc.) can be realistically supported?

---

## Phase 3 — Tech Stack Recommendation

Based on the complexity, payload shapes, and patterns observed in the HAR/OpenAPI data, recommend the most appropriate tech stack for this facade. Consider:

- **Ecosystem maturity** for FHIR (existing libraries, validators, terminology servers)
- **Transformation complexity** — if mappings are heavy, prefer a language with strong data manipulation ergonomics
- **Operational simplicity** — prefer something deployable as a single service without heavy infrastructure
- **Team familiarity signals** — if the OpenAPI spec or HAR reveals tech clues (e.g. JSON conventions, versioning style), factor that in

Provide a brief justification for your recommendation and name any key libraries (e.g. `fhir.js`, `fhirpy`, `HAPI FHIR`, `medplum`).

---

## Phase 4 — Architecture Plan

Outline the facade's internal structure:

1. **Layer breakdown**
   - FHIR REST layer — handles incoming FHIR requests, routing, and basic validation
   - Mapping/translation layer — bidirectional transforms between FHIR and proprietary models
   - Upstream client layer — calls the underlying API; handles auth token injection, retries, and error normalization
2. **Authentication strategy**
   - How the facade authenticates *to* the upstream API (based on HAR evidence — token type, header name, refresh pattern)
   - How the facade authenticates *its own callers* (recommend a sensible default; note this can be swapped out)
   - Flag whether credential handling can be simplified or stubbed for an initial implementation
3. **Caching strategy** — where caching adds value given observed traffic patterns
4. **Error handling** — how upstream errors map to FHIR `OperationOutcome` responses
5. **Capability statement** — what the `/metadata` endpoint will advertise

---

## Phase 5 — Implementation Roadmap

Break work into ordered milestones. For each, list the modules/files to be created:

- **Milestone 1:** Project scaffold + upstream API client (raw calls working, auth token injected)
- **Milestone 2:** First FHIR resource end-to-end — pick the simplest/most complete mapping from Phase 2
- **Milestone 3:** Remaining FHIR resources in priority order
- **Milestone 4:** Search parameter support
- **Milestone 5:** Validation, error normalization, `/metadata` conformance statement
- **Milestone 6:** Tests — unit for mappers, integration against a FHIR validator, contract tests

---

## Flags & Decisions Required

Before finishing the plan, surface:
- **Ambiguities** in the HAR/OpenAPI that need a human decision before implementation can proceed
- **Mandatory FHIR fields** that are unavailable from the source system and require a policy decision
- **Security observations** from the HAR (tokens in query strings, missing headers, overly broad scopes)
- **Scope recommendations** — if the HAR reveals the system is large, suggest a phased scope rather than boiling the ocean

---

**Do not write implementation code yet.**
Deliver the full analysis and plan, then pause and wait for review and approval before proceeding.
```

## Results

### Sweden

Christian Hilmersson registered late and decided to learn the track by doing it: he opened 1177 in the browser, grabbed an appointment-flow request from DevTools, sanitised it, and built a small local app that takes the raw 1177 JSON and converts it to FHIR. The interesting twist is that the model runs entirely on his own machine - a 4B-parameter Qwen model picked because it supports tool calling, where smaller open-weights models often don't. He then wrapped a UI around it with a chat panel, so once the data is converted you can ask things like "summarise my bookings" without scrolling through JSON. Two ingestion paths are supported (paste a pre-sanitised file, or pipe a fresh curl through the sanitiser script first).

Rickard Ötvös took a different angle from inside Cambio: rather than scraping 1177, he pointed an AI coding agent at Cambio's existing REST APIs - the older system-to-system endpoints that already expose much of the same data - and used it to upgrade and translate the responses into a FHIR-like shape, reusing FHIR resources where they already aligned and filling the rest from the REST payloads. The output was generated fast but is not yet validated; the takeaway is that the same agent-driven facade pattern works equally well against a vendor's existing integration surface, not just patient portals.

Jens Kristian Villadsen built [c3po-initiative/1177](https://github.com/c3po-initiative/1177), a read-only HAPI FHIR R4 proxy that fronts three Swedish 1177 services and exposes them as standard FHIR. The proxy authenticates against the Inera QA environment using HTTP Basic (personnummer + portal password), performs a SAML/Shibboleth login dance against the shared Inera IDP for each upstream SP, and joins the responses into a single FHIR API per authenticated patient. It maps `journalen` (journal records, HTML fragments inside JSON envelopes) to `DocumentReference`, `bokadetider` (appointment booking) to `Appointment`, and `e-tjanster` (patient inbox) to `Communication`/`DocumentReference`. The whole thing was assembled in under five hours of AI-assisted coding, and Jens Kristian's takeaway was that there are no technical limitations to building a FHIR facade for 1177 - it is entirely a governance question. The repo follows the pattern previously established for the Danish [Dhroxy](https://github.com/c3po-initiative/dhroxy) project.

### Finland

Mikael worked on Kanta implementation. Claude got pretty far with implementation. It was able to
find many types of information and map those to correct FHIR resource types. It required some
guidance to find the rest of the information. This seems to be due to the Kanta portal currently
being split to two instances, a legacy and a new one, with most information being available only
on one instance.

Eventually Claude got derailed attempting to implement authentication properly.

It may be worthwhile to try again, from a clean slate.

The repo used in this exercise contains personal health information and is not shared publicly.
