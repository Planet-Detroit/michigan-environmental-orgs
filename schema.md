# ECOcensus 2.0 Schema Design

**Status:** Proposed — replaces the current `organizations` table.
**Purpose:** Support the three-tier ECOcensus database (501(c)(3) nonprofits, for-profit environmental businesses, informal groups) with AI-assisted classification, full provenance tracking, and a clean separation between raw data and curation.

---

## Why a new schema

The current `organizations` table grew organically since December 2025 to serve two products (the public directory and the ECOcensus prototype). It works, but it carries problems that hand-editing won't fix:

- **Two parallel tagging systems** (`focus` and `tags` columns) with no clear distinction
- **Two parallel type fields** (`org_type` and `organization_type`)
- **No provenance** — if a tag is wrong, there's no way to know whether a human, an admin, a submitter, or an import script set it
- **No confidence scores** — every tag is treated equally regardless of how it was assigned
- **Geography hardcoded in JS**, not in the database
- **One row per org** with no concept of tiers, even though the proposal calls for nonprofit / for-profit / informal as distinct categories
- **Financial data shape** is bolted onto a row that was originally designed for a directory listing

The new schema treats AI classification as a first-class citizen, separates raw data from human curation, and lets each entity carry as much or as little structured data as its source provides.

---

## Tables

### `entities`

The base record. One row per organization, business, or informal group, regardless of tier.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `slug` | text (unique) | URL-safe identifier |
| `name` | text | Display name |
| `legal_name` | text | If different from display name |
| `entity_type` | enum | `nonprofit_501c3`, `nonprofit_501c4`, `nonprofit_other`, `b_corp`, `for_profit`, `government`, `unincorporated`, `other` |
| `tier` | smallint | 1 = nonprofit, 2 = for-profit, 3 = informal — matches the proposal |
| `status` | enum | `active`, `inactive`, `dissolved`, `pending_review`, `rejected` |
| `ein` | text (unique when present) | For 501(c)(3)s |
| `external_ids` | jsonb | Other identifiers (B Corp number, EGLE facility ID, etc.) keyed by source |
| `mission_statement` | text | From 990 or website |
| `description` | text | AI-generated long-form summary |
| `year_founded` | smallint | |
| `website` | text | |
| `email` | text | |
| `phone` | text | |
| `street_address` | text | |
| `city` | text | |
| `state` | char(2) | |
| `zip` | text | |
| `latitude` | numeric | |
| `longitude` | numeric | |
| `county` | text | |
| `region_id` | uuid (FK → `geographic_regions`) | |
| `metro_id` | uuid (FK → `geographic_regions`) | |
| `is_planet_champion` | bool | Editorial flag |
| `is_impact_partner` | bool | Editorial flag |
| `pd_verified` | bool | True when a Planet Detroit human has confirmed the record |
| `pd_verified_at` | timestamptz | |
| `pd_verified_by` | text | |
| `last_verified_active` | timestamptz | When this entity was last confirmed to still be operating. Critical for Tier 3 informal groups whose Facebook pages can go dark without warning. |
| `confidence_indicator` | numeric(3,2) | 0.00 – 1.00. How confident are we that this record is accurate and the entity is real? Defaults to 1.0 for Tier 1 with 990 verification, lower for Tier 3 records sourced from social media discovery alone. |
| `primary_source` | text | Where the record originated. Examples: `propublica`, `planet_detroit_curation`, `mec_intake`, `b_corp_directory`, `egle_arcgis`, `osm`, `submission_form`, `google_search_facebook`, `google_search_instagram`, `manual_intake_mec`, etc. |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `tags`

Controlled vocabulary for issue areas, activities, and populations served. Replaces the dual `focus` / `tags` arrays.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `slug` | text (unique) | e.g. `water`, `environmental_justice`, `pfas` |
| `label` | text | Human-readable |
| `category` | enum | `focus_area` (broad — Water, Energy), `issue` (specific — PFAS, lead), `activity` (Advocacy, Research), `population` (Youth, Tribal) |
| `description` | text | What this tag means, for admin reference |
| `is_official` | bool | True for the 14 curated focus areas; false for AI-discovered tags |
| `parent_tag_id` | uuid (FK → self, nullable) | For hierarchical tags (e.g. PFAS is a child of Water) |
| `color` | text | Hex, for UI |
| `icon` | text | Lucide icon name |

### `entity_tags`

Junction table with full provenance. Every tag assignment carries who set it, when, and how confident.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `entity_id` | uuid (FK → `entities`) | |
| `tag_id` | uuid (FK → `tags`) | |
| `confidence` | numeric(3,2) | 0.00 – 1.00 |
| `source` | enum | `ai_classifier`, `admin_review`, `submitter`, `imported`, `mec_curation` |
| `assigned_by` | text | User ID or system identifier |
| `assigned_at` | timestamptz | |
| `is_primary` | bool | True for the org's primary issue area |
| `notes` | text | Admin override reasoning |

Composite unique constraint on `(entity_id, tag_id)`.

### `entity_classifications`

The full AI-generated classification record per entity, separate from individual tags. One current row per entity (history kept via `classified_at`).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `entity_id` | uuid (FK → `entities`) | |
| `relevance_score` | numeric(3,2) | How environmentally relevant is this entity? 0.0 – 1.0 |
| `ai_summary` | text | Plain-English description for the public dashboard |
| `issue_keywords` | text[] | Extracted entities: PFAS, lead, brownfield, watershed, etc. |
| `embedding` | vector(1536) | For "similar orgs" search |
| `model_version` | text | e.g. `claude-opus-4-6-20260101` |
| `classified_at` | timestamptz | |
| `reviewed_by` | text | Admin who confirmed/overrode (nullable) |
| `reviewed_at` | timestamptz | |

### `financials`

One-to-many by year. May already exist in the current schema; this is the canonical shape.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `entity_id` | uuid (FK → `entities`) | |
| `fiscal_year` | smallint | |
| `revenue_total` | numeric | |
| `revenue_contributions` | numeric | |
| `revenue_program_services` | numeric | |
| `expenses_total` | numeric | |
| `expenses_program` | numeric | |
| `expenses_administrative` | numeric | |
| `assets_total` | numeric | |
| `liabilities_total` | numeric | |
| `net_assets` | numeric | |
| `health_score` | text | `healthy`, `stable`, `at_risk` (computed) |
| `source_990_id` | text | ProPublica filing ID |
| `imported_at` | timestamptz | |

Composite unique constraint on `(entity_id, fiscal_year)`.

### `geographic_regions`

Promotes the hardcoded `geographicHierarchy.js` into the database. Self-referencing tree.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `name` | text | |
| `level` | enum | `region`, `metro`, `county`, `city` |
| `parent_id` | uuid (FK → self, nullable) | |
| `state` | char(2) | For multi-state expansion |
| `slug` | text (unique within parent) | |

### `entity_service_areas`

Junction. An entity can serve multiple areas; an area can be served by many entities.

| Column | Type | Notes |
|---|---|---|
| `entity_id` | uuid (FK → `entities`) | |
| `region_id` | uuid (FK → `geographic_regions`) | |

### `enrichment_log`

Audit trail for every change to an entity record. Critical for trust and debugging.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `entity_id` | uuid (FK → `entities`) | |
| `field_name` | text | |
| `old_value` | text | |
| `new_value` | text | |
| `source` | text | Same enum as elsewhere |
| `changed_by` | text | |
| `changed_at` | timestamptz | |

### `media_mentions`

Already exists. Update FK from `organization_id` → `entity_id`.

### `outlets`

Already exists. No structural changes.

### `submissions`

For pending submissions from the public form. Separate from `entities` so the curated dataset isn't polluted with unreviewed records.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `submitted_data` | jsonb | Raw form payload |
| `submitter_email` | text | |
| `submitter_name` | text | |
| `status` | enum | `pending`, `approved`, `rejected` |
| `merged_into_entity_id` | uuid (FK → `entities`, nullable) | Set when admin approves |
| `created_at` | timestamptz | |
| `reviewed_by` | text | |
| `reviewed_at` | timestamptz | |

---

## Migration mapping from current schema

| Current `organizations` column | New location |
|---|---|
| `id`, `name`, `slug`, `ein` | `entities` (same names) |
| `mission_statement_text` | `entities.mission_statement` |
| `mission_statement_url` | `enrichment_log` (as source URL) |
| `email`, `phone`, `address`, `city`, `state`, `zip` | `entities` (same names) |
| `latitude`, `longitude` | `entities` (same names) |
| `region`, `metro_area` | `entities.region_id`, `entities.metro_id` (FKs) |
| `geo_location` (array) | `entity_service_areas` (junction) |
| `focus` (array) | `entity_tags` with `category=focus_area`, `source=admin_review`, `confidence=1.0` |
| `tags` (array) | `entity_tags` with `category=issue`, same provenance |
| `org_type` / `organization_type` | `entities.entity_type` (consolidated) |
| `for_profit` | Implied by `entity_type` |
| `planet_champion`, `impact_partner` | `entities.is_planet_champion`, `is_impact_partner` |
| `verified`, `verified_at`, `verified_by` | `entities.pd_verified`, `pd_verified_at`, `pd_verified_by` |
| `status` | `entities.status` |
| `featured` | Drop unless used (check first) |
| `submitted_by_email`, `submitted_by_name`, `rejection_reason` | Move to `submissions` table |
| `revenue`, `assets` | `financials` (one row per year) |
| `ntee_code` | `entities.external_ids` (`{"ntee": "C30"}`) |
| Social URLs (facebook, twitter, instagram, etc.) | `entities` columns (keep flat for now) |
| `search_vector` | Recreate as a tsvector on the new `entities` table |
| `tags` array vs `focus` array | **Consolidated into `entity_tags` with category** |

---

## Key design decisions

1. **One table per entity, three tiers via column.** The proposal's three tiers all live in `entities`. Tier is a column, not a separate table. This keeps cross-tier queries (geographic, topical) trivial and avoids three-way unions.

2. **Provenance everywhere.** Every classification, every tag, every field change carries source attribution and timestamps. This is what lets us mix AI classifications, admin overrides, MEC ground-truthing, and reader submissions in one record without losing track of which is which.

3. **AI classifications are first-class data, not metadata.** `entity_classifications` is its own table because the AI output (relevance score, summary, embedding, keywords) is reference-quality structured data, not just supporting evidence for a tag.

4. **Geography in the database, not the codebase.** The hardcoded JS hierarchy moves to `geographic_regions`. This unblocks multi-state expansion (the long-term sustainability section of the proposal).

5. **Submissions kept separate.** Pending submissions don't pollute the curated dataset. They get merged into `entities` only after admin approval, preserving the integrity of cross-tier queries.

6. **Confidence is numeric, not boolean.** A 0.85-confidence Water tag from AI is meaningfully different from a 1.0 Water tag from a human admin. Both belong; the application layer decides what to display where.

---

## What this schema does NOT include

These are intentionally out of scope for v1:

- **Board/staff tables** (Schedule J/O data from 990s) — deferred to a later phase
- **Grants/funding flows** (Form 990 Schedule I) — deferred
- **Multi-language support** for tags or descriptions
- **Custom roles/permissions** — handled at the Supabase RLS layer
- **Tracking changes to ProPublica source data over time** — `enrichment_log` covers our edits, not upstream changes

These can be added without breaking the v1 design.
