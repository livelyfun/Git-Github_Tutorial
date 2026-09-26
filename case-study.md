# Infoclick Digital Solutions — Platform Case Study & Engineering Audit

**Subject:** `infoclick_main` — Next.js 15 marketing platform with two AI-adjacent lead-generation diagnostics
**Location:** `/home/mithleshkumardas/Documents/infoclick_main`
**Repository state at audit:** branch tip `dddd504` ("web replace update"); working tree dirty (`app/api/ai-assessments/route.ts`, `package-lock.json` modified; `scratch.ts` untracked)
**Document classification:** Internal engineering document
**Date of audit:** 2026-09-26

> **Handling note.** This document names the location of committed secret material but **redacts all secret values**. It is intended to live in the repository and must not be published externally in this form. Findings are evidence-bound: every claim cites a `file:line` reference that was verified against source during the audit.

---

## Table of contents

| § | Section |
|---|---------|
| 1 | [Executive Summary](#1-executive-summary) |
| 2 | [System Overview](#2-system-overview) |
| 3 | [Architecture](#3-architecture) |
| 4 | [Engine Deep-Dives](#4-engine-deep-dives) |
| 5 | [Technology Decisions & Trade-offs](#5-technology-decisions--trade-offs) |
| 6 | [Advantages](#6-advantages) |
| 7 | [Disadvantages & Risks](#7-disadvantages--risks) |
| 8 | [Security Analysis](#8-security-analysis) |
| 9 | [SEO Analysis](#9-seo-analysis) |
| 10 | [Recommended Improvements](#10-recommended-improvements) |
| 11 | [Prioritised Roadmap](#11-prioritised-roadmap) |
| 12 | [Appendices](#12-appendices) |

---

## 1. Executive Summary

### 1.1 What was audited

Infoclick Digital Solutions is a production Next.js 15 application serving two purposes simultaneously: a **marketing website** for a digital agency operating in Eastern Nepal, and a **pair of lead-generation diagnostics** that produce client-facing audit reports. The two diagnostics — a *Digital Visibility Check* and an *AI Opportunity Assessment* — are the platform's actual product. The website exists to feed them.

The application is roughly **14,000 lines** across 74 source files, backed by a serverless Neon PostgreSQL database. It is deployed in `output: 'standalone'` mode and is configured for Google AI Studio / Cloud Run hosting (`metadata.json:5`, `next.config.ts:34`).

### 1.2 Method

The audit was conducted by full static reading of all application source, plus programmatic verification of specific claims:

- Every one of the **34 HTTP endpoints** was enumerated and programmatically audited for authentication enforcement.
- Both scoring engines were reverse-engineered formula-by-formula, including arithmetic simulation of score floors and ceilings.
- The client-to-API contract on the primary conversion form was diffed field-by-field against the route handler that consumes it.
- Tailwind v4 plugin registration was checked directly against `app/globals.css`.
- Claims surfaced by automated exploration were re-verified against source before inclusion here; unverified claims are marked as such and excluded from the findings register.

### 1.3 Verdict

The platform is **commercially clever and operationally under-defended**. Its genuine engineering strengths — SSRF hardening, a well-considered print stylesheet, genuinely broad structured-data coverage, strict TypeScript, and a zero-dependency cost profile for the core diagnostic — are undercut by three classes of systemic problem:

1. **There is no authentication.** Not weak authentication, not partial authentication — none. Thirty-four endpoints, including every admin CRUD route and both customer-report read/write routes, enforce nothing. The one file that touches authentication sets a cookie that no code ever reads.
2. **The reporting layer asserts things the system does not measure.** The competitor benchmark compares a website against itself. The email delivery endpoint reports `DELIVERED` without a mail transport. Several "derived" metrics are hard-coded strings or arithmetic identities that return the same value on every input.
3. **The primary conversion form is wired to the wrong field names.** Four of its inputs are silently discarded on every submission, which means every visibility score the platform has ever produced was computed from an incomplete form.

None of these are architectural mistakes requiring a rewrite. All are correctable at bounded cost. §10 specifies the remediation for each.

### 1.4 Top findings

| # | Finding | Severity | Evidence |
|---|---------|----------|----------|
| 1 | **34 of 34 endpoints have no auth enforcement.** The login route sets a cookie nothing reads; `/api/admin/data` returns every lead, audit, and assessment (with PII) to any caller. | **Critical** | `app/api/admin/auth/route.ts:6-19`; all of `app/api/admin/*` |
| 2 | **Committed secret material.** An unencrypted OpenSSH private key is tracked in git and not gitignored; a live Neon connection string with an embedded password is the hard-coded `DATABASE_URL` fallback. | **Critical** | `./y`, `./y.pub`, `.gitignore`; `lib/db.ts:671-673` |
| 3 | **The "AI" visibility audit contains no AI.** `@google/genai` is imported in exactly one production file. The 790-line audit engine is pure regex and arithmetic, yet its output is presented as AI analysis. | **High** | `app/api/analyze-website/route.ts` (whole file) |
| 4 | **Broken client↔API contract.** Form sends `primarySearchKeywords`/`googleBusinessProfileUrl`/`socialMediaUrls`/`competitorUrls`-as-string; route reads `primarySearchTerm`/`gbpUrl`/`socialUrls`/array. Four inputs discarded, ~45 raw score points lost, every submission. | **High** | `app/digital-visibility-check/page.tsx:95-98` vs `app/api/analyze-website/route.ts:16-26` |
| 5 | **Report email delivery is fictional.** No mail dependency exists; the endpoint hard-codes `status: 'DELIVERED'`. The client separately reports success on all three outcomes including network failure. | **High** | `app/api/reports/send-email/route.ts:71-80`; `app/digital-visibility-check/page.tsx:157-164` |
| 6 | **Fabricated competitor benchmark.** Competitors are never fetched. The comparison table is the target's own scores relabelled, rendered as a genuine head-to-head. | **High** | `app/api/analyze-website/route.ts:508-542` → `app/reports/visibility/[id]/page.tsx:877-964` |
| 7 | **Tailwind plugins installed but never registered.** No `@plugin` directive exists, so all 7 `animate-in` entrances are silent no-ops and `prose` generates no styles — every insight article renders as unstyled text. | **High** | `app/globals.css:1`; `app/insights/[slug]/page.tsx:111` |
| 8 | **Score floors make the bottom half of the display range unreachable.** A completely unreachable site scores 38/100. | **Medium** | `app/api/analyze-website/route.ts:324-400` |
| 9 | **Silent LLM degradation with no provenance.** Gemini output and a hard-coded fallback are indistinguishable in the response and the database record. | **High** | `app/api/ai-assessments/route.ts:188-211` |
| 10 | **Report pages are indexable but empty to crawlers.** Client-rendered, no metadata, no `noindex`, not blocked in `robots.txt`, canonicalised to the homepage. | **High** | `app/reports/*/[id]/page.tsx`; `app/robots.ts:11`; `app/layout.tsx` canonical |

### 1.5 Overall risk rating

| Dimension | Rating | Rationale |
|---|---|---|
| Confidentiality | **High risk** | Unauthenticated access to full customer PII corpus; committed credentials. |
| Integrity | **High risk** | Unauthenticated `PUT`/`DELETE` on CMS content and user records. |
| Availability | **Low risk** | No destructive public endpoint; DB writes are additive. |
| Analytical trust | **High risk** | Client-facing reports contain fabricated and mislabelled findings. |
| Maintainability | **Moderate risk** | 2,597-line single-component admin; scoring logic undocumented. |
| SEO posture | **Moderate risk** | Strong structured data undermined by plugin and metadata defects. |

---

## 2. System Overview

### 2.1 Product surface

The application exposes **16 routes** across three functional tiers.

**Marketing tier** (10 routes) — `/`, `/about`, `/services`, `/industries`, `/contact`, `/case-studies`, `/case-studies/[slug]`, `/insights`, `/insights/[slug]`, `/testimonials`. All are Server Components. Content is served either from static literals or from the database.

**Diagnostic tier** (4 routes) — the actual product.
- `/digital-visibility-check` — single-page form; posts a URL, receives a 0–100 score across seven categories.
- `/ai-assessment` — 4-step wizard (22 fields); posts a business profile, receives an AI-readiness score and a generated roadmap.
- `/reports/visibility/[id]` — 13-section client-rendered report document.
- `/reports/ai-opportunity/[id]` — 12-section client-rendered report document.

**Administration tier** (1 route) — `/admin-route-login`, a single 2,597-line client component that bundles authentication, a lead CRM, a case-study editor, an insights editor, a testimonial manager, user management, and a system diagnostics panel. There is no route splitting.

### 2.2 Business context

Infoclick Digital Solutions (`lib/seo.ts:1-46`) positions itself as an AI and digital-systems consultancy operating from Biratnagar, Koshi Province, Nepal, with declared service coverage across Biratnagar, Dharan, Itahari, Kathmandu, Pokhara, and internationally. The commercial thesis is stated plainly in the seeded insights content: Nepali SMEs lose a large fraction of high-intent inquiries to slow response times, and static web forms with 70%+ abandonment are the primary culprit (`lib/db.ts:539-549`).

This matters for the audit because it establishes the **evidentiary standard the product must meet**. The platform's differentiator is *trustworthy automated assessment*. Every finding in §7 concerning fabricated output is a direct attack on that differentiator, not a cosmetic defect.

### 2.3 Primary user journeys

**Journey A — Visibility audit (the conversion funnel).**

```mermaid
sequenceDiagram
    autonumber
    actor V as Visitor
    participant F as /digital-visibility-check
    participant API as POST /api/analyze-website
    participant SSRF as lib/ssrf.ts
    participant T as Target website
    participant DB as Neon PostgreSQL

    V->>F: Enters URL + 4 detail fields + optional contact
    F->>API: POST (12 keys)
    API->>API: Rate limit check (10 req/min per IP)
    API->>SSRF: validateUrlForSSRF(url)
    SSRF-->>API: valid / normalised URL
    API->>T: GET page (8s AbortController, 1.5MB cap)
    T-->>API: HTML
    API->>T: GET /robots.txt (3.5s)
    API->>T: HEAD /sitemap.xml (3.5s)
    API->>API: 40+ regex signals → 7 category scores
    API->>API: Weighted aggregate → overallScore
    API->>DB: INSERT visibility_audits (JSONB blob)
    API->>DB: INSERT leads (if contact given)
    API->>DB: INSERT analytics_events
    API-->>F: { success, auditId, audit, ...full record spread }
    F-->>V: Score badge + 7 bars + 4 issues
    V->>F: "Open Full PDF Report"
    Note over F: navigates to /reports/visibility/[id] — full page reload
```

**Journey B — AI readiness assessment.**

```mermaid
sequenceDiagram
    autonumber
    actor V as Visitor
    participant W as /ai-assessment (4-step)
    participant API as POST /api/ai-assessments
    participant Z as Zod
    participant H as Heuristic scorer
    participant G as Gemini 2.5 Flash
    participant DB as Neon PostgreSQL

    V->>W: Completes 4 steps (22 fields)
    W->>API: POST payload
    API->>Z: safeParse (25-field schema)
    Z-->>API: parsed
    API->>H: Compute 6 opportunity scores + readiness
    H-->>API: scores (INDEPENDENT of LLM)
    alt GEMINI_API_KEY set
        API->>G: generateContent, responseMimeType=json
        G-->>API: nested JSON (no responseSchema, no token cap)
        API->>API: JSON.parse — uncaught, falls through on throw
    else key absent
        API->>API: use hard-coded fallback prose
    end
    Note over API,G: No provenance flag — paths are indistinguishable
    API->>DB: INSERT ai_assessments (record stored twice)
    API->>DB: INSERT leads + analytics_events
    API-->>W: { success, assessmentId, assessment }
    W-->>V: Hours-saved gauge + link to 12-section report
```

**Journey C — Report consumption.** The visitor lands on a client-rendered report, which fetches its own payload via `useEffect`, renders a branded document, and offers three delivery actions: print-to-PDF (`window.print()`), email (no-op, §7.2), and WhatsApp share (opens `wa.me` with a pre-filled message containing the report URL).

---

## 3. Architecture

### 3.1 Container view

```mermaid
flowchart TB
    subgraph Client["Browser / Client"]
        MKT["Marketing pages<br/>10 Server Components"]
        DIAG["Diagnostics<br/>/digital-visibility-check<br/>/ai-assessment"]
        RPT["Report documents<br/>2 client-rendered routes"]
        ADM["Admin CMS<br/>2,597-line monolith"]
    end

    subgraph Edge["Next.js Server Runtime — output: standalone"]
        RSC["React Server Components<br/>+ db.* direct calls"]
        RT["Route Handlers<br/>15 route.ts files"]
        GUARD["lib/ssrf.ts<br/>rate limit + URL validation"]
        SEO["lib/seo.ts<br/>JSON-LD builders"]
        DB["lib/db.ts<br/>1,843-line repository layer<br/>pg.Pool singleton"]
    end

    subgraph External["External Services"]
        NEON[("Neon PostgreSQL<br/>8 tables")]
        GEM["Google Gemini API<br/>gemini-2.5-flash"]
        TGT["Target websites<br/>(audit subjects)"]
        MAIL["Email transport<br/>*** DOES NOT EXIST ***"]
    end

    MKT --> RSC
    DIAG --> RT
    RPT --> RT
    ADM --> RT
    RSC --> DB
    RT --> GUARD
    RT --> DB
    RT --> GEM
    RT --> TGT
    RSC --> SEO
    RT -.-> MAIL

    style MAIL stroke-dasharray: 5 5,stroke-width:2px
    style GEM stroke-width:2px
```

### 3.2 The storage pattern

The platform uses a deliberate and unusual schema strategy: **a small set of indexed scalar columns for querying, plus one wide `JSONB` column carrying the entire rich record.**

For a visibility audit, ten scalar columns (`business_name`, `website`, `industry`, `location`, `email`, `phone`, `contact_name`, `overall_score`, `status`, `requested_full_report`) support admin list views, while `report_data` holds a ~165-field JSON document (`lib/db.ts:1231-1252`). The row mapper then reconstructs the object by spreading the blob first and overlaying scalars second, so columns always win (`lib/db.ts:978-994`):

```ts
// lib/db.ts:978-994 — simplified
return {
  ...reportData,                                   // blob first
  id: row.id,                                      // scalars override
  businessName: row.business_name || reportData.businessName || 'Business Audit',
  overallScore: row.overall_score ?? reportData.overallScore ?? 0,
  // ...
};
```

**Advantage.** Adding a diagnostic output requires no migration — the blob absorbs it. Schema evolution is effectively free.

**Disadvantage.** The database cannot answer questions about the data. "How many audits mentioned WhatsApp as a missing trust signal?" requires a full scan and JSONB path extraction in application code. There are no indexes on any JSONB path, no aggregate queries, and no reporting layer. The admin dashboard is therefore forced to load the entire table into memory to display a list.

AI assessments store the record **twice** — once in `raw_submission` and again, superset-included, in `opportunities` (`lib/db.ts:1342-1343`).

### 3.3 Entity-relationship model

```mermaid
erDiagram
    LEADS {
        varchar id PK
        varchar name
        varchar email
        varchar phone
        varchar company_name
        varchar status "NEW, CONTACTED, QUALIFIED, PROPOSAL, WON, LOST, ARCHIVED"
        jsonb   notes
        varchar source
        varchar source_page
        varchar cta_identifier
        timestamptz created_at
    }

    VISIBILITY_AUDITS {
        varchar id PK "audit-timestamp-random"
        varchar business_name
        text    website
        varchar industry
        varchar location
        varchar email "PII, cleartext"
        varchar phone "PII, cleartext"
        integer overall_score
        boolean requested_full_report
        jsonb   report_data "full ~165-field record"
        jsonb   snapshot "declared, never written"
        timestamptz created_at
    }

    AI_ASSESSMENTS {
        varchar id PK "ai-assess-timestamp-random"
        varchar business_name
        varchar email "PII, cleartext"
        varchar phone "PII, cleartext"
        varchar contact_name
        varchar industry
        varchar employee_count
        integer readiness_score
        jsonb   raw_submission "record, 1st copy"
        jsonb   opportunities "same record, 2nd copy"
        timestamptz created_at
    }

    ANALYTICS_EVENTS {
        varchar id PK
        varchar event_type
        varchar cta_location
        varchar page_path
        jsonb   metadata
        timestamptz created_at
    }

    TESTIMONIALS {
        varchar id PK
        varchar client_name
        varchar client_title
        varchar company_name
        varchar quote
        integer rating
        boolean featured
    }

    CASE_STUDIES {
        varchar id PK
        varchar slug UK
        varchar client
        varchar industry
        text    problem
        text    strategy
        text    solution
        text    technology
        text    outcome
        boolean featured
    }

    INSIGHTS {
        varchar id PK
        varchar slug UK
        varchar title
        text    content
        jsonb   tags
        varchar author_name
        boolean published
        timestamptz published_at
    }

    USERS {
        varchar id PK
        varchar email UK
        varchar username UK
        varchar password_hash "declared, never written"
        varchar role
        timestamptz last_login
    }
```

**Schema divergence.** `lib/schema.sql` is a stale, unused artefact. It declares an `admin_users` table and a `business_audits` table that are never created, and its `leads` table has `inquiry`/`budget_range` columns where the live code writes `message`/`estimated_budget`. The authoritative DDL is the inline bootstrap in `lib/db.ts:701-947`, which runs `CREATE TABLE IF NOT EXISTS` on every cold start. The two files disagree, and only one of them is real.

Note also that `users.password_hash` (`lib/db.ts:756`), `visibility_audits.snapshot` (`lib/db.ts:797`), and `leads.utm_source`/`utm_medium`/`utm_campaign` are declared but never written. The interface `User` has no password field, confirming authentication was never built.

### 3.4 The bootstrap anti-pattern

`ensureDbInitialized()` (`lib/db.ts:701-947`) is invoked before every single repository method. It is guarded by a module-level boolean, and on first call it issues the full DDL script, three `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` migration batches, three `COUNT(*)` queries, and conditionally up to thirteen seed `INSERT`s.

The guard is not concurrency-safe. Two concurrent cold starts can both observe `isDbInitialized === false` and both execute the DDL. This is survivable only because every statement is `IF NOT EXISTS`, but it means the bootstrap is re-entrant-by-luck rather than by design.

More significantly, **the function swallows its own failures**:

```ts
// lib/db.ts:943-946
} catch (err) {
  console.error('Database initialization warning:', err);
  // Let subsequent queries proceed
}
```

If DDL fails, the error is logged and execution continues. The user request then proceeds to a query against a table that may not exist, producing a confusing downstream error instead of a clear initialisation failure. This is the mechanism behind the recommendation in §8 to move schema ownership to a migration step.

### 3.5 Request lifecycle characteristics

| Concern | Current state | Reference |
|---|---|---|
| Rendering strategy | Undefined. No `generateStaticParams`, `revalidate`, `dynamic`, or `runtime` export exists anywhere in `app/`. | project-wide |
| Caching | DB-backed Server Components with no dynamic API call may be prerendered at build time and serve stale CMS content indefinitely. | — |
| Detail routes | Re-query per request with no revalidation. | `app/case-studies/[slug]`, `app/insights/[slug]` |
| Report routes | Ship a full React bundle, then block on a `useEffect` round-trip. | both report pages |
| Error boundaries | None. No `error.tsx`, `global-error.tsx`, `not-found.tsx`, or `loading.tsx` anywhere. | project-wide |
| Observability | `console.error` only. No Sentry, no OpenTelemetry, no structured logging. | project-wide |
| Request timeout | Worst case ≈ 8s + 3.5s + 3.5s + DB. No `maxDuration` export. | `analyze-website/route.ts` |

---

## 4. Engine Deep-Dives

The platform's value rests entirely on two scoring engines. Both were reverse-engineered in full.

### 4.1 Engine 1 — the deterministic visibility scorer

`app/api/analyze-website/route.ts` is 790 lines and contains **no calls to any language model**. It is a fetch-and-regex pipeline. This is a defensible engineering choice in isolation — deterministic scoring is auditable, free, fast, and reproducible — but the product presents its output as AI analysis, and that mismatch is a commercial risk (§7.2).

#### 4.1.1 Pipeline stages

```mermaid
flowchart TD
    S0["0 · Rate limit<br/>x-forwarded-for, 10/min in-memory"] --> S1["1 · Destructure body<br/>NO schema validation"]
    S1 --> S2["2 · SSRF validation<br/>protocol · userinfo · blocklist<br/>IPv4/IPv6 private ranges · DNS resolve"]
    S2 --> S3["3 · Fetch page<br/>8s AbortController · 1.5MB cap<br/>on failure: status=504, html=''"]
    S3 --> S4["4 · robots.txt<br/>gated on status===200<br/>content-based detection"]
    S4 --> S5["5 · sitemap.xml<br/>HEAD request, 3.5s<br/>many CDNs 405 on HEAD"]
    S5 --> S6["6 · Extract 40+ signals<br/>regex on raw HTML string<br/>no DOM parser, no entity decoding"]
    S6 --> S7["7 · Score 7 categories<br/>clamped, weighted"]
    S7 --> S8["8 · Generate findings<br/>[VERIFIED] / [ESTIMATED] tags<br/>competitors NEVER fetched"]
    S8 --> S9["9 · Persist<br/>JSONB blob + 10 scalars"]
    S9 --> S10["10 · Lead capture<br/>if contact given"]
    S10 --> S11["11 · Analytics event<br/>PII-free"]
    S11 --> S12["12 · Respond<br/>record returned 3x"]

    style S6 stroke-width:2px
    style S8 stroke-width:2px
    style S3 stroke-dasharray: 3 3
```

**Stage 3 is the most consequential.** A fetch failure is caught and discarded:

```ts
// app/api/analyze-website/route.ts:84-87
} catch (err: any) {
  responseStatus = 504;
  html = '';
}
```

No logging, no distinction between DNS failure, TLS failure, timeout, and non-HTML response. The pipeline then continues and returns a complete, confident-looking report. The user is not told the site could not be reached.

#### 4.1.2 The scoring formulas

All seven category scores are additive rule systems with a base value, conditional increments, and a clamp. The complete set:

**Technical foundations** — `analyze-website/route.ts:324-334`
```ts
let techScore = 30;
if (isHttps) techScore += 25;
if (responseStatus === 200) techScore += 15;
if (responseTimeMs > 0 && responseTimeMs < 1200) techScore += 15;
else if (responseTimeMs < 2500) techScore += 8;   // ← unguarded, see below
if (hasViewport) techScore += 10;
if (hasRobotsTxt) techScore += 5;
if (hasSitemapXml) techScore += 5;
if (hasCanonical) techScore += 5;
if (hasHsts) techScore += 5;
techScore = Math.min(100, Math.max(25, techScore));
```
Raw maximum 115, clamped to 100. Range **[25, 100]**.

> **Defect.** The `else if` at line 328 is not guarded by `> 0`. On a total fetch failure `responseTimeMs === 0`, and `0 < 2500` is true, so **+8 is awarded for a request that never completed.**

**Search visibility** — `:337-345`
```ts
let searchScore = 30;
if (hasTitle) searchScore += 15;
if (titleLength >= 35 && titleLength <= 65) searchScore += 10;
if (hasMetaDesc) searchScore += 15;
if (metaDescLength >= 110 && metaDescLength <= 165) searchScore += 10;
if (h1Count === 1) searchScore += 10;
if (keywordMentionsInTitle) searchScore += 10;
if (!noIndexTag) searchScore += 10;
searchScore = Math.min(100, Math.max(20, searchScore));
```
Raw maximum 110, clamped to 100. Range **[20, 100]**. The title and description windows are internally consistent with the report UI, which labels 35–65 and 110–165 as "Optimal" (`app/reports/visibility/[id]/page.tsx:595-597, 610-612`).

**Local visibility** — `:348-354`
```ts
let localScore = 30;
if (gbpUrl && gbpUrl.trim().length > 5) localScore += 25;
else if (/google\.com\/maps|maps\.app\.goo\.gl/i.test(html)) localScore += 20;
if (cityMentionsInBody) localScore += 20;
if (hasPhone) localScore += 15;
if (hasExplicitAddress) localScore += 10;
localScore = Math.min(100, Math.max(20, localScore));
```
Maximum exactly 100. Range **[20, 100]**.

> The heaviest single local signal (+25) is `gbpUrl` — a value the user typed into a form, not something the engine observed. Because the client sends this field under the wrong name (§4.3), it is **always** `undefined`, and the platform permanently penalises every audited business by 25 points. Additionally, `gbpUrl` is only ever supplied by a user who already has a Google Business Profile — so the metric rewards the digitally literate and penalises precisely the businesses the agency targets.

**Content SEO** — `:357-365`
```ts
let contentScore = 30;
if (h1Count >= 1) contentScore += 10;
if (h2Count >= 2) contentScore += 10;
if (pageSizeKb >= 15) contentScore += 15;
if (servicePagesFound.length >= 2) contentScore += 15;
if (totalImages > 0 && missingAltCount === 0) contentScore += 10;
if (hasFaqContent) contentScore += 10;
if (keywordMentionsInBody) contentScore += 10;
contentScore = Math.min(100, Math.max(25, contentScore));
```
Raw maximum 110, clamped to 100. Range **[25, 100]**.

> `pageSizeKb` is computed as `html.length / 1024` — a **character** count of the raw HTML string, including all `<script>`, `<style>`, and JSON-LD payloads (`:136`). A page with a 200 KB inline script and no visible content scores as "substantial". The same variable is then used with *inverted* thresholds in two places: `>= 15` means "not thin" (`:360`) while `< 15` means "High (Thin content)" (`:640`).

**AEO and GEO** — `:296-318`, `:368`
```ts
const aeoScore = Math.min(100,
  (hasFaqContent ? 35 : 10) +
  (detectedSchemas.length > 0 ? 30 : 0) +
  (questionHeadersCount > 2 ? 20 : 10) +
  (hasTitle && hasMetaDesc ? 15 : 5)
);
const geoScore = Math.min(100,
  (hasAboutUs ? 25 : 10) +
  (detectedSchemas.includes('Organization') || detectedSchemas.includes('LocalBusiness') ? 35 : 10) +
  (hasExplicitAddress ? 20 : 5) +
  (servicePagesFound.length > 0 ? 20 : 10)
);
const aeoGeoReadiness = Math.round((aeoScore + geoScore) / 2);
```
`aeoScore` ∈ [25, 100]; `geoScore` ∈ [35, 100]; combined ∈ **[30, 100]**.

> `questionHeadersCount` is computed by matching interrogative phrases across tag-stripped body text (`:300`) — but that text retains script and JSON-LD content, so JavaScript can satisfy the signal.

**Conversion readiness** — `:371-378`
```ts
let convScore = 30;
if (hasWhatsApp) convScore += 25;
if (hasPhone) convScore += 15;
if (hasContactForm) convScore += 15;
if (hasBookingForm) convScore += 10;
if (hasTestimonials) convScore += 10;
if (hasPricing) convScore += 5;
convScore = Math.min(100, Math.max(20, convScore));
```
Raw maximum 110, clamped to 100. Range **[20, 100]**. This is the best-evidenced category — every input is directly observable in the markup.

**Digital presence** — `:381-389`
```ts
let presenceScore = 30;
const socialInputsProvided =
  socialUrls && Object.values(socialUrls).some((u) => typeof u === 'string' && u.trim().length > 5);
if (socialInputsProvided) presenceScore += 20;
if (fbFound) presenceScore += 15;
if (igFound) presenceScore += 15;
if (liFound) presenceScore += 10;
if (ytFound || ttFound) presenceScore += 10;
presenceScore = Math.min(100, Math.max(25, presenceScore));
```
Maximum exactly 100. Range **[25, 100]**. The +20 self-reported input is dead for the same reason as `gbpUrl` (§4.3).

#### 4.1.3 The weighted aggregate

```ts
// app/api/analyze-website/route.ts:392-400
const overallScore = Math.round(
  techScore       * 0.18 +
  searchScore     * 0.18 +
  localScore      * 0.15 +
  contentScore    * 0.15 +
  aeoGeoReadiness * 0.14 +
  convScore       * 0.12 +
  presenceScore   * 0.08
);
```

The seven weights sum to exactly 1.00.

```mermaid
flowchart LR
    subgraph Inputs["Weighted inputs — sum = 1.00"]
        T["technicalFoundations<br/>0.18"]
        S["searchVisibility<br/>0.18"]
        L["localVisibility<br/>0.15"]
        C["contentSeo<br/>0.15"]
        A["aeoGeoReadiness<br/>0.14"]
        V["conversionReadiness<br/>0.12"]
        P["digitalPresence<br/>0.08"]
    end
    T --> AGG["Σ weighted<br/>Math.round<br/>overallScore"]
    S --> AGG
    L --> AGG
    C --> AGG
    A --> AGG
    V --> AGG
    P --> AGG
    AGG --> BAND{"UI band<br/>≥75 Strong<br/>≥50 Actionable<br/>else Critical"}
    BAND --> OUT["0–100 displayed score"]

    style T stroke-width:3px
    style S stroke-width:3px
```

**Theoretical range: [23, 100]**, computed from the sum of all category floors.

**Calibration analysis.** Simulating a *completely unreachable* website — `fetch` throws, so `html = ''`, `responseStatus = 504`, `responseTimeMs = 0`, and `isHttps = true` because the URL string begins with `https://`:

| Category | Value | Derivation |
|---|---|---|
| technical | **63** | `30` base + `25` (HTTPS inferred from the URL string) + `8` (the line-328 defect) |
| search | **40** | `30` base + `10` (`!noIndexTag` — no HTML means no noindex tag) |
| local | 30 | base only |
| content | 30 | base only |
| aeoGeo | 30 | floor of `(25 + 35) / 2` |
| conversion | 30 | base only |
| presence | 30 | base only |

```
overallScore = round(63·.18 + 40·.18 + 30·.15 + 30·.15 + 30·.14 + 30·.12 + 30·.08)
             = round(37.74) = 38
```

A site that does not exist scores **38/100**. A live but unoptimised site lands around 55–70. The practical consequence: **the bottom half of the displayed range is effectively unreachable**, and the only real calibration is the three-tier label in the client UI (`app/digital-visibility-check/page.tsx:448-453`).

#### 4.1.4 Provenance tags are unreliable

Findings are prefixed `[VERIFIED]` or `[ESTIMATED]`, and the report includes a legend explaining the distinction (`app/reports/visibility/[id]/page.tsx:451-481`). The distinction does not hold:

| Tag | Claim | Reality | Ref |
|---|---|---|---|
| `[VERIFIED]` | "Valid SSL/TLS security certificate deployed" | Derived from `targetUrl.startsWith('https://')` — the URL **string**, not a TLS handshake. A site serving `https://` on an expired certificate still passes. | `:52, 414-418` |
| `[ESTIMATED]` | "Google Business Profile link not connected" | 100% determined by whether the user typed a value into a form field. Not an estimate. | `:485-488` |
| `[ESTIMATED]` | Competitor benchmark row | Not an estimate — a fabrication. Competitors are never fetched. | `:508-542` |

#### 4.1.5 The fabricated competitor benchmark

`competitorComparison` is constructed by taking **the target's own scores** and mapping them through fixed thresholds, once per competitor:

```ts
// app/api/analyze-website/route.ts:508-542 — representative
const isAheadInTech = techScore >= 70;
const isAheadInConversion = convScore >= 65;
const isAheadInSeo = searchScore >= 65;
```

No request is ever made to any competitor URL. The two code paths — one for real URLs, one for the synthetic `'Regional Industry Benchmark'` fallback — use **different thresholds and different label polarity**; in the real-URL path, `localVisibility` can never evaluate to "Ahead" (`:515-527` vs `:530-542`). Every row receives an identical hard-coded `summary` string.

The result is rendered as a genuine head-to-head comparison table in the client report (`app/reports/visibility/[id]/page.tsx:877-964`).

Because the client sends `competitorUrls` as a comma-separated **string** rather than the array the route expects (§4.3), `rawCompetitors` is always empty and the platform **always** falls through to the synthetic benchmark row.

#### 4.1.6 Derived metrics that carry no information

Several fields presented as analysis are constant, duplicated, or hard-coded. These are catalogued in Appendix A.2; the most consequential:

| Field | Behaviour | Ref |
|---|---|---|
| `geoDetails.structuredDataScore` | `detectedSchemas.length * 20` — **unclamped**, can exceed 100. Six schemas renders as "120% Entity Synced" in the report. | `:696`; report `:730` |
| `aeoDetails.conversationalIntentScore` | Exactly `aeoScore`. A duplicate metric displayed as an independent dimension. | `:688` |
| `aeoDetails.trustSignals` | `strengths.slice(0, 3)` — the first three findings, always in insertion order. | `:687` |
| `searchVisibilityDetails.serpFeatures` | Hard-coded `['Local 3-Pack','Sitelinks','FAQ Rich Snippets']` for every website. | `:672` |
| `localVisibilityDetails.locationLandingPages` | Hard-coded `false`, unconditionally. | `:658` |
| `socialDetails.activityEstimate` | Hard-coded `'Cross-linking detected in site structure'` — asserted even when zero links exist. | `:729` |
| `conversionDetails.trustSignals` | Hard-coded, including `'SSL Secure'` — printed even when the site is plain HTTP. | `:718` |
| `contentSeoDetails.reviewVolumeEstimate` | Hard-coded string referencing a "full GBP API sync" that is not implemented. | `:655` |
| `recommendedServices` | Four entirely hard-coded strings, not derived from any signal. | `:545-566` |

### 4.2 Engine 2 — the Gemini assessment pipeline

`app/api/ai-assessments/route.ts` (290 lines) is the **only** production file that imports `@google/genai`. It is a genuinely different design from Engine 1: a heuristic pre-scorer, an LLM for narrative content, and a hard-coded fallback.

#### 4.2.1 Heuristic scoring

Six opportunity scores are computed from boolean feature flags (`:54-70`), followed by a weighted readiness aggregate (`:72-80`):

```ts
const overallAIReadinessScore = Math.round(
  (automationOpportunity        * 0.22 +
   customerExperienceOpportunity * 0.22 +
   salesOpportunity             * 0.18 +
   marketingOpportunity         * 0.14 +
   operationsOpportunity        * 0.14 +
   dataAnalyticsOpportunity     * 0.10) * 0.92
);
```

Weights sum to 1.00, then a flat **× 0.92 haircut** — a deliberate conservatism factor.

The financial model is similarly explicit (`:47-52`):
```ts
const weeklyRepHours = data.estimatedHoursSpentRepetitiveWeekly || 15;
const estimatedMonthlyHoursSaved = Math.round(weeklyRepHours * 3.8);  // 3.8, not 4.33
const estimatedMonthlySavingsNPR =
  `NPR ${(estimatedMonthlyHoursSaved * 350).toLocaleString()} - NPR ${(estimatedMonthlyHoursSaved * 600).toLocaleString()}/month`;
```
Weeks-per-month of 3.8 rather than 4.33 is a deliberate ~12% haircut. Rates of NPR 350–600/hour are hard-coded constants with no market data behind them.

**Achievable range: [38, 84].** Both ends are unreachable by construction — the minimum is the sum of all floors times 0.92, the maximum the sum of all ceilings times 0.92. Consequently the client-side defaults of `|| 78` (`app/reports/ai-opportunity/[id]/page.tsx:322`) and `|| 80` (`app/ai-assessment/page.tsx:696`) are **dead values outside the possible range**.

Defects in the flag logic:
- `data.manualDataEntry.length > 5` (`:69`) is **always true**, because the Zod default is a 60-character string with no `min()` constraint. `operationsOpportunity` can therefore only ever be 70 or 94.
- `currentAutomation.includes('None')` and `currentBusinessChallenges.includes('Missed')` (`:65-66`) are **case-sensitive**, while sibling checks use `.toLowerCase()`. Input `"none"` silently scores as "has automation".
- `monthlyLeadVolume.includes('100')` is a substring match that fires for `"100-500"` but not `"50-150"`.

#### 4.2.2 The Gemini call and its configuration gaps

```ts
// app/api/ai-assessments/route.ts:84-86, 177-183
if (process.env.GEMINI_API_KEY) {
  const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });
  // ... prompt built at :87-175 ...
  const response = await ai.models.generateContent({
    model: 'gemini-2.5-flash',
    config: { responseMimeType: 'application/json' },
  });
  if (response.text) { generatedAnalysis = JSON.parse(response.text); }
}
```

That single `config` object is the entire LLM configuration. It is missing:

| Omission | Consequence |
|---|---|
| `responseSchema` | No server-side structural guarantee. The entire ~40-field JSON contract rests on prompt prose compliance. |
| `maxOutputTokens` | The nested response (7 analysis blocks, 4–6 matrix rows, 3 roadmap phases, 3 solution objects) can be **silently truncated**. Truncated JSON throws on parse, is caught, and silently downgrades to the fallback. |
| `temperature`, `topP` | SDK defaults. For a client-facing advisory document, non-determinism is undesirable. |
| `abortSignal` / timeout | An unbounded network call inside a request with no `maxDuration` export. |
| Retry / backoff | One transient failure permanently degrades that request to template copy. |
| Model fallback | `gemini-2.5-flash` is a bare string literal — no env override, no version alias, no secondary model. |
| `systemInstruction` | Role is set in the user prompt instead, where it is more susceptible to injection. |

Two further structural issues:

- **Client instantiated per request** (`:86`) rather than as a module singleton, adding handshake overhead to every call.
- **Prompt injection surface.** The entire Zod-parsed payload is injected via `JSON.stringify(data, null, 2)` at `:92`, including fully attacker-controlled free-text fields such as `commonCustomerQuestions`, `repetitiveEmployeeTasks`, and `currentBusinessChallenges`. Zod imposes **no length caps** on these.

#### 4.2.3 The silent fallback — the critical design flaw

```mermaid
flowchart TD
    A["Zod-validated payload"] --> B["Compute 6 opportunity scores<br/>+ readiness score<br/>(heuristic, deterministic)"]
    B --> C{"GEMINI_API_KEY set?"}
    C -->|Yes| D["Gemini 2.5 Flash<br/>narrative analysis"]
    C -->|No| F["Hard-coded fallback<br/>prose template"]
    D --> E["JSON.parse — no local try/catch"]
    E -->|valid JSON| G["Generated narrative"]
    E -->|throws<br/>(truncation, malformed)| F
    D -->|network error<br/>429, timeout| F
    F --> H["Persisted record"]
    G --> H
    H --> I["API response + 12-section report"]
    I --> J["User sees a tailored report"]

    style F stroke-width:3px
    style J stroke-width:2px
    F -.->|"no provenance flag<br/>no model name<br/>no timestamp"| I
```

The critical observation: **the numeric scores are computed before and independently of the LLM.** Therefore the Gemini path and the fallback path produce *identical scores* with *completely different prose*. Nothing in the response, the database record, or the rendered report indicates which path executed.

The fallback (`:193-211`) is a fully hard-coded object using only four dynamic interpolations. It asserts statistics as fact:

- *"Inquiries responded to within 5 minutes convert at 7x higher rates"*
- *"Estimated 20% to 35% of off-hours inquiries abandon"*
- *"autonomously resolve 60-70% of routine inquiries"*
- *"Standard quotes take 4 to 24 hours"*

**Failure scenario.** A visitor in Ohio submits a dental clinic profile with no `GEMINI_API_KEY` configured in the environment. The platform returns a fully generic Nepali-SME template — complete with NPR-denominated savings figures — presented as a "tailored AI adoption roadmap" for that specific clinic. Nothing in the UI, the API response, or the database distinguishes this from a genuine Gemini-generated analysis.

The remediation in §10.6 adds a provenance field, which converts a silent integrity failure into a visible, auditable state.

#### 4.2.4 Prompt injection and unescaped JSON-LD

Two injection surfaces exist in the platform.

The first is the LLM prompt described above (`ai-assessments/route.ts:92`).

The second is **structured data injection**. All seven JSON-LD call sites use unescaped injection:

```tsx
<script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(schema) }} />
```
`app/layout.tsx:137-149`; `app/ai-assessment/layout.tsx:52-59`; `app/digital-visibility-check/layout.tsx:52-59`

The standard mitigation — replacing `<` with `<` to prevent `</script>` breakout — is absent. Today every input to these graphs is either static or admin-authored, so this is **latent rather than active**. It becomes live the moment an admin-authored field containing `<` is rendered into a schema, which is why §10.8 treats it as a preventive fix rather than an incident.

### 4.3 The broken client↔API contract

This is the highest-impact correctness defect in the platform, because it silently degrades the product's primary conversion path on **every single submission**.

`app/digital-visibility-check/page.tsx:89-102` sends:
```ts
body: JSON.stringify({
  businessName, websiteUrl, industry, cityRegion, targetService,
  primarySearchKeywords,        // ← route expects `primarySearchTerm`
  competitorUrls,               // ← comma-separated STRING; route expects array or `competitorUrl`
  googleBusinessProfileUrl,     // ← route expects `gbpUrl`
  socialMediaUrls,              // ← STRING; route expects `socialUrls` OBJECT
  contactName, contactEmail, contactPhone,
})
```

`app/api/analyze-website/route.ts:16-26` destructures:
```ts
const {
  businessName, websiteUrl, industry, cityRegion, targetService,
  primarySearchTerm,            // ← never sent
  competitorUrls, competitorUrl,// ← `Array.isArray(string)` is false
  gbpUrl,                       // ← never sent
  socialUrls,                   // ← never sent (and wrong type)
  contactName, contactEmail, contactPhone, requestFullReport,
} = body;
```

There is **no schema validation** in this route (unlike `ai-assessments`), so unmatched keys are discarded without error. The consequence chain:

| Field sent | Field read | Downstream effect |
|---|---|---|
| `primarySearchKeywords` | `primarySearchTerm` | `targetTerm` is `''` → all four keyword booleans permanently false → `+10` search and `+10` content never awarded |
| `googleBusinessProfileUrl` | `gbpUrl` | `undefined` → **−25 localVisibility**; report always claims "GBP not connected" |
| `socialMediaUrls` | `socialUrls` (object) | `undefined` → `socialInputsProvided` false → **−20 digitalPresence** |
| `competitorUrls` (string) | array or `competitorUrl` | `Array.isArray` fails → `cleanCompetitors = []` → synthetic benchmark row always used |

**Net effect: a user who completes every field on the form still loses approximately 45 raw score points**, and every report contains a fabricated competitor comparison. Because the UI renders a confident result either way, no one — including the agency's own staff reviewing the report with a client — has any signal that the input was discarded.

The fix is a field rename on one side of the boundary, ideally with a shared Zod schema so the two can never drift again (§10.3).

### 4.4 Engine comparison

| Dimension | Engine 1 — Visibility | Engine 2 — AI Assessment |
|---|---|---|
| Lines | 790 | 290 |
| Uses an LLM | **No** | Yes (`gemini-2.5-flash`) |
| Input validation | **None** | Zod, 25 fields |
| Scoring | 7 weighted categories, `[23, 100]` | 6 weighted categories, `[38, 84]` |
| Determinism | Fully deterministic | Scores deterministic; **prose is not** |
| External network calls | 3 per audit (page, robots, sitemap) | 1 (Gemini), unbounded duration |
| Auth on read/write | None | None |
| ID format | `audit-<ms>-<6 base36>` | `ai-assess-<ms>-<6 base36>` |
| Failure visibility | Silent — returns a full report on fetch failure | Silent — falls back to template with no flag |
| Stores PII | Yes, cleartext | Yes, cleartext |

Both ID formats are `Date.now()` plus six base36 characters. These are **enumerable**, which compounds the missing-authentication finding in §8: report IDs are not secrets, and neither are they unpredictable enough to serve as bearer tokens even if they were treated as such.

---

## 5. Technology Decisions & Trade-offs

Every significant architectural choice in the platform, with its rationale, benefit, and cost.

| # | Decision | Rationale | Advantage | Disadvantage |
|---|---|---|---|---|
| 1 | **Next.js 15 App Router, `output: 'standalone'`** | Single-deployable server + static export in one framework; matches the Cloud Run / AI Studio hosting target. | One deploy artifact serves both the marketing site and the API. Excellent code splitting, image optimisation, and metadata ergonomics. Standalone output suits container platforms. | No explicit rendering strategy is declared anywhere, so caching behaviour is implicit and data-dependent rather than designed. |
| 2 | **Regex-based scraping instead of an LLM or a DOM parser** (`analyze-website/route.ts`) | Deterministic, free, fast, reproducible, auditable. | Zero marginal cost per audit; no model dependency; identical output for identical input; trivially explainable to a client. | Structurally fragile — every heuristic breaks on markup that does not match its pattern. `<script>` contamination pollutes body-text signals. Cannot understand rendered DOM, JS-injected content, or intent. |
| 3 | **LLM for narrative, heuristics for numbers** (`ai-assessments/route.ts`) | Scores must be stable and defensible; prose benefits from generation. | Scores remain reproducible even when the model is unavailable. Narrative quality exceeds what templates can produce. | Creates the divergence in §4.2.3: the fallback silently substitutes entirely different prose for identical scores. No provenance field records which ran. |
| 4 | **Neon serverless PostgreSQL via `pg.Pool`** | Zero-ops managed Postgres with branching; the `@neondatabase/serverless` driver is a dependency but **unused**. | No connection management to write; pooled connections with a 10-connection cap; real relational querying available. | The installed serverless driver is dead weight. A conventional `Pool` against a serverless database is less efficient than the driver's HTTP interface. `ssl.rejectUnauthorized: false` (`lib/db.ts:685`) weakens TLS verification. |
| 5 | **Schema bootstrap in application code** (`lib/db.ts:701-947`) | Zero-migration-step deployment — the database self-heals on first request. | Genuinely useful for a small team; no migration tooling to learn or run. | DDL executes on every cold start. Not concurrency-safe. **Swallows its own errors**, converting a clear initialisation failure into a confusing downstream query error. Schema ownership is split between `lib/db.ts` and a stale, contradictory `lib/schema.sql`. |
| 6 | **JSONB blob + indexed scalar columns** | Schema-free evolution of rich diagnostic output. | New diagnostic fields require no migration. Queryable columns remain available for list views. | The database cannot answer aggregate questions about diagnostic content. No JSONB path indexes. The admin dashboard must load entire tables into memory to render lists. |
| 7 | **"JSONB blob" persistence of full records** | Single-round-trip read; no joins. | One query returns a complete report. | AI assessments store the record **twice** (`raw_submission` and `opportunities`, `lib/db.ts:1342-1343`), roughly doubling storage for that table. |
| 8 | **Single 1,843-line `lib/db.ts` repository** | One place for all persistence; no ORM. | No ORM dependency; full SQL control; typed interfaces exported for the whole domain; readable in one sitting. | Every entity's types, mappers, seed data, and DDL share one file. Seed data (`INITIAL_CASE_STUDIES`, `INITIAL_INSIGHTS`, `INITIAL_TESTIMONIALS` — `lib/db.ts:440-666`) is 226 lines of it. |
| 9 | **In-memory rate limiting** (`lib/ssrf.ts:8-24`) | No Redis or external store required. | Zero infrastructure. Appropriate for a single-instance deployment. | Per-process only — on a multi-instance or serverless deploy the effective limit is **N × 10/min**. Keyed on `x-forwarded-for`, which is client-spoofable. The map is **never evicted**, so it grows unboundedly with unique IPs. |
| 10 | **`window.print()` as the PDF feature** | No PDF library, no headless browser, no server-side rendering. | Zero cost, zero dependencies, and the visitor keeps a real, high-quality document. The A4 print stylesheet is genuinely well built (`app/globals.css:38-69`). | Relies on the visitor's browser save-as-PDF. No server-side artifact, so `pdfDownloaded` analytics are unverifiable. No `.print-page-break` is ever applied, so pagination is uncontrolled. |
| 11 | **Hand-rolled JSON-LD builders** (`lib/seo.ts`) | Full control over entity graph shape; no schema library. | Impressive coverage: Organization, LocalBusiness, WebSite, Service, FAQPage, BreadcrumbList, Article, WebApplication. No dependency. | `getServiceSchema` and the two tool layouts embed a **duplicate inline `LocalBusiness`** rather than referencing the root `#localbusiness` `@id`, so the graph contains three disconnected copies of the same business entity. |
| 12 | **`picsum.photos` / Unsplash / Cloudinary as image sources** | Zero asset management; fast to build. | No image pipeline to maintain. All images go through `next/image` with `alt` text. | Hero imagery is generic stock photography on a site whose entire value proposition is local Nepalese credibility. |
| 13 | **No web font** | One less network round-trip; native rendering. | Faster first paint; no FOUT. | The site renders in the platform UI sans stack. Typography is the single largest carrier of perceived design quality, and 442 `font-bold` / 52 `font-black` declarations are doing the work that a type system should. |
| 14 | **`eslint.ignoreDuringBuilds: true`** (`next.config.ts:5-7`) | Avoid AI Studio build failures on lint warnings. | Builds do not break on style nits. | Lint errors can never fail a build. Combined with `eslint-config-next@16.0.8` against Next `^15.4.9` (a major-version mismatch) and a duplicate `.eslintrc.json` alongside `eslint.config.mjs`, the lint setup is non-functional. `package.json:9` also defines `next clean`, which is not a valid Next CLI command. |

---

## 6. Advantages

This section records what the platform does genuinely well, so that remediation in §10 does not inadvertently destroy it.

### 6.1 Security work that is actually good

**The SSRF defence in `lib/ssrf.ts` is the strongest engineering in the codebase.** Because the platform fetches user-supplied URLs, SSRF is its most serious *inherent* risk, and it is handled in layers:

1. Protocol allow-list (`http:` / `https:` only) — `ssrf.ts:87-89`
2. **Userinfo rejection** — blocks `http://user:pass@evil.com` credential-confusion attacks — `ssrf.ts:92-94`
3. Cloud metadata hostname block-list: `metadata.google.internal`, `169.254.169.254`, `instance-data`, and others — `ssrf.ts:98-106`
4. Suffix rejection for `*.internal` and `*.local` — `ssrf.ts:108-110`
5. Full IPv4 private-range rejection: loopback, RFC1918 (all three blocks), link-local, `0.0.0.0/8`, multicast, reserved — `ssrf.ts:26-48`
6. IPv6 loopback, unspecified, link-local, and unique-local rejection — `ssrf.ts:50-61`
7. **DNS resolution with post-resolution IP validation** — resolves both A and AAAA records and rejects any that land in private space — `ssrf.ts:120-138`

Most implementations stop at step 4. Steps 5–7 are the work that actually prevents cloud-credential theft. Two residual gaps remain and are documented in §8.

**Additional positive controls:**
- Response size capped at 1.5 MB (`:81-83`) and an 8-second `AbortController` (`:58-59`) bound the outbound request.
- User-agent is honest and identifies itself (`:65-66`).
- Zod validation is present on `/api/leads` and `/api/ai-assessments`.
- No raw `<img>` tags anywhere — 100% `next/image` usage, all with `alt` attributes.
- Analytics metadata is deliberately PII-free on both audit completion paths (`:770-775`; `ai-assessments/route.ts:270-279`).
- No `@ts-ignore`, no lint suppressions, and no `TODO`/`FIXME` in application source.
- `typescript.ignoreBuildErrors: false` (`next.config.ts:8-10`) — **type errors do fail the build**, which is stricter than most production Next.js setups.

### 6.2 The print stylesheet

`app/globals.css:38-69` is a considered piece of work:

```css
@page { size: A4 portrait; margin: 10mm; }
@media print {
  html, body { background-color:#fff !important; color:#171717 !important;
               print-color-adjust:exact !important; }
  .no-print { display: none !important; }
  .print-only { display: block !important; }
  .print-avoid-break, .rounded-2xl, .rounded-3xl { break-inside: avoid !important; }
}
```

`print-color-adjust: exact` preserves brand fills through the print pipeline. Every report `<section>` carries `print-avoid-break` (7 sections in the AI report, 13 in the visibility report) so cards do not split across pages, and the action bar and modal are correctly marked `no-print`. This is what makes §5 decision 10 viable at all.

### 6.3 Structured-data coverage

`lib/seo.ts` implements eight schema builders — Organisation, LocalBusiness + ProfessionalService, WebSite with `SearchAction`, Service, FAQPage, BreadcrumbList, and Article — and the root layout injects four of them site-wide (`app/layout.tsx:135-150`). For an SME competing against larger agencies, this is genuinely above-average. The `DEFAULT_AEO_FAQS` set (`lib/seo.ts:48-89`) is written specifically for AI answer-engine citation, including an explicit definitional answer to "What is the difference between SEO, AEO, and GEO?" — the kind of query an LLM is likely to surface.

### 6.4 Cost and operational profile

The platform runs at close to zero marginal infrastructure cost per audit. Engine 1 makes three HTTP requests and one database write. There is no per-audit model spend, no headless browser, no queue, no object storage, no email provider. `db.getPool()` is exposed for ad-hoc querying, and the pool is correctly registered on `globalThis` in non-production to survive hot reloads (`lib/db.ts:692-694`).

### 6.5 Domain modelling

The TypeScript interfaces in `lib/db.ts:3-435` are detailed, well-named, and genuinely useful — `VisibilityAudit` alone models nine nested detail objects with precise optionality. They serve as the de facto domain specification and make the report renderers readable. Strict mode is on, `skipLibCheck` is appropriately enabled, and path aliases are configured.

### 6.6 Conversion design

The commercial funnel is well designed. Every lead-capture path writes both a `leads` row and an `analytics_events` row with `ctaIdentifier` attribution. The WhatsApp integration (`lib/whatsapp.ts`) builds context-aware pre-filled messages — a completed assessment message includes the business name and score, which materially improves reply rates. The floating WhatsApp CTA, the report-page share flow, and the "open full report" upsell are consistent across both tools.

---

## 7. Disadvantages & Risks

### 7.1 Correctness

**7.1.1 The client↔API contract failure** (§4.3) is the most consequential correctness defect in the platform. Four inputs are discarded on every submission, costing roughly 45 raw score points and guaranteeing a fabricated competitor row in every report ever generated.

**7.1.2 Scoring defects.**

- `analyze-website/route.ts:328` — the `else if (responseTimeMs < 2500)` branch is unguarded, awarding +8 for a failed request.
- `:136` — `pageSizeKb` is a character count of raw HTML including `<script>` and JSON-LD, then used to judge content substance, and with **inverted thresholds** in two places: `>= 15` = "not thin" (`:360`) versus `< 15` = "High thin risk" (`:640`).
- `:696` — `structuredDataScore = detectedSchemas.length * 20` is unclamped and can render above 100%.
- `:688` — `conversationalIntentScore` is arithmetically identical to `aeoScore`, presented as an independent metric.
- `:688, 687` — several "derived" values are slices of other arrays or constants (§4.1.6).
- `:515-527` vs `:530-542` — the two competitor-comparison code paths use different thresholds and inverted polarity; `localVisibility` can never be "Ahead" on the real-URL path.
- `:159` — the canonical-tag regex requires `rel` before `href`; reversed attribute order yields a false negative.
- `:104` — `hasRobotsTxt` is detected by searching the body for `user-agent:`, so a 200 soft-404 page passes.
- `:178` — `alt=""` counts as a present alt attribute.

**7.1.3 Lead-capture defects.**

- `lib/db.ts:1130-1131` — `addLead` writes `leadData.sourcePage` into **both** the `source` and `source_page` columns. The real source (e.g. `contact_page_form`) is never stored, corrupting attribution reporting.
- `app/api/leads/route.ts:5-18` — the schema has no `industry` key and `contact/page.tsx:40` sends one. Zod strips unknown keys, so `industry` is **always** written as `NULL`.
- `leads.utm_source` / `utm_medium` / `utm_campaign` are declared in the interface and in `schema.sql` but never persisted.
- `analyze-website/route.ts:757-768` — when a user supplies only a phone number, the shared mailbox `audit-request@infoclick.com.np` is silently written as the lead's email address.

**7.1.4 UI defects.**

- `app/globals.css:1` — no `@plugin` directive exists, so `tw-animate-css` and `@tailwindcss/typography` are installed but never registered. All 7 `animate-in fade-in` entrances are **silent no-ops** (`app/admin-route-login/page.tsx:1680, 2471`; `app/ai-assessment/page.tsx:636`; `app/digital-visibility-check/page.tsx:422`; both report pages; `components/layout/floating-whatsapp.tsx:32`), and `prose prose-neutral` at `app/insights/[slug]/page.tsx:111` **generates no styles at all** — every database-authored insight article renders as an unstyled wall of text. This is the single highest-impact visual defect in the repository.
- `app/ai-assessment/page.tsx:31` vs `:217` — the `industry` state default is `"Schools & Education"` but the matching `<option>` value is `"Schools &amp; Educational Institutions"`. No option matches, so the select **renders blank on first paint**.
- `app/digital-visibility-check/page.tsx:48, 435` — `responseTimeMs` is read from the top level of the response but the API nests it under `technicalDetails`, so the latency line never renders.
- `app/ai-assessment/page.tsx:635-755` — the step-5 results view **never displays the six opportunity scores**, even though the API returns them. Only the hours-saved gauge is shown, discarding the most differentiated output of the engine.

### 7.2 Reporting integrity

This is the most commercially damaging category, because the platform's differentiator is *trustworthy automated assessment*.

**7.2.1 Fabricated competitor benchmark** (§4.1.5). Rendered as a real comparison table (`:877-964`) with no indication that competitors were never fetched. A client comparing this table against their actual competitors would be demonstrably wrong.

**7.2.2 Phantom email delivery.** `app/api/reports/send-email/route.ts` contains no mail dependency — none is installed — yet returns:

```ts
deliveryDetails: { subject, recipient: recipientEmail, timestamp: now, status: 'DELIVERED' }
```
`send-email/route.ts:78`

Nothing is queued and nothing is sent. The endpoint's only effects are flipping a database boolean, logging an analytics event containing the recipient's email in plaintext (`:68`), and asserting success. The client then renders *"Report delivered successfully to {email}!"* Separately, `app/digital-visibility-check/page.tsx:157-164` sets `reportSuccess = true` on **all** outcomes — `res.ok`, non-`ok`, and thrown exceptions — so a failed request is reported as success. The report modals advertise *"a secure report link"* (`ai-opportunity/[id]/page.tsx:816`) and *"a secure, permanent link"* (`visibility/[id]/page.tsx:1142`); **no such link is ever generated or transmitted.**

**7.2.3 Unlabelled hard-coded claims.** A non-exhaustive sample of strings presented as measured findings:

| Claim | Reality | Ref |
|---|---|---|
| `"SSL Secure"` in `trustSignals` | Printed even on plain HTTP | `:718` |
| `"Cross-linking detected in site structure"` | Asserted when zero links exist | `:729` |
| `"Over 80% of local mobile buyers convert via WhatsApp"` | Hard-coded; no source | `:469` |
| Fixed `serpFeatures` for every website | Hard-coded | `:672` |
| `"full GBP API sync available on request"` | Not implemented | `:655` |
| `"7x higher rates"`, `"20% to 35% abandon"`, `"60-70% of routine inquiries"` | Unsourced statistics in LLM-fallback prose | `ai-assessments/route.ts:196-210` |
| `"NPR 25,000 - 45,000/mo"` | Fabricated financial figure used as a display default | `ai-opportunity/[id]/page.tsx:371` |
| Score defaults `78` / `80` | **Outside** the achievable `[38, 84]` range, so unreachable | `ai-opportunity/[id]/page.tsx:322`; `ai-assessment/page.tsx:696` |
| `"Document Hash: {audit.id}"` | An ID labelled as a hash | `visibility/[id]/page.tsx:1117` |

**7.2.4 Provenance tags that do not mean what they claim** (§4.1.4).

**7.2.5 Discarded LLM output.** `app/reports/ai-opportunity/[id]/page.tsx:210` assigns `const roadmap = assessment.roadmap;` and **never reads it**. The Gemini-generated three-phase roadmap is discarded in favour of hard-coded phase copy at `:672-741`. The platform pays for model calls whose most expensive output is thrown away.

**7.2.6 Missing empty states.** The opportunity priority matrix renders an empty table body with a header and no fallback if the array is empty (`ai-opportunity/[id]/page.tsx:209, 609`).

### 7.3 Architecture and maintainability

**7.3.1 No rendering strategy.** Zero occurrences of `generateStaticParams`, `revalidate`, `dynamic`, or `runtime` across the entire `app/` tree. Database-backed Server Components have no dynamic API call, so Next may prerender them at build time and serve stale CMS content indefinitely, while detail routes re-query per request with no revalidation. The behaviour is data-dependent and unintentional.

**7.3.2 Monolithic components.** `app/admin-route-login/page.tsx` is **2,597 lines** in a single component — authentication, a lead CRM, three content editors, user management, and a diagnostics panel — largely `any`-typed, with no code splitting. The two report pages are 1,215 and 888 lines, also single components.

**7.3.3 Client-side data waterfalls.** Both report routes ship a full React bundle and then block on a `useEffect` round-trip. The reports are the highest-value sales artifacts and simultaneously the least crawlable and most visibly janky.

**7.3.4 No error boundaries.** No `error.tsx`, `global-error.tsx`, `not-found.tsx`, or `loading.tsx` anywhere. A thrown `TypeError` in the report renderers yields the generic Next.js error page with no signal beyond server logs. Such throws are reachable: optional-chaining gaps such as `tech?.imageAltText.missing` (`visibility/[id]/page.tsx:553, 555, 559`), `tech?.securityHeaders.hsts` (`:567-568`), and `audit.strengths[0]?.` (`:394`) guard only the first level, so a missing nested object throws and blanks the entire report.

**7.3.5 Duplicated source of truth.** `COMPANY_INFO` exists centrally in `lib/seo.ts:1-46` but `components/layout/footer.tsx:21-33` hardcodes its own copy and `app/llms.txt/route.ts` a third. `INDUSTRIES_DATA` is redeclared on the homepage. Two hand-maintained FAQ sets exist (`HOMEPAGE_FAQS` in `app/page.tsx` and `DEFAULT_AEO_FAQS` in `lib/seo.ts`). A NAP change requires edits in three places with no compile-time guard.

**7.3.6 Dead code and configuration drift.**

| Item | Detail |
|---|---|
| `POST /api/visibility-audits` | Never invoked by any client. Its user-facing promise of a human-compiled emailed report (`route.ts:44-47`) has **no live code path**. |
| `scratch.ts` | Byte-identical duplicate of `app/api/ai-assessments/route.ts` (290 lines). |
| `scripts/test-crud-10x.ts` | A one-off load script at project root. |
| `lib/schema.sql` | Stale and contradicted by the live DDL. |
| `public/llms.txt` | Byte-for-byte duplicate of the generated route, free to drift. |
| `.eslintrc.json` + `eslint.config.mjs` | Both present, conflicting formats. |
| `package.json:9` | `next clean` is not a valid Next CLI command. |
| `bun.lock` + `package-lock.json` | Two lockfiles for two package managers. |
| `.env.example` | Sample WhatsApp number `9779852028888` vs. production fallback `9779823644144` (`lib/whatsapp.ts:1`). |
| `@neondatabase/serverless` | Installed but never imported. |
| `@hookform/resolvers` | Installed but never imported. |
| `y`, `y.pub` | Committed key material — see §8. |

### 7.4 Performance

- **Worst-case request duration** for a visibility audit is approximately 8s + 3.5s + 3.5s + database time, with no `maxDuration` export. This exceeds the default function ceiling on several serverless hosts.
- **The audit response is inflated ~2.5×** — the full record is returned three times, once nested under `audit` and again spread across every top-level key (`:777-782`).
- **The admin dashboard loads entire tables.** `GET /api/admin/data` calls seven `getAll()` methods and returns every lead, audit, assessment, testimonial, case study, insight, and user in one response with no pagination.
- **`db.getUsers()` is exposed unauthenticated** via the same endpoint.
- Report tables have no pagination controls.
- No `next/image` `sizes` tuning, no font optimisation, and 16,498 bytes of lockfile churn in the working tree.

---

## 8. Security Analysis

### 8.1 Threat model

The platform processes **personally identifiable information about members of the public** — names, email addresses, phone numbers, and business details submitted by prospective customers through two public forms — and stores it in cleartext. It fetches arbitrary attacker-supplied URLs from a server with access to a cloud metadata endpoint. It exposes an administrative CMS. It calls an external LLM with user-supplied text.

| Asset | Sensitivity | Exposure |
|---|---|---|
| Customer PII corpus | **High** | Unauthenticated via `/api/admin/data` |
| Admin CMS content | Medium | Unauthenticated read/write/delete |
| User records | Medium | Unauthenticated read/write/delete |
| Neon database credential | **Critical** | Committed to git in source |
| SSH private key | **Critical** | Committed to git, not gitignored |
| LLM prompt integrity | Medium | Unsanitised user text |
| Server-side network | High | Fetched by user-supplied URL |

### 8.2 Trust boundaries and authentication

```mermaid
flowchart TB
    subgraph Public["Public Internet — untrusted"]
        U1["Anonymous visitor<br/>(diagnostic forms)"]
        U2["Anyone with a report ID<br/>(enumerable)"]
        U3["Anyone at all"]
    end

    subgraph Edge["Next.js route handlers"]
        R1["POST /api/leads<br/>POST /api/ai-assessments<br/>POST /api/analyze-website"]
        R2["GET/PATCH/DELETE<br/>/api/visibility-audits/[id]<br/>/api/ai-assessments/[id]"]
        R3["GET /api/admin/data<br/>+ 5 CRUD routes<br/>(28 endpoints)"]
        AUTH["POST /api/admin/auth<br/>SETS cookie"]
    end

    subgraph Assets["Protected assets"]
        PII[("Full PII corpus<br/>leads · audits · assessments")]
        CMS[("CMS content<br/>+ user records")]
        ENV[("Neon credential<br/>in git history")]
    end

    U1 --> R1
    U2 --> R2
    U3 --> R3
    U1 -.->|"can also reach"| R3
    U1 -.->|"can also reach"| R2
    R1 --> PII
    R2 -->|"NO CHECK"| PII
    R3 -->|"NO CHECK"| PII
    R3 -->|"NO CHECK — PUT/DELETE"| CMS
    AUTH -.->|"sets a cookie nobody reads"| R3
    ENV --> PII

    style R2 stroke-width:3px,stroke:#c00
    style R3 stroke-width:4px,stroke:#c00
    style PII stroke-width:2px
    style CMS stroke-width:2px
    style ENV stroke-width:3px
    style AUTH stroke-dasharray: 4 3
```

**The diagram's central fact: `POST /api/admin/auth` is the only file in the entire API surface that references authentication — and it only *sets* a cookie. No code anywhere reads it.**

This was verified programmatically across all 15 route files:

| Route | Methods | Auth references |
|---|---|---|
| `admin/auth` | POST | 2 *(sets cookie — reads none)* |
| `admin/case-studies` | GET, POST, PUT, DELETE | **0** |
| `admin/contacts` | GET, POST, PUT, DELETE | **0** |
| `admin/data` | GET, PATCH | **0** |
| `admin/insights` | GET, POST, PUT, DELETE | **0** |
| `admin/testimonials` | GET, POST, PUT, DELETE | **0** |
| `admin/users` | GET, POST, PUT, DELETE | **0** |
| `ai-assessments/[id]` | GET, PATCH, DELETE | **0** |
| `ai-assessments` | POST | 0 |
| `analytics/track` | POST | 0 |
| `analyze-website` | POST | 0 |
| `leads` | POST | 0 |
| `reports/send-email` | POST | 0 |
| `visibility-audits/[id]` | GET, PATCH, DELETE | **0** |
| `visibility-audits` | POST | 0 |

**Thirty-four endpoints. Zero enforcement.**

`app/robots.ts:11` disallows `/api/admin/` — but `robots.txt` controls crawler politeness, not access control. `curl https://<host>/api/admin/data` returns the complete customer database.

### 8.3 Findings register — security

| ID | Sev | Finding | Evidence | Remediation |
|---|---|---|---|---|
| SEC-01 | **Critical** | Unencrypted OpenSSH private key committed to git and not gitignored. Key comment identifies an owner email. | `./y` (419 B), `./y.pub`, `.gitignore` | Rotate the key at the provider, then `git rm --cached`, add to `.gitignore`, and purge from history (§10.1) |
| SEC-02 | **Critical** | Live Neon PostgreSQL connection string with embedded password as the hard-coded `DATABASE_URL` fallback. *Password redacted.* | `lib/db.ts:671-673` | Rotate the credential; remove the fallback so the app fails fast without `DATABASE_URL` (§10.1) |
| SEC-03 | **Critical** | No authentication on any endpoint. `/api/admin/data` returns every lead, audit, and assessment including cleartext email and phone. | all `app/api/admin/*` | Add session verification middleware; gate all admin and `[id]` routes (§10.2) |
| SEC-04 | **High** | Unauthenticated `PUT`/`DELETE` on case studies, insights, testimonials, contacts, and users. | `app/api/admin/*/route.ts` | Same as SEC-03 |
| SEC-05 | **High** | Unauthenticated `PATCH`/`DELETE` on both customer report routes — arbitrary mutation of customer PII records, including `contactEmail`. Bodies are entirely unvalidated. | `ai-assessments/[id]/route.ts:20-51`; `visibility-audits/[id]/route.ts:20-51` | Require a per-report unguessable token **and** validate the body with Zod (§10.2) |
| SEC-06 | **High** | Report IDs are `Date.now()` + 6 base36 characters — enumerable. They are the only thing standing between a PII record and the public. | `lib/db.ts:1223, 1317` | Use `crypto.randomUUID()` or a 128-bit random token (§10.2) |
| SEC-07 | **High** | `/reports/**` is neither blocked in `robots.ts` nor `noindex`ed, yet contains `contactEmail`, `contactPhone`, and the full business profile. | `app/robots.ts:11`; report pages | Add `noindex` per report; add `/reports/` to the disallow list (§10.7) |
| SEC-08 | **Medium** | Session cookie is the constant string `'authenticated'` with no signature, no `secure`, no `sameSite`. A second hard-coded backdoor password `'infoclick2026'` is accepted alongside `ADMIN_PASSWORD`. | `app/api/admin/auth/route.ts:6-19` | Signed, `httpOnly` + `secure` + `sameSite=lax` session; remove the fallback password; add rate limiting |
| SEC-09 | **Medium** | SSRF time-of-check/time-of-use gap: DNS is validated, then `fetch` resolves independently, permitting DNS rebinding. `redirect: 'follow'` performs no per-hop revalidation, so a public URL can 302 to `169.254.169.254`. | `analyze-website/route.ts:62-70`; `lib/ssrf.ts:120-138` | Set `redirect: 'manual'` and re-validate each hop; or pin the validated IP (§10.4) |
| SEC-10 | **Medium** | Rate limiting is per-process, keyed on the client-spoofable `x-forwarded-for` header (no `x-real-ip` fallback), and the map is never evicted — unbounded memory growth per unique IP. Effective limit is N×10/min on N instances. | `analyze-website/route.ts:7`; `lib/ssrf.ts:8-24` | Trust-proxy-aware client IP; shared store; periodic eviction (§10.4) |
| SEC-11 | **Medium** | Unescaped JSON-LD injection at 7 call sites. `JSON.stringify` output is not `<`-escaped, so a `</script>` sequence in any stored field breaks out. Latent today; live as soon as an admin-authored field contains `<`. | `app/layout.tsx:137-149`; both tool layouts | Apply the standard `<` → `<` escaping (§10.8) |
| SEC-12 | **Medium** | `POST /api/analytics/track` is unauthenticated and accepts arbitrary `eventType` and `metadata` — an unbounded write amplifier against the database. | `app/api/analytics/track/route.ts:4-23` | Allow-list event types; cap metadata size; rate limit |
| SEC-13 | **Medium** | `ssl.rejectUnauthorized: false` disables TLS certificate verification on the database connection. | `lib/db.ts:685` | Remove; use a proper CA bundle |
| SEC-14 | **Low** | `send-email` stores the recipient's email address in plaintext inside `analytics_events.metadata`, duplicating PII into a second table with different access patterns. | `app/api/reports/send-email/route.ts:68` | Store a hash or omit |
| SEC-15 | **Low** | No CSRF protection on state-changing endpoints. Partly mitigated by the absence of cookie-based auth — but that mitigation disappears the moment SEC-03 is fixed. | all mutating routes | SameSite cookies + origin checks **as part of** the SEC-03 fix |
| SEC-16 | **Low** | Zod schemas impose no maximum lengths on free-text fields, which are concatenated into the LLM prompt and stored in JSONB. | `ai-assessments/route.ts:6-32` | Add `.max()` bounds |
| SEC-17 | **Info** | PII is stored in cleartext with no retention policy, no consent gate, and no encryption at rest. | `lib/db.ts:1221-1254, 1315-1348` | Add a retention policy and a privacy notice; consider column-level encryption for phone numbers |

### 8.4 Positive controls

To be explicit about what is already sound: the SSRF defence-in-depth (§6.1), outbound response size and time caps, PII-free analytics metadata on the two audit paths, Zod validation on the two public write endpoints, `httpOnly` on the session cookie, and build-time type checking are all genuinely good. The problem is not the absence of security thinking — it is that the thinking was never applied to the application's own access control.

---

## 9. SEO Analysis

### 9.1 Crawl and index topology

```mermaid
flowchart TD
    subgraph Bots["Crawler entry points"]
        G["Googlebot"]
        AI["GPTBot · ClaudeBot<br/>PerplexityBot · Google-Extended"]
    end

    subgraph Discovery["Discovery surfaces"]
        SM["/sitemap.ts<br/>10 static + DB routes"]
        RT["/robots.ts"]
        LT["/llms.txt<br/>route + static duplicate"]
    end

    subgraph Indexed["Intended index"]
        GOOD["10 marketing routes<br/>Server-rendered · per-route metadata"]
    end

    subgraph Problem["Index problems"]
        REP["/reports/**<br/>client-rendered · no metadata<br/>inherits canonical '/'<br/>NOT disallowed"]
        ADM["/admin-route-login<br/>no metadata · inherits canonical '/'"]
        TOOL["/digital-visibility-check<br/>/ai-assessment<br/>absent from sitemap"]
    end

    G --> SM
    G --> RT
    AI --> RT
    AI --> LT
    SM --> GOOD
    SM -.->|"omits"| TOOL
    RT -->|"allows all"| REP
    RT -->|"disallows"| ADM
    REP --> SPIN["Crawler receives<br/>spinner only"]
    SPIN --> TRAP["Thin-content URL<br/>canonicalised to homepage"]

    style REP stroke-width:3px,stroke:#c00
    style TRAP stroke-width:3px,stroke:#c00
    style SPIN stroke-dasharray: 4 3
    style GOOD stroke-width:2px
```

### 9.2 What is working

- **Per-route metadata** on all 10 marketing routes, with a root title template (`app/layout.tsx:23`).
- **`robots.ts` has a genuinely modern AI-crawler allow-list** covering GPTBot, ChatGPT-User, PerplexityBot, ClaudeBot, Google-Extended, Applebot-Extended, and Amazonbot (`app/robots.ts:14-24`) — ahead of most SME sites, and directly aligned with the platform's AEO/GEO positioning.
- **`/llms.txt` served as a route** with correct `text/plain` and a 24-hour cache header, plus a hand-written entity and capability summary (`app/llms.txt/route.ts`).
- **Nested tool layouts** correctly emit `BreadcrumbList` plus `WebApplication` JSON-LD as Server Components — the right pattern (`app/ai-assessment/layout.tsx`, `app/digital-visibility-check/layout.tsx`).
- **Dynamic metadata on detail routes** via `generateMetadata` for case studies and insights.
- **`hreflang` alternates** (`en`, `ne`, `x-default`) at the root layout.
- **Geo meta tags** emitted alongside `LocalBusiness` schema.
- **Sitemap pulls from the database**, so new case studies and insights are discovered automatically.

### 9.3 Findings register — SEO

| ID | Sev | Finding | Evidence | Remediation |
|---|---|---|---|---|
| SEO-01 | **High** | Report pages are client-rendered with **no `metadata`/`generateMetadata` and no layout**, so they inherit only the root title, description, and `canonical: '/'`. They are not disallowed in `robots.ts` either. Result: potentially large numbers of thin, duplicate URLs — containing customer PII — all canonicalised to the homepage, whose content is a spinner to any non-JS crawler. | both report pages; `app/robots.ts:11` | Add a server-side wrapper with `noindex`, or a `layout.tsx` for `app/reports/` (§10.7) |
| SEO-02 | **High** | `prose` generates no styles because `@tailwindcss/typography` is installed but never registered. Every DB-authored insight article renders as unstyled text — the platform's entire content-marketing surface is visually broken. | `app/globals.css:1`; `app/insights/[slug]/page.tsx:111` | Add `@plugin "@tailwindcss/typography";` (§10.5) |
| SEO-03 | **High** | Root `alternates.canonical: '/'` is inherited by every route lacking its own canonical — both report routes and `/admin-route-login`. This instructs search engines to consolidate those URLs onto the homepage. | `app/layout.tsx` metadata | Remove the root canonical; set it per route (§10.7) |
| SEO-04 | **Medium** | Title template double-appends the brand. The template is `'%s \| Infoclick Digital Solutions'` and six pages already hardcode the company name, producing *"About Us \| Infoclick Digital Solutions \| Infoclick Digital Solutions"*. | `app/layout.tsx:23`; six page files | Remove the hardcoded suffix from page titles (§10.7) |
| SEO-05 | **Medium** | The two primary tool pages are **absent from the sitemap**, and neither is in the AI-crawler allow-list — despite being the platform's main AEO/GEO link targets. | `app/sitemap.ts:10-26`; `app/robots.ts:14-24` | Add both routes to the sitemap and the AI allow-list |
| SEO-06 | **Medium** | Three disconnected copies of the `LocalBusiness` entity. `getServiceSchema` embeds an inline provider rather than referencing the root `#localbusiness` `@id`; both tool layouts embed a third inline copy. Duplicate entities dilute the graph. | `lib/seo.ts:211-232`; both tool layouts | Reference by `@id` (§10.9) |
| SEO-07 | **Medium** | `lastModified: new Date()` on every sitemap entry means each crawl reports every URL as freshly modified — eroding the change signal crawlers use to prioritise recrawls. | `app/sitemap.ts:23, 30` | Use real `updated_at` values from the database |
| SEO-08 | **Medium** | `public/llms.txt` is a byte-for-byte duplicate of the generated route and is not kept in sync automatically. | `app/llms.txt/route.ts`; `public/llms.txt` | Delete the static copy |
| SEO-09 | **Medium** | `WebApplication.offers` declares `price: '0', priceCurrency: 'USD'` for services priced in NPR. | both tool layouts | Use `price: '0'` with `priceCurrency: 'NPR'`, or omit |
| SEO-10 | **Low** | No `aggregateRating` or `Review` schema despite a testimonials page with 5-star ratings — a missed rich-result opportunity. | `lib/seo.ts`; `app/testimonials` | Add `aggregateRating` to the organisation node |
| SEO-11 | **Low** | `getArticleJsonLd` accepts `authorName`/`author` in its parameter type and silently ignores them, always emitting a generic publisher as author. | `lib/seo.ts:262-297` | Honour the parameter |
| SEO-12 | **Low** | No `not-found.tsx`. The default 404 is unstyled and carries no `metadata`, so a mistyped or retired URL produces a bare error page. | project-wide | Add `app/not-found.tsx` |
| SEO-13 | **Info** | `/llms.txt` is listed in `robots.ts` but is not a sitemap entry, and `public/site.webmanifest` declares no `start_url` coherence with the canonical domain. | `app/robots.ts`; `public/site.webmanifest` | Minor consistency pass |

### 9.4 AEO/GEO readiness — an honest assessment

The platform markets itself as specialist in Answer Engine Optimization and Generative Engine Optimization, and it holds two genuine advantages: the AI-crawler allow-list and the citation-oriented `DEFAULT_AEO_FAQS` set.

However, the term "AI-ready" in the **product** (`aeoGeoReadiness`, one of the seven scored categories) measures only four deterministic markup facts: presence of FAQ content, presence of any JSON-LD, more than two question-shaped phrases in tag-stripped text, and a title plus meta description. It is a reasonable heuristic for on-page readiness. It is not an AI analysis, and the report's surrounding narrative implies otherwise.

This distinction is worth preserving carefully in client conversations, because it is defensible: *"we measure whether your site is structured in a way that answer engines can parse and quote"* is a true and valuable claim. *"Our AI analysed your visibility"* is not. §10.6 adds the provenance flag that keeps the distinction honest.

---

## 10. Recommended Improvements

Each recommendation states the problem, the change, the benefit, the effort, and the cost of inaction. Ordered by severity of the underlying problem.

### 10.1 Rotate and purge committed secrets — P0

**Problem.** An unencrypted OpenSSH private key is tracked in git and not gitignored (`./y`, `./y.pub`). A live Neon PostgreSQL connection string with an embedded password is the hard-coded `DATABASE_URL` fallback (`lib/db.ts:671-673`). Both are in git history and therefore already exposed to anyone with repository read access, and to any backup or clone of it.

**Change.**
1. Rotate the SSH key at whatever provider issued it. Treat the current key as compromised — deletion alone does not undo exposure.
2. Rotate the Neon database password and update the deployment secret store.
3. `git rm --cached y y.pub`, add `y`, `y.pub`, and `*.pem` to `.gitignore`.
4. Replace the `lib/db.ts` fallback with a fail-fast guard:
   ```ts
   const connectionString = process.env.DATABASE_URL;
   if (!connectionString) throw new Error('DATABASE_URL is not configured');
   ```
5. Rewrite history to purge the key material (`git filter-repo --invert-paths -- y y.pub`), then force-push and have collaborators re-clone. Coordinate this — it is a destructive operation.
6. Add `gitleaks` or `trufflehog` to CI so this class of leak cannot recur.

**Benefit.** Removes an active, permanent credential exposure; converts a silent architectural hazard into a loud deployment failure. The CI secret scanner prevents recurrence across the whole team, not just this incident.

**Effort.** ~2 hours including coordination. **Cost of inaction:** anyone with repository access — including every former collaborator and every CI system with clone rights — retains the database credential and the private key indefinitely.

### 10.2 Implement real authentication and authorisation — P0

**Problem.** Thirty-four endpoints enforce nothing. The one authentication-related file sets a cookie that no code reads. `/api/admin/data` returns the entire customer database to any caller. Admin CMS content and user records are readable, writable, and deletable by anyone. Both customer-report routes allow unauthenticated `PATCH` and `DELETE` on records containing PII.

**Change.** In three parts.

**(a) A real session.** Replace the constant-string cookie with a signed, server-verified session:
```ts
// Signed: HMAC-SHA256 over payload + expiry, secret from env
response.cookies.set({
  name: 'infoclick_admin_session',
  value: signedToken,
  httpOnly: true,
  secure: true,
  sameSite: 'lax',
  path: '/',
  maxAge: 60 * 60 * 8,
});
```
Hash the admin password with `scrypt` and compare in constant time. Remove the `'infoclick2026'` backdoor fallback (`app/api/admin/auth/route.ts:13`). Add rate limiting to the login endpoint — it is the one endpoint where brute force matters.

**(b) A guard applied everywhere.** One helper, applied to every `/api/admin/*` handler and both `[id]` routes:
```ts
// lib/auth.ts
export async function requireAdmin(req: NextRequest): Promise<Response | null> {
  const session = await verifySession(req.cookies.get('infoclick_admin_session')?.value);
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  return null; // null means "proceed"
}
```
Every protected handler starts with `const denied = await requireAdmin(req); if (denied) return denied;`. Because there are 15 route files, the highest-leverage move is `middleware.ts` with a `matcher` on `/api/admin/:path*` plus the two `[id]` prefixes — one file, one review, complete coverage, and it fails closed by default for anything added later.

**(c) Per-report access tokens.** Report pages must stay publicly viewable by their owner, so guard them with a token rather than a session:
- Generate report IDs with `crypto.randomUUID()` instead of `Date.now()` + 6 base36 characters (`lib/db.ts:1223, 1317`).
- Add a separate `access_token_hash` column; store only the hash.
- Put the token in a signed, expiring link emailed or WhatsApp-shared to the report owner.
- Validate it in `GET`/`PATCH`/`DELETE` on both `[id]` routes.
- Validate all `PATCH` bodies with Zod — currently they are passed straight to `db.update*` with no schema at all.

**Benefit.** Closes a total confidentiality and integrity failure. Converts customer PII from world-readable to owner-only. Removes the enumerable-ID exposure. Because `middleware.ts` fails closed, this also protects routes added in future. The CSRF risk that §8 SEC-015 currently defers is resolved as a natural consequence of `sameSite: 'lax'` plus signed tokens.

**Effort.** §(a)+(b) ≈ 1 day. §(c) ≈ 1 day including the two column additions and link plumbing. **Cost of inaction:** continued exposure of the full customer database, and a platform that cannot lawfully hold PII in a market with data-protection obligations.

### 10.3 Fix the client↔API contract and adopt a shared schema — P0

**Problem.** Four form inputs are sent under names the route does not read (`§4.3`). Every visibility score the platform has produced was computed from an incomplete form, and every report contains a fabricated competitor table.

**Change.** Align the field names, then make drift structurally impossible:

```ts
// lib/contracts/visibility.ts — single source of truth, imported by BOTH sides
export const visibilityRequestSchema = z.object({
  businessName: z.string().min(1),
  websiteUrl: z.string().min(1),
  industry: z.string().optional(),
  cityRegion: z.string().optional(),
  targetService: z.string().optional(),
  primarySearchTerm: z.string().optional(),      // client sends this
  competitorUrls: z.array(z.string().url()).max(5).optional(),  // array, not CSV
  gbpUrl: z.string().url().optional(),
  socialUrls: z.object({
    facebook: z.string().url().optional(),
    instagram: z.string().url().optional(),
    linkedin: z.string().url().optional(),
    youtube: z.string().url().optional(),
    tiktok: z.string().url().optional(),
  }).optional(),
  contactName: z.string().max(120).optional(),
  contactEmail: z.string().email().optional(),
  contactPhone: z.string().max(40).optional(),
});
```

Apply it with `safeParse` in `analyze-website` — which today has **no** validation at all — and derive the client-side TypeScript type with `z.infer` so the form cannot be typed against a shape the server does not accept. Change the client's `competitorUrls` textarea to split on commas into an array before sending.

**Benefit.** Restores roughly 45 score points of genuine signal, makes the competitor comparison real for the first time, and removes an entire class of defect permanently — the client and server now cannot disagree about the contract, which is the actual root cause. Surfacing all Zod issues rather than only `issues[0]` additionally shortens the debug cycle for every future field change.

**Effort.** ~3 hours. **Cost of inaction:** every report remains structurally wrong, and the platform cannot honestly claim the assessment reflects the submitted information.

### 10.4 Harden the outbound fetch path — P1

**Problem.** Three residual gaps in an otherwise strong SSRF defence: a DNS-rebinding TOCTOU window between validation and fetch; `redirect: 'follow'` with no per-hop revalidation, so a public URL can 302 into cloud metadata; and rate limiting that is per-process, keyed on a spoofable header, and never evicted.

**Change.**
- Set `redirect: 'manual'` and re-run `validateUrlForSSRF` on each hop, with a hop ceiling of 3.
- Resolve once, validate the returned IP, and pin the connection to that IP with a `Host` header override — closing the rebinding window.
- Derive the client IP from a trusted-proxy-aware helper that prefers `x-real-ip` and takes only the last entry of `x-forwarded-for`.
- Move the rate-limit store to a shared backend if deploying to more than one instance; at minimum, evict map entries older than the window on a timer.
- Distinguish fetch failures explicitly. When the target is unreachable, return a clear error state — not a confident report with a 38/100 score.
- Remove the `responseTimeMs === 0` defect at `:328` and award latency points only on a completed request.

**Benefit.** Closes the last paths to cloud-credential theft from a user-supplied URL, makes the rate limit actually effective in the actual deployment topology, and — most valuable commercially — stops the platform from confidently reporting a score for a website it could not reach.

**Effort.** ~1 day. **Cost of inaction:** a residual SSRF path in the one feature whose entire purpose is fetching attacker-supplied URLs.

### 10.5 Fix the Tailwind plugin registration — P1

**Problem.** `app/globals.css` contains exactly one directive — `@import "tailwindcss"` — with no `@plugin` line. `tw-animate-css` and `@tailwindcss/typography` are installed and listed in `devDependencies`, but in Tailwind v4 they must be explicitly registered. Consequently all 7 `animate-in fade-in` entrances are no-ops, and `prose prose-neutral` at `app/insights/[slug]/page.tsx:111` produces no styles at all, so every database-authored insight article renders as an unstyled wall of text.

**Change.**
```css
@import "tailwindcss";
@plugin "@tailwindcss/typography";
@plugin "tw-animate-css";
```

**Benefit.** Two lines restore every broken entrance animation across the admin panel, both tools, both reports, and the floating CTA — and repair the entire content-marketing surface. The `typography` plugin is the highest-leverage single fix in this document relative to its cost.

**Effort.** ~10 minutes. **Cost of inaction:** the insights section, which exists to establish topical authority for AEO/GEO, currently renders as unstyled paragraphs.

### 10.6 Add provenance to all generated output — P1

**Problem.** Three distinct sources of unlabelled synthetic content: the Gemini path and the hard-coded fallback are indistinguishable (§4.2.3); the competitor benchmark is fabricated (§4.1.5); and roughly a dozen report strings are hard-coded constants presented as findings (§7.2.3).

**Change.**
1. **Record how the narrative was produced.** Add to the AI assessment record:
   ```ts
   provenance: {
     narrativeSource: 'gemini' | 'template_fallback',
     model: 'gemini-2.5-flash' | null,
     promptVersion: 'v1',
     generatedAt: new Date().toISOString(),
   }
   ```
2. **Fix the LLM configuration** while in the same code: add `responseSchema` for a guaranteed structure, `maxOutputTokens` sized to the actual response so truncation fails loudly rather than silently, an explicit `temperature`, a `systemInstruction` separated from the user content, and a `maxDuration` export on the route.
3. **Surface it in the UI.** Render a small provenance line on the AI report: *"Narrative generated by gemini-2.5-flash on &lt;date&gt;"* or *"Template analysis — AI narrative generation was unavailable at submission time."*
4. **Either fetch competitors or remove the table.** Fetching is the better product and is a small addition — the SSRF guard already exists, so each competitor URL can be run through `validateUrlForSSRF` and scored with the existing `scoreCategory` logic. If fetching is out of scope, delete the table rather than shipping a fabricated one.
5. **Delete or label every hard-coded claim.** The `recommendedServices` list (`analyze-website/route.ts:545-566`) should at minimum be derived from the actual findings — a missing WhatsApp link should surface a WhatsApp service, not a fixed four-item list.
6. **Remove the out-of-range defaults.** `|| 78` and `|| 80` (`ai-opportunity/[id]/page.tsx:322`; `ai-assessment/page.tsx:696`) and the fabricated `'NPR 25,000 - 45,000/mo'` (`:371`) should render an explicit "not available" state instead of an invented number.

**Benefit.** This is the single most important change for the platform's *positioning*. The product sells trustworthy automated assessment; provenance labelling converts a silent integrity failure into a visible, defensible, auditable state — and in the case of the competitor table, removes an outright false claim from client-facing documents. Fixing `responseSchema` and `maxOutputTokens` also eliminates the silent-truncation path, which currently degrades quality with no error signal.

**Effort.** Provenance field and UI: ~2 hours. LLM configuration hardening: ~3 hours. Competitor fetching: ~1 day. **Cost of inaction:** the platform's central differentiator is its credibility, and it currently spends that credibility on output it cannot substantiate.

### 10.7 Correct the SEO surface — P1

**Problem.** Report pages are indexable but empty to crawlers, canonicalised to the homepage, unprotected, and carrying customer PII (SEO-01, SEO-03, SEO-04, SEO-05).

**Change.**
1. Add `app/reports/layout.tsx` with `metadata: { robots: { index: false, follow: false } }`, covering both report routes.
2. Add `/reports/` to the `robots.ts` disallow list.
3. Remove `alternates.canonical: '/'` from the root layout; set canonicals per route.
4. Strip the hardcoded brand suffix from the six page titles that already include it.
5. Add `/digital-visibility-check` and `/ai-assessment` to the sitemap and to the AI-crawler allow-list.
6. Add `app/not-found.tsx` with its own metadata.

**Benefit.** Removes a crawl trap and stops customer PII URLs from entering the index; eliminates duplicate-title and wrong-canonical signals across the whole site; and puts the platform's two highest-value tool pages in front of the crawlers its own AEO positioning targets.

**Effort.** ~2 hours. **Cost of inaction:** dilution of index quality, a growing set of thin duplicate URLs, and PII-bearing pages eligible for indexing.

### 10.8 Escape structured-data output — P1

**Problem.** All seven JSON-LD call sites inject `JSON.stringify` output into a `<script>` tag without `<`-escaping, so any stored field containing `</script>` breaks out of the tag. Latent today because inputs are static or admin-authored; live the moment an admin pastes markup into a case study or insight.

**Change.** One helper, used everywhere:
```ts
export function jsonLd(schema: object): string {
  return JSON.stringify(schema).replace(/</g, '\\u003c');
}
```

**Benefit.** Removes a stored-XSS class permanently, and is a five-line change against seven call sites. Preventive fixes are always cheaper than incident response.

**Effort.** ~30 minutes. **Cost of inaction:** a latent vulnerability in the one place the site deliberately renders dynamic, database-authored content into markup.

### 10.9 Repair the entity graph — P2

**Problem.** Three disconnected `LocalBusiness` copies rather than one referenced entity (SEO-06), plus a `SearchAction` pointing at `/insights?q=...`, a search route that does not exist (`lib/seo.ts:187-209`).

**Change.** Make `getServiceSchema` and both tool layouts reference `{"@id": "${COMPANY_INFO.url}/#localbusiness"}` instead of embedding an inline copy. Either build a real `/insights` search page or remove the `SearchAction`.

**Benefit.** A single unambiguous entity node is what knowledge graphs actually consume. Deduplicating the business entity is a prerequisite for any credible AEO/GEO outcome, and the fix is smaller than it sounds.

**Effort.** ~2 hours. **Cost of inaction:** the structured-data investment the platform has already made delivers diluted value.

### 10.10 Replace the client-rendered report routes with server components — P2

**Problem.** Both reports are 888- and 1,215-line client components that ship a full React bundle and then block on a `useEffect` fetch, rendering a spinner to any non-JavaScript client (§7.3.3).

**Change.** Make `/reports/[type]/[id]/page.tsx` a Server Component that awaits `params`, fetches via `db.get*ById`, and renders the document directly. Move the delivery actions (print, email, WhatsApp) into small client components. Add `generateMetadata` for a per-report title.

**Benefit.** Four improvements at once: reports become crawlable with real metadata, the spinner disappears, the JavaScript bundle drops by most of the page weight, and delivery actions move behind a real auth boundary in the same refactor. It is the structural fix that resolves SEO-01, SEO-03, and SEC-07 together rather than separately.

**Effort.** ~2–3 days, mostly mechanical. **Cost of inaction:** the highest-value sales artifacts remain the slowest-loading and least indexable pages on the platform.

### 10.11 Decompose the admin monolith — P2

**Problem.** `app/admin-route-login/page.tsx` is 2,597 lines in one component, largely `any`-typed, covering authentication, a lead CRM, three content editors, user management, and diagnostics, with no code splitting (§7.3.2).

**Change.** Split by domain into `components/admin/{auth,leads,case-studies,insights,testimonials,users,diagnostics}.tsx`. Extract the authentication gate into a dedicated `/admin` route group with its own `layout.tsx` and server-side redirect, leaving the login form at `/admin-route-login` as the only public admin surface. Type the payloads with interfaces shared with `lib/db.ts` instead of `any`.

**Benefit.** The admin bundle stops being the largest in the application; the authentication gate moves to the server where it cannot be bypassed by a client-side check; and the `any` types stop defeating `strict` TypeScript. Restructuring also creates the natural home for the route-group-level `robots` metadata the current layout lacks.

**Effort.** ~2 days. **Cost of inaction:** the CMS is the component most likely to break under modification, and it is the one with no type safety.

### 10.12 Declare a rendering and caching strategy — P2

**Problem.** No `generateStaticParams`, `revalidate`, `dynamic`, or `runtime` export exists anywhere (§7.3.1). Behaviour is data-dependent and unintentional: DB-backed pages may be prerendered at build time and serve stale CMS content indefinitely, while detail routes re-query per request.

**Change.** Make the intent explicit per route:
- `/`, `/case-studies`, `/insights`, `/testimonials` → `export const revalidate = 300`
- `/case-studies/[slug]`, `/insights/[slug]` → `generateStaticParams` from the database plus `dynamicParams = true`
- `/api/analyze-website`, `/api/ai-assessments` → `export const runtime = 'nodejs'` and `export const maxDuration = 30`
- Admin routes → `export const dynamic = 'force-dynamic'`

**Benefit.** Eliminates an entire class of unexplained staleness bug, and — combined with `generateStaticParams` — turns the case-study and insight detail pages into static HTML, which is both faster and better for crawlers. The `maxDuration` exports also make the long-running audit path explicit to the hosting platform.

**Effort.** ~4 hours. **Cost of inaction:** content staleness and caching behaviour that no one can reason about.

### 10.13 Migrate schema ownership out of the request path — P2

**Problem.** `ensureDbInitialized()` runs full DDL, migrations, three `COUNT(*)` checks, and up to thirteen seed inserts before every repository method, is not concurrency-safe, and **swallows its own errors** so that a failed initialisation surfaces later as a confusing query error (`lib/db.ts:701-947`).

**Change.** Add a `scripts/migrate.ts` (or a `db:migrate` npm script) that owns DDL and seeding, run it once per deployment. Leave `ensureDbInitialized` as a lightweight `CREATE TABLE IF NOT EXISTS` safety net for local development, or remove it. Delete the stale, contradicted `lib/schema.sql` in favour of one authoritative schema file. Apply real migrations for subsequent changes rather than `ADD COLUMN IF NOT EXISTS` batches.

**Benefit.** Removes DDL from the hot path, eliminates a class of cold-start race, and makes a broken database fail at deploy time with a clear message instead of at request time with a misleading one. Deleting the contradictory `schema.sql` removes a genuine trap for the next engineer.

**Effort.** ~4 hours. **Cost of inaction:** cold-start races and misleading failures persist indefinitely.

### 10.14 Fix the remaining data-layer defects — P2

**Problem.** `addLead` writes `sourcePage` into both the `source` and `source_page` columns, so real attribution is never stored (`lib/db.ts:1130-1131`); `industry` is collected and always written `NULL`; UTM parameters are declared and never persisted; AI assessments store the record twice; `analysis-route` substitutes a shared mailbox as the lead email when only a phone is given.

**Change.** Correct the column mapping, add `industry` and the three UTM columns to the live DDL and the insert statement, drop the duplicate `opportunities` write (or drop `raw_submission`), and make the phone-only path write `NULL` rather than a shared mailbox.

**Benefit.** Lead attribution becomes trustworthy — which matters directly, because `ctaIdentifier` and `sourcePage` are the fields the growth team uses to know which CTAs actually convert. Roughly halves AI-assessment storage.

**Effort.** ~3 hours. **Cost of inaction:** the platform cannot answer "which call to action generates the most leads", which is the question its own analytics table exists to answer.

### 10.15 Enforce typing and repair the toolchain — P3

**Problem.** `any` types on the error path of every route and across most of the admin component; `eslint-config-next@16.0.8` against Next `^15.4.9`; `eslint.ignoreDuringBuilds: true`; both `.eslintrc.json` and `eslint.config.mjs` present; an invalid `next clean` script; two lockfiles; `scratch.ts` duplicating a route file; `@neondatabase/serverless` and `@hookform/resolvers` installed but never imported.

**Change.** Replace `err: any` with `unknown` plus a narrowing helper. Align the ESLint major version with Next. Delete `.eslintrc.json` and the `clean` script. Choose one package manager and delete the other lockfile. Delete `scratch.ts` and the unused dependencies. Once lint is trustworthy, set `ignoreDuringBuilds: false`.

**Benefit.** Restores the value of `strict` TypeScript and makes the build a genuine quality gate. A build that cannot fail on lint errors is a build that cannot catch regressions.

**Effort.** ~3 hours. **Cost of inaction:** the safety net exists but has no holes in it only by accident.

### 10.16 Adopt a type system and a font — P3

**Problem.** No `next/font`, no `@font-face` — the site renders in the platform UI sans stack, with 442 `font-bold` and 52 `font-black` declarations carrying hierarchy that a type system should. Hand-picked heading sizes include many arbitrary 10–12px values inside the reports. No dark mode, despite the brand shipping a dark logo variant consumed via a `variant` prop.

**Change.** Load one or two families through `next/font/google` (or `next/font/local`) with a self-hosted variable font, define a type scale in `@theme`, and replace arbitrary sizes with named steps. Add a dark palette if desired — the neutral ramp is already defined.

**Benefit.** Meaningful improvement in perceived quality and readability at low cost, and a genuine differentiator for an agency selling design-led work. `next/font` also eliminates a render-blocking request and prevents FOUT.

**Effort.** ~1 day. **Cost of inaction:** the site reads as generic, which works against the premium positioning the copy attempts.

### 10.17 Improve accessibility — P3

**Problem.** `faq-accordion` sets `aria-expanded` but has no `aria-controls`, and its panel has no `id` or `role="region"`. `testimonial-slider` has no `aria-live` and no accessible pause control for its auto-advance. The header mobile drawer has no focus trap and no `Escape` handler. Icon-only buttons — including the report modal close buttons — carry no `aria-label`.

**Change.** Add the ARIA relationships, a visible pause control on the slider, a focus trap and `Escape` handler in the drawer, and `aria-label` on every icon-only control.

**Benefit.** WCAG conformance, keyboard operability, and screen-reader support — and a measurable, demonstrable item for the accessibility section of a pitch.

**Effort.** ~4 hours. **Cost of inaction:** the site is partially unusable for keyboard and screen-reader users.

### 10.18 Replace generic stock imagery — P3

**Problem.** Every case study uses an Unsplash photograph — restaurant interiors, a generic classroom, stock dental stock — on a site whose entire proposition is specific local credibility in Eastern Nepal. One of the six case studies has a logistics hero showing a container ship (`lib/db.ts:508`) for a Biratnagar–Itahari road freight business.

**Change.** Photograph or commission actual client work. Where a client cannot be photographed, use a designed graphic treatment — a results panel, a before/after, a product screenshot — rather than unrelated stock photography.

**Benefit.** Credibility is the product. Stock imagery of a classroom in another country, presented as a client's project, is an integrity risk of exactly the kind §10.6 addresses elsewhere.

**Effort.** Ongoing, client-dependent. **Cost of inaction:** the site undercuts its own positioning on the first scroll of every case study.

---

## 11. Prioritised Roadmap

```mermaid
flowchart TB
    subgraph P0["P0 — Contain the exposure · do first, hours not days"]
        direction LR
        A1["10.1 Rotate + purge<br/>committed secrets"]
        A2["10.2 Real auth on all<br/>34 endpoints"]
        A3["10.3 Fix client/API<br/>contract"]
    end

    subgraph P1["P1 — Correctness & trust · the reporting layer"]
        direction LR
        B1["10.5 Register Tailwind<br/>plugins"]
        B2["10.4 Harden outbound<br/>fetch + rate limit"]
        B3["10.6 Add provenance;<br/>fix or remove fabricated output"]
        B4["10.7 Correct SEO<br/>surface"]
        B5["10.8 Escape JSON-LD"]
    end

    subgraph P2["P2 — Structure · makes the rest durable"]
        direction LR
        C1["10.10 Server-render<br/>reports"]
        C2["10.11 Decompose<br/>admin"]
        C3["10.12 Declare caching<br/>strategy"]
        C4["10.13 Migrate schema<br/>out of hot path"]
        C5["10.9 Repair<br/>entity graph"]
        C6["10.14 Fix data-layer<br/>defects"]
    end

    subgraph P3["P3 — Polish & credibility"]
        direction LR
        D1["10.15 Typing +<br/>toolchain"]
        D2["10.16 Type system<br/>+ font"]
        D3["10.17 Accessibility"]
        D4["10.18 Real imagery"]
    end

    P0 ==>|"unblocks"| P1
    P1 ==>|"unblocks"| P2
    P2 ==>|"unblocks"| P3

    style P0 fill:#fee,stroke:#c00,stroke-width:2px
    style P1 fill:#fff4e6,stroke:#e80,stroke-width:2px
    style P2 fill:#eef4ff,stroke:#06c,stroke-width:2px
    style P3 fill:#f0f0f0,stroke:#888
```

### 11.1 Sequencing rationale

**P0 is roughly two days of work and closes an active breach.** The secret rotation (§10.1) is urgent independently of this document — the credential is already in git history and should be considered compromised. Authentication (§10.2) is the largest single finding in the audit and gates any decision to keep storing customer PII. The contract fix (§10.3) is small, independent, and immediately improves every score the platform produces, so it belongs in the same batch.

**P1 is where the product's credibility is won or lost.** §10.5 is ten minutes for the highest visual return in this document. §10.6 is the strategic one: adding provenance and removing the fabricated competitor table is what allows the platform to keep claiming trustworthy automated assessment. §10.4 completes the SSRF work that is already 80% done. §10.7 and §10.8 close the SEO and injection findings cheaply.

**P2 is structural.** §10.10 resolves three separate findings at once and should be scheduled first within this tier. §10.12 and §10.13 remove classes of bug rather than instances. §10.11 matters most if the admin panel is about to be extended.

**P3 raises quality above parity** and can absorb spare capacity. §10.15 should not be deferred indefinitely, because an untrustworthy lint gate is how the type-level defects in §7 accumulated.

### 11.2 Effort summary

| Tier | Items | Estimated effort | Character |
|---|---|---|---|
| P0 | 3 | ~2 days | Containment. Security boundary + a contract fix. |
| P1 | 5 | ~4 days | Correctness and trust. The reporting layer becomes defensible. |
| P2 | 6 | ~2 weeks | Structure. Classes of defect removed. |
| P3 | 4 | ~1 week | Polish. Design quality, accessibility, credibility. |
| **Total** | **18** | **~4 weeks** | |

The critical observation is the ratio: **the three P0 items — the ones that matter most — are approximately two days of work.** The finding is not that the platform is expensive to fix. It is that a small number of omissions, mostly around access control, sit underneath a genuinely capable diagnostic engine.

---

## 12. Appendices

### A. Scoring formula reference

#### A.1 Visibility audit — `analyze-website/route.ts`

| Category | Base | Increments | Raw max | Clamp | Weight | Attainable range |
|---|---|---|---|---|---|---|
| `technicalFoundations` | 30 | HTTPS +25 · status 200 +15 · RT<1200ms +15 (else RT<2500ms +8) · viewport +10 · robots +5 · sitemap +5 · canonical +5 · HSTS +5 | 115 | [25, 100] | **0.18** | 25–100 |
| `searchVisibility` | 30 | title +15 · title len 35–65 +10 · meta desc +15 · desc len 110–165 +10 · exactly one H1 +10 · keyword in title +10 · not noindex +10 | 110 | [20, 100] | **0.18** | 20–100 |
| `localVisibility` | 30 | `gbpUrl` supplied +25 (else Maps link in HTML +20) · city in body +20 · phone +15 · explicit address +10 | 100 | [20, 100] | **0.15** | 20–100 |
| `contentSeo` | 30 | H1 +10 · ≥2 H2 +10 · `pageSizeKb`≥15 +15 · ≥2 service pages +15 · images w/ zero missing alt +10 · FAQ present +10 · keyword in body +10 | 110 | [25, 100] | **0.15** | 25–100 |
| `aeoGeoReadiness` | — | `aeoScore` = FAQ(35/10) + any schema(30/0) + >2 questions(20/10) + title&desc(15/5) · `geoScore` = about(25/10) + org|localBusiness(35/10) + address(20/5) + service pages(20/10) · mean of the two | 200 | [30, 100] | **0.14** | 30–100 |
| `conversionReadiness` | 30 | WhatsApp +25 · phone +15 · form +15 · booking +10 · testimonials +10 · pricing +5 | 110 | [20, 100] | **0.12** | 20–100 |
| `digitalPresence` | 30 | social URLs supplied +20 · FB +15 · IG +15 · LI +10 · YT or TT +10 | 100 | [25, 100] | **0.08** | 25–100 |

`overallScore = round(Σ category × weight)` — weights sum to **1.00**. Theoretical range **[23, 100]**.

| UI band | Threshold | Source |
|---|---|---|
| Strong Foundation | ≥ 75 | `app/digital-visibility-check/page.tsx:448-453` |
| Actionable Gaps | ≥ 50 | same |
| Critical Deficiencies | < 50 | same |

**Simulated unreachable site:** tech 63 · search 40 · all others 30 → `overallScore = 38`.

#### A.2 AI assessment — `ai-assessments/route.ts`

| Score | Formula | Clamp | Range |
|---|---|---|---|
| `automationOpportunity` | `(hasNoCrm ? 35 : 15) + (hasHighRepetition ? 30 : 15) + (automation is 'None' ? 25 : 10)` | [45, 95] | 45–95 |
| `customerExperienceOpportunity` | `(usesWhatsAppPrimary ? 30 : 15) + (challenges include 'Missed' ? 35 : 20) + 25` | [40, 95] | 40–95 |
| `salesOpportunity` | `(hasNoCrm ? 35 : 15) + (lead volume includes '100' ? 30 : 20) + 25` | [35, 92] | 35–92 |
| `marketingOpportunity` | `(marketing is 'occasional' ? 35 : 15) + 35` | [40, 88] | 40–88 |
| `operationsOpportunity` | `(manualDataEntry.length > 5 ? 30 : 15) + (hasHighRepetition ? 35 : 20) + 20` | [40, 94] | 40–94 |
| `dataAnalyticsOpportunity` | `(reporting is 'manual' ? 40 : 20) + 30` | [35, 90] | 35–90 |

`overallAIReadinessScore = round((Σ score × weight) × 0.92)` — weights 0.22 / 0.22 / 0.18 / 0.14 / 0.14 / 0.10 sum to 1.00, then a flat 8% haircut. **Attainable range [38, 84].**

Financial model: `monthlyHoursSaved = round(weeklyHours × 3.8)`; `monthlySavings = monthlyHoursSaved × NPR 350–600`.

#### A.3 Metrics that carry no independent information

| Field | Behaviour | Ref |
|---|---|---|
| `contentSeoDetails.keywordRelevance` | `keywordScore` ∈ [40, 100]; computed and stored but feeds **no** category score | `:280-293, 634` |
| `categoryScores.customerAccessibility` | Duplicates `convScore`; **not** in the weighted sum — a phantom 8th category | `:611` |
| `geoDetails.structuredDataScore` | `detectedSchemas.length × 20`, **unclamped**, can exceed 100 | `:696` |
| `aeoDetails.conciseAnswerScore` | `round(aeoScore × 0.9)` | `:683` |
| `aeoDetails.conversationalIntentScore` | `round(aeoScore)` — identical to its source | `:688` |
| `aeoDetails.trustSignals` | `strengths.slice(0, 3)` | `:687` |
| `contentSeoDetails.duplicateThinRisk` | `< 15` = "High risk" — **inverted** vs `≥ 15` = "not thin" | `:640` vs `:360` |
| `localVisibilityDetails.locationLandingPages` | Hard-coded `false` | `:658` |
| `localVisibilityDetails.reviewVolumeEstimate` | Hard-coded string | `:655` |
| `searchVisibilityDetails.serpFeatures` | Hard-coded 3-item array | `:672` |
| `socialDetails.activityEstimate` | Hard-coded "Cross-linking detected" | `:729` |
| `conversionDetails.trustSignals` | Hard-coded, includes "SSL Secure" on HTTP sites | `:718` |
| `conversionDetails.ctaScore` | Duplicates `convScore` | `:712` |
| `recommendedServices` | 4 entirely hard-coded strings | `:545-566` |
| `competitorComparison` | Target's own scores, relabelled; competitors never fetched | `:508-542` |
| `roadmap` (AI report) | Gemini output assigned and never read | `ai-opportunity/[id]/page.tsx:210` |

### B. Consolidated findings register

**Totals: 77 findings — 3 Critical, 20 High, 33 Medium, 19 Low, 2 Informational.**

#### B.1 Security (17)

| ID | Sev | Finding | Ref |
|---|---|---|---|
| SEC-01 | Critical | Unencrypted OpenSSH private key committed and not gitignored | `./y`, `.gitignore` |
| SEC-02 | Critical | Neon DSN with embedded password as hard-coded fallback | `lib/db.ts:671-673` |
| SEC-03 | Critical | No authentication on any of 34 endpoints | all `app/api/*` |
| SEC-04 | High | Unauthenticated PUT/DELETE on CMS content and users | `app/api/admin/*` |
| SEC-05 | High | Unauthenticated, unvalidated PATCH/DELETE on both report routes | `*/[id]/route.ts:20-51` |
| SEC-06 | High | Report IDs enumerable (`Date.now()` + 6 base36) | `lib/db.ts:1223, 1317` |
| SEC-07 | High | `/reports/**` indexable, carries customer PII | `app/robots.ts:11` |
| SEC-08 | Medium | Unsigned constant session cookie; hard-coded backdoor password | `admin/auth/route.ts:6-19` |
| SEC-09 | Medium | SSRF TOCTOU + unvalidated redirect following | `analyze-website:62-70` |
| SEC-10 | Medium | Rate limit per-process, spoofable key, never evicted | `lib/ssrf.ts:8-24` |
| SEC-11 | Medium | Unescaped JSON-LD injection, 7 sites | `app/layout.tsx:137-149` |
| SEC-12 | Medium | Unauthenticated unbounded analytics write | `analytics/track/route.ts` |
| SEC-13 | Medium | `ssl.rejectUnauthorized: false` on DB connection | `lib/db.ts:685` |
| SEC-14 | Low | Recipient email duplicated into analytics metadata | `send-email/route.ts:68` |
| SEC-15 | Low | No CSRF protection on state-changing routes | all mutating routes |
| SEC-16 | Low | No max lengths on Zod free-text fields | `ai-assessments:6-32` |
| SEC-17 | Info | Cleartext PII, no retention policy or consent gate | `lib/db.ts:1221-1348` |

#### B.2 SEO (13)

| ID | Sev | Finding | Ref |
|---|---|---|---|
| SEO-01 | High | Report pages indexable, metadata-less, canonicalised to `/` | report pages; `app/robots.ts:11` |
| SEO-02 | High | `typography` plugin unregistered → all articles unstyled | `app/globals.css:1`; `insights/[slug]:111` |
| SEO-03 | High | Root canonical `/` inherited by report + admin routes | `app/layout.tsx` |
| SEO-04 | Medium | Title template double-appends brand | `app/layout.tsx:23` + 6 pages |
| SEO-05 | Medium | Both tool pages absent from sitemap and AI allow-list | `sitemap.ts`; `robots.ts` |
| SEO-06 | Medium | Three disconnected `LocalBusiness` entity copies | `lib/seo.ts:211-232` |
| SEO-07 | Medium | `lastModified: new Date()` on every sitemap entry | `sitemap.ts:23, 30` |
| SEO-08 | Medium | Static `public/llms.txt` duplicates the generated route | `public/llms.txt` |
| SEO-09 | Medium | `WebApplication` priced in USD for NPR services | both tool layouts |
| SEO-10 | Low | No `aggregateRating` despite a 5-star testimonials page | `lib/seo.ts` |
| SEO-11 | Low | `getArticleJsonLd` ignores its `authorName` parameter | `lib/seo.ts:262-297` |
| SEO-12 | Low | No `not-found.tsx` | project-wide |
| SEO-13 | Info | `/llms.txt` in robots but not sitemap; webmanifest coherence | `robots.ts`; `site.webmanifest` |

#### B.3 Correctness (14)

| ID | Sev | Finding | Ref |
|---|---|---|---|
| COR-01 | High | 4 form fields silently discarded on every submission | `dvc/page.tsx:95-98` vs `analyze:16-26` |
| COR-02 | High | +8 latency points awarded on a failed request | `analyze-website:328` |
| COR-03 | High | Client reports report-request success on all outcomes incl. failure | `dvc/page.tsx:157-164` |
| COR-04 | High | `POST /api/visibility-audits` dead; its promise has no code path | `visibility-audits/route.ts` |
| COR-05 | Medium | `pageSizeKb` counts script bytes; thin thresholds inverted | `analyze-website:136, 360, 640` |
| COR-06 | Medium | `structuredDataScore` unclamped, can exceed 100 | `analyze-website:696` |
| COR-07 | Medium | `conversationalIntentScore` identical to `aeoScore` | `analyze-website:688` |
| COR-08 | Medium | Tag-stripped body text retains script + JSON-LD | `analyze-website:282` |
| COR-09 | Medium | Competitor code paths use different thresholds, opposite polarity | `analyze-website:515-527, 530-542` |
| COR-10 | Medium | `industry` state default matches no `<option>` → blank select | `ai-assessment/page.tsx:31, 217` |
| COR-11 | Low | `roadmap` assigned and never read | `ai-opportunity/[id]:210` |
| COR-12 | Low | `responseTimeMs` read top-level, nested in response | `dvc/page.tsx:48, 435` |
| COR-13 | Low | `alt=""` counted as present | `analyze-website:178` |
| COR-14 | Low | Canonical regex requires `rel` before `href` | `analyze-website:159` |

#### B.4 Data layer (5)

| ID | Sev | Finding | Ref |
|---|---|---|---|
| DAT-01 | High | `addLead` writes `sourcePage` into both `source` and `source_page` | `lib/db.ts:1130-1131` |
| DAT-02 | Medium | `industry` collected, always written `NULL` | `leads/route.ts:5-18` |
| DAT-03 | Medium | AI assessment record stored twice | `lib/db.ts:1342-1343` |
| DAT-04 | Medium | Phone-only submissions store a shared mailbox as the email | `analyze-website:757-768` |
| DAT-05 | Low | UTM columns declared, never persisted | `lib/db.ts`; `schema.sql` |

#### B.5 LLM and prompt (8)

| ID | Sev | Finding | Ref |
|---|---|---|---|
| LLM-01 | High | No `responseSchema` — structure rests on prompt prose | `ai-assessments:180-182` |
| LLM-02 | High | No `maxOutputTokens` — silent truncation → silent fallback | `ai-assessments:177-190` |
| LLM-03 | High | No provenance field; Gemini and fallback indistinguishable | `ai-assessments:193-211` |
| LLM-04 | Medium | No timeout, `abortSignal`, or `maxDuration` on the LLM call | `ai-assessments:177-183` |
| LLM-05 | Medium | No retry/backoff; no 429 handling | `ai-assessments:188-190` |
| LLM-06 | Medium | Prompt injection: unescaped user text, no length caps | `ai-assessments:92, 6-32` |
| LLM-07 | Low | `GoogleGenAI` instantiated per request, not a singleton | `ai-assessments:86` |
| LLM-08 | Low | Model hard-coded, no env override or fallback | `ai-assessments:178` |

#### B.6 Reporting integrity (8)

| ID | Sev | Finding | Ref |
|---|---|---|---|
| RPT-01 | High | Competitor benchmark fabricated from the target's own scores | `analyze-website:508-542` |
| RPT-02 | High | Email endpoint hard-codes `status: 'DELIVERED'`; no transport exists | `send-email:71-80` |
| RPT-03 | High | Visibility report's AEO/GEO narrative is regex output, presented as AI | `analyze-website` (whole file) |
| RPT-04 | Medium | `[VERIFIED]`/`[ESTIMATED]` tags do not track actual provenance | `analyze-website:414-418, 485-488` |
| RPT-05 | Medium | Unsourced statistics asserted as measured fact (×6) | `ai-assessments:196-210`; `analyze:718, 729` |
| RPT-06 | Medium | Out-of-range score defaults `78` / `80` and fabricated NPR default | `ai-opportunity/[id]:322, 371` |
| RPT-07 | Low | "Document Hash" is an ID, not a hash | `visibility/[id]:1117` |
| RPT-08 | Low | Empty-state gap in the opportunity matrix table | `ai-opportunity/[id]:209, 609` |

#### B.7 Frontend, architecture & tooling (12)

| ID | Sev | Finding | Ref |
|---|---|---|---|
| FRN-01 | High | Tailwind plugins unregistered — 7 dead animations, all articles unstyled | `app/globals.css:1` |
| FRN-02 | High | Admin CMS is one 2,597-line component | `admin-route-login/page.tsx` |
| FRN-03 | Medium | No rendering or caching strategy anywhere in `app/` | project-wide |
| FRN-04 | Medium | Report routes client-rendered → spinner to all non-JS clients | both report pages |
| FRN-05 | Medium | No error boundaries; optional-chaining gaps can blank a report | report pages; project-wide |
| FRN-06 | Medium | NAP duplicated in 3 places with no compile-time guard | `seo.ts:1-46`; `footer:21-33`; `llms.txt` |
| FRN-07 | Medium | Audit response inflated ~2.5× — record returned three times | `analyze-website:777-782` |
| FRN-08 | Medium | Admin dashboard loads entire tables, no pagination | `admin/data/route.ts` |
| FRN-09 | Low | `handleCopyReportLink` dead code | `visibility/[id]:97` |
| FRN-10 | Low | ARIA gaps: accordion, slider, drawer, icon-only buttons | `components/ui/*`; reports |
| FRN-11 | Low | Lint non-functional; `ignoreDuringBuilds: true`; dual config | `next.config.ts:5-7`; both eslint files |
| FRN-12 | Low | Repo debris: `scratch.ts`, dead deps, two lockfiles, stale `schema.sql` | project-wide |

### C. Route inventory

| Route | Component type | Data source | Metadata |
|---|---|---|---|
| `/` | Server | `db.getCaseStudies()`, `db.getTestimonials()` | inherited |
| `/about` | Server | static | ✅ |
| `/services` | Server | static + `lib/seo.ts` | ✅ |
| `/industries` | Server | `INDUSTRIES_DATA` | ✅ |
| `/case-studies` | Server | `db.getCaseStudies()` | ✅ |
| `/case-studies/[slug]` | Server | `db.getCaseStudyBySlug` | ✅ `generateMetadata` |
| `/insights` | Server | `db.getInsights()` | ✅ |
| `/insights/[slug]` | Server | `db.getInsightBySlug` | ✅ `generateMetadata` |
| `/testimonials` | Server | `db.getTestimonials()` | ✅ |
| `/contact` | **Client** | `POST /api/leads` | via layout |
| `/digital-visibility-check` | **Client** | `POST /api/analyze-website` | via layout |
| `/ai-assessment` | **Client** | `POST /api/ai-assessments` | via layout |
| `/reports/visibility/[id]` | **Client** (1,215 ln) | `GET /api/visibility-audits/[id]` | ❌ none |
| `/reports/ai-opportunity/[id]` | **Client** (888 ln) | `GET /api/ai-assessments/[id]` | ❌ none |
| `/admin-route-login` | **Client** (2,597 ln) | `/api/admin/*` | ❌ none |

Special files: `layout.tsx`, `globals.css`, `page.tsx`, `sitemap.ts`, `robots.ts`, `llms.txt/route.ts`, `icon.tsx`, `apple-icon.tsx`, `favicon.ico/route.ts`.
**Absent:** `not-found.tsx`, `error.tsx`, `global-error.tsx`, `loading.tsx`, `app/reports/layout.tsx`, `app/admin-route-login/layout.tsx`, `middleware.ts`.

### D. Key file:line index

| Concern | Location |
|---|---|
| SSRF validation | `lib/ssrf.ts:69-144`; IP checks `:26-61`; rate limit `:8-24` |
| Audit fetch | `app/api/analyze-website/route.ts:56-87` |
| robots / sitemap probes | `:89-131` |
| Signal extraction (40+) | `:136-318` |
| **Scoring block** | **`:320-400`** — tech `:324`, search `:337`, local `:348`, content `:357`, aeo `:301`, geo `:312`, combined `:368`, conversion `:371`, presence `:381`, **aggregate `:392-400`** |
| Findings / competitors / services | `:405-566` |
| Persist · lead · event · respond | `:591-754` · `:757-768` · `:770-775` · `:777-782` |
| AI Zod schema | `app/api/ai-assessments/route.ts:6-32` |
| Financial model | `:47-52` |
| **Opportunity scores** | **`:65-70`** |
| **Readiness aggregate** | **`:72-80`** |
| Gemini prompt · call · catch | `:87-175` · `:177-183` · `:188-190` |
| Fallback prose | `:193-211` |
| Admin auth (cookie set only) | `app/api/admin/auth/route.ts:6-19` |
| Fake email delivery | `app/api/reports/send-email/route.ts:64-80` |
| DB pool · hard-coded DSN | `lib/db.ts:680-694` · **`lib/db.ts:671-673`** |
| DDL bootstrap (swallows errors) | `lib/db.ts:701-947` (catch `:943-946`) |
| Row mappers | `lib/db.ts:978-1013` |
| `addLead` source-column defect | `lib/db.ts:1130-1131` |
| Report ID generation | `lib/db.ts:1223, 1317` |
| SEO helpers | `lib/seo.ts:1, 48, 91, 135, 187, 211, 234, 249, 262` |
| JSON-LD injection sites | `app/layout.tsx:137-149`; both tool layouts `:52-59` |
| Print / PDF | `ai-opportunity/[id]:69-88`; `visibility/[id]:76-95`; `app/globals.css:38-69` |
| Tailwind directives (plugins absent) | `app/globals.css:1` |
| Broken `prose` usage | `app/insights/[slug]/page.tsx:111` |
| Form wizard | `app/ai-assessment/page.tsx:27, 166-181, 184-632, 635-755` |
| Visibility form (contract mismatch) | `app/digital-visibility-check/page.tsx:25-49, 73-137, 89-102` |
| Committed secrets | `./y`, `./y.pub`, `.gitignore`; `lib/db.ts:671-673` |

### E. Audit method and limitations

**What was done.** Full static reading of all 74 source files. Programmatic enumeration of all 34 HTTP endpoints with an authentication-reference scan. Formula-by-formula reverse engineering of both scoring engines, including arithmetic simulation of floors, ceilings, and the unreachable-site case. Field-by-field diff of the primary form's payload against its route handler's destructuring. Direct inspection of `app/globals.css` for plugin registration, and of `app/api/admin/*` for auth guards. Verification of git tracking status for `y` and `y.pub`.

**What was not done, and what that means.**

- **No build or runtime execution.** Findings are from source reading. `npm run build` was not run, so compile-time and build-time behaviour is unverified. Note that `next.config.ts:8-10` sets `typescript.ignoreBuildErrors: false`, so a build would be a meaningful additional signal.
- **No database access.** Whether the live Neon instance still contains the credentials in `lib/db.ts:671-673`, and whether real PII is present, is unknown. A read-only row count per table would size the actual exposure of SEC-03.
- **No dependency audit.** `npm audit` was not run. The dependency set is small and current-looking, but this is unverified.
- **No penetration testing.** The security analysis is a source-level review. It identifies the *absence* of controls with high confidence, but does not attempt exploitation.
- **Gemini output quality was not evaluated.** The prompt was read, not run. The finding is about the absence of `responseSchema`, token limits, and provenance — not about whether `gemini-2.5-flash` produces good prose.
- **Score distributions were not measured.** Calibration is derived from formula simulation, not from querying the score histogram of existing audits. §10.6's provenance work should be paired with a distribution analysis before the bands are retuned.

**Confidence summary.** The security findings (B.1) and the contract mismatch (COR-01) rest on direct, programmatic verification and are stated with high confidence. The scoring analysis (A) rests on verbatim formula transcription and arithmetic and is high confidence. The reporting-integrity findings (B.6) rest on reading the code paths end to end and are high confidence. Performance and maintainability characterisations (B.7) are qualitative judgement and should be read as such.

---

*End of document. 77 findings across 7 categories, 10 diagrams, 18 prioritised recommendations. The three P0 items total approximately two days of work.*
