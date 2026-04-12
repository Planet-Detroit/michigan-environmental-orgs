# Next Steps — ECOcensus 2.0 Rebuild

**Decision:** Start fresh on the database schema, ingest the full IRS 990 XML universe with AI classification, and merge the existing 607 curated organizations as a verified overlay. Keep the public site running on the old schema for a 2-3 week overlap period, then cut over.

**Source of truth for schema design:** [`schema.md`](./schema.md)

---

## Phase 0 — Freeze (1 day, before any code)

Goal: stop the bleeding so the migration target doesn't keep moving.

- [ ] Snapshot the current Supabase project (full backup)
- [ ] Document which apps read/write the current `organizations` table (see Phase 4 inventory below for the complete list)
- [ ] Stop adding new columns to the current table — any schema changes go into v2
- [ ] Communicate the freeze to anyone else with admin access
- [ ] **Audit orphaned financial rows.** As of April 2026 the `financials` table has 1,197 records but only 243 distinct organizations are referenced — and a January 2026 doc said the count was 286. Distinct-org coverage went *down* over time, which suggests rows in `organizations` were deleted without their child rows in `financials` being cleaned up (or vice versa). Run a join query to find:
  - Financial records pointing to org IDs that no longer exist in `organizations`
  - Org rows that *should* have financial data based on EIN, but don't
  - Whether the 286 → 243 drift has a clean explanation (e.g., dedup) or whether it represents real data loss
  This audit informs Phase 3, where EIN matching needs to be reliable.

**Out:** Current schema documented, locked, and backed up. Orphaned-row situation understood.

---

## Phase 1 — New schema, empty (3-4 days)

Goal: stand up the v2 schema in the same Supabase project under a new namespace, with no data yet.

- [ ] Create a new Supabase schema named `ecocensus` (separate from `public`) — keeps the old tables intact and the new ones isolated
- [ ] Write SQL migration files for every table in `schema.md`
  - `entities`, `tags`, `entity_tags`, `entity_classifications`, `financials`, `geographic_regions`, `entity_service_areas`, `enrichment_log`, `submissions`
  - Update `media_mentions` and `outlets` to FK against `entities` (these stay in `public` for now, just gain a new column)
- [ ] Seed `geographic_regions` from the existing `geographicHierarchy.js` file
- [ ] Seed `tags` with the 14 official focus areas from `issueIcons.js`, marked `is_official=true`
- [ ] Set up Supabase RLS policies (read-public for non-rejected entities, write-restricted to authenticated admins)
- [ ] Verify the schema with a few hand-inserted test rows, then truncate

**Out:** Empty `ecocensus` schema, migration files committed, RLS in place.

---

## Phase 2 — Ingest IRS 990 XML + AI classify (4-5 days)

Goal: populate `entities` with the full universe of MI environmental nonprofits, AI-classified at ingest time, with board member data from day one.

- [ ] Build the IRS XML ingestion pipeline
  - Download monthly bulk ZIP files from `apps.irs.gov/pub/epostcard/990/xml/`
  - Parse 990 XML for each MI nonprofit matching target NTEE codes (C, D, plus selected codes from food/ag, housing, recreation, science, community improvement per the Johnson Center scoping)
  - Extract core org data: mission, program descriptions, financials, address, EIN
  - Extract Part VII Section A: board members, officers, directors, trustees (name, title, hours, compensation) → write to `entity_people`
  - Extract Schedule J compensation detail where available
  - Insert orgs into `entities` with `tier=1`, `primary_source='irs_990_xml'`
  - Insert financial history into `financials` (one row per fiscal year per org)
- [ ] Build the AI classification step
  - For each new entity, call Claude API with mission + program descriptions
  - Output: relevance score, primary focus area, secondary focus areas, issue keywords (PFAS, lead, etc.), AI summary
  - Write to `entity_classifications`
  - Also write top tags to `entity_tags` with `source='ai_classifier'` and confidence scores
- [ ] Geocode addresses (Google Maps API — already in use elsewhere)
- [ ] Spot-check 50 random orgs against the AI output. Catch obvious failures before scale.
- [ ] Run full ingestion. Expect 1,500-3,000 entities depending on NTEE code breadth.
- [ ] Generate embeddings for "similar orgs" search (one batch call per ~100 orgs)

**Out:** `entities` populated with the full universe, AI-classified, geocoded, with board member data in `entity_people`.

**Estimated AI cost:** Under $50 total at fractions of a cent per record.

**Note on data source:** The IRS bulk XML replaces the ProPublica API as the primary data source. ProPublica's API returns only aggregate financial fields and no board member data. The IRS XML contains the full 990 filing including Part VII (officers/directors), Schedule J (compensation detail), and Part III (program descriptions). The trade-off is that the XML requires a custom parser, but once built it covers all states at no additional cost and provides substantially richer data than any free API.

---

## Phase 3 — Merge curated overlay (2-3 days)

Goal: preserve the human work in the existing `organizations` table by joining it onto the new entities by EIN.

- [ ] For each row in the existing `organizations` table:
  - If EIN matches a row in `entities`: enrich the new row with the curated data (verified contact info, social URLs, mission text from website, partnership flags, geocoding if missing)
  - If no EIN match: insert as a new entity with `primary_source='planet_detroit_curation'` and `tier=1` (these are likely 501(c)(4)s, fiscally-sponsored projects, or orgs not in the IRS e-file data)
- [ ] For each curated row, also migrate the focus tags into `entity_tags` with `source='admin_review'` and `confidence=1.0` (so human tags rank above AI tags)
- [ ] Set `pd_verified=true` and copy `verified_at` / `verified_by` for all curated orgs
- [ ] Set `is_planet_champion` and `is_impact_partner` flags from the old table
- [ ] Move pending submissions into the new `submissions` table
- [ ] Run a duplicate check — same name, similar address, no EIN — and flag for human review (don't auto-merge)

**Out:** Curated data preserved as a high-confidence overlay on the new schema. Nothing lost.

---

## Phase 4 — Cutover all consumers (4-6 days)

Goal: point everything that reads or writes the old `organizations` table at the new `ecocensus.entities` schema. The old table goes read-only as a fallback.

**Consumer inventory** (from a code audit run April 2026):

| Consumer | Type | Migration step |
|---|---|---|
| `michigan-environmental-orgs/src/pages/Organizations.jsx` | Direct Supabase read | Update query + filter |
| `michigan-environmental-orgs/src/pages/SubmissionForm.jsx` | Direct Supabase write | Switch to `submissions` table |
| `michigan-environmental-orgs/src/components/OrganizationMap.jsx` | Direct Supabase read | Update join shape |
| `michigan-environmental-orgs/src/components/FocusDebugger.jsx` | Direct Supabase read | Internal tool — deprioritize |
| `michigan-environmental-orgs/admin-tools/org-admin-FINAL-with-pending.html` | Direct Supabase read + write | Replaced in Phase 5 |
| `michigan-environmental-orgs/admin-tools/org-admin-FINAL.html` | Direct Supabase read + write | Confirm if still in use; deprecate if not |
| `org-profiles/src/components/Home.jsx` | Direct REST read of all rows | Update endpoint |
| `org-profiles/src/components/OrgProfile.jsx` | Direct REST read by ID/slug | Update query |
| `org-profiles/src/components/Dashboard.jsx` | Direct REST read | Update query |
| `org-profiles/src/components/Landing.jsx` | Direct REST read | Update query |
| `org-profiles/scripts/pull_990_data.py` | Direct write (production ingest) | **Replaced by Phase 2 ingestion script — delete after cutover** |
| `org-profiles/scripts/pull_990_data_TEST.py` | Direct write (test) | Delete |
| **`ask-planet-detroit/api/main.py:212` `get_all_organizations()`** | Python backend, deployed to Railway, reads all 607 rows, drops "test"/"example" in code | **Backend update + Railway redeploy required** |
| **`ask-planet-detroit/api/main.py:860` `/api/stats` endpoint** | Counts orgs for homepage stats | Update query |
| **`civic-action-builder/src/lib/api.js:68`** | Calls `${API_BASE}/api/organizations?limit=700` | Indirect — fixed by ask-planet-detroit update; also bump the limit since the new table will exceed 700 rows |

### Cutover steps

- [ ] Update `michigan-environmental-orgs/src/supabaseClient.js` to query the `ecocensus` schema
- [ ] Update `Organizations.jsx`, `OrganizationMap.jsx`, `FocusDebugger.jsx` for the new shape
  - Joins to `entity_tags` (filter `category='focus_area'`)
  - Joins to `geographic_regions` instead of reading hardcoded JS
  - Show AI summary (`entity_classifications.ai_summary`) if no human-curated mission text
- [ ] Update `SubmissionForm.jsx` to write to `submissions`, not `entities`
- [ ] Update `org-profiles` (4 components) to query `entities` instead of `organizations`
- [ ] Update `ask-planet-detroit/api/main.py`:
  - `get_all_organizations()` — query `ecocensus.entities` with appropriate joins for focus tags
  - `/api/stats` org count — query `entities` table
  - **Redeploy to Railway** (this is the critical path for civic-action-builder)
- [ ] Bump `civic-action-builder` API limit from `700` to a higher number (or add proper pagination) — the new table will exceed 700 rows after Phase 2 IRS XML ingest
- [ ] Test all consumers in a staging/preview deployment
- [ ] Keep the old `public.organizations` table read-only for one week as fallback
- [ ] Cutover production in this order:
  1. ask-planet-detroit Railway redeploy (frees civic-action-builder)
  2. michigan-environmental-orgs Vercel deploy
  3. org-profiles / ecocensus.vercel.app deploy
- [ ] Communicate the change to MEC if they have any active queries against the old schema

**Out:** All consumers running on new schema. Old `public.organizations` archived after one-week fallback period.

---

## Phase 5 — Rebuild admin tool (3-5 days)

Goal: replace the keyword-search-and-bulk-tag admin interface with one designed for AI-assisted review.

The current admin tool (`admin-tools/org-admin-FINAL-with-pending.html`) was built around humans typing tags. The new one should be built around humans reviewing AI suggestions.

- [ ] **Review queue view**: orgs sorted by AI confidence (lowest first), with the AI's suggested tags visible. Admin clicks accept / reject / override.
- [ ] **Tag management view**: see the official 14 focus areas, plus AI-discovered tags awaiting curation. Promote useful AI tags to `is_official=true`.
- [ ] **Pending submissions view**: same as before, but now with AI pre-classification of the submitted org so the admin doesn't start from scratch.
- [ ] **Data quality view**: orgs missing fields, low-confidence classifications, duplicate candidates from Phase 3.
- [ ] **Geographic querying**: query orgs by state senate or house district using spatial joins against entity lat/lon and publicly available legislative district boundary shapefiles. This is MEC's primary membership recruitment use case — pull all environmental orgs in a target district for outreach. Implemented as a query-time spatial join, not pre-computed tags.
- [ ] **Board member search**: query `entity_people` to find shared board members across orgs, people serving on multiple environmental boards, and board composition by region.
- [ ] **Bulk actions**: keep the keyword search + bulk-tag from the old tool — it's still useful, just supplementary now.

**Out:** Admin tool optimized for AI-first workflows, with legislative-district querying and board member search.

---

## Phase 6 — Tier 2 ingestion (matches proposal "Data Expansion" phase)

Once Tier 1 is solid, layer in for-profits.

- [ ] EGLE Materials Management ArcGIS feed → `entities` with `tier=2`, `primary_source='egle_arcgis'`
- [ ] B Lab B Corp directory (Playwright scrape, MI filter)
- [ ] Michigan Sustainable Business Forum members
- [ ] Michigan Energy Innovation Business Council members
- [ ] Michigan Saves authorized contractors
- [ ] Great Lakes Renewable Energy Association members
- [ ] Good for Michigan members
- [ ] OpenStreetMap (recycling, EV charging, refill, repair, food co-ops)
- [ ] Each scraper writes to a single shared `tier 2 ingestion` script that handles dedup, AI classification, and entity creation

**Out:** Tier 2 ingested. Matches proposal Phase: "Data Expansion."

---

## Phase 7 — Tier 3 informal groups (matches proposal "Data Expansion" budget phase, then ongoing)

The initial technology build for Tier 3 is modest but real — it shares the Data Expansion phase with Tier 2. The larger ongoing cost is recurring staff or contractor time for sweeps, outreach, and verification.

### Initial build (within Data Expansion phase, ~20-40 hours)

- [ ] Build the AI-assisted Google search runner: scheduled job that runs queries against `site:facebook.com`, `site:instagram.com`, and similar with rotating keyword and geography filters; captures URLs and search-result metadata
- [ ] Build the Open Graph metadata fetcher: pulls page name, description, and image from each candidate URL via standard HTTP (no Facebook login or API access required)
- [ ] Build the EIN-matching filter: compares each candidate against `entities` to separate "already in Tier 1" from "potentially Tier 3" — the same workflow surfaces Tier 1 enrichment opportunities and Tier 3 discovery candidates
- [ ] Build the Tier 3 review queue in the admin tool: surfaces candidates with thumbnails, source attribution, and accept/reject controls
- [ ] Build a "verification refresh" workflow: periodic re-check that existing Tier 3 records are still active (page hasn't gone dark, last activity within a configurable window)
- [ ] Build a lightweight intake form for MEC staff and Planet Detroit reporters to add groups they encounter directly, bypassing the search-discovery step
- [ ] Surface `confidence_indicator` and `last_verified_active` date on every Tier 3 record in the admin tool and the public directory

### Ongoing operations (Year 2+, the larger commitment)

- [ ] Run twice-yearly focused sweeps: review the AI-surfaced candidates, decide which to accept, do additional manual research for borderline cases
- [ ] Maintain search keywords and filters as new issues emerge (PFAS, data centers, lithium mining, lake associations, etc.)
- [ ] Outreach to conservation districts and community foundations for groups that don't appear in search results
- [ ] Review of public meeting minutes from priority municipalities for groups mentioned by name
- [ ] Refresh verification on existing Tier 3 records; flag stale ones for re-confirmation or removal

**Out:** Tier 3 build complete and integrated into the entities table; ongoing sweep practice established.

---

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| EIN matching in Phase 3 misses orgs that were curated under different name/EIN | Manual review queue for unmatched curated rows; nothing gets dropped without a human looking |
| Public site breaks during cutover | Deploy to Vercel preview first; old table stays read-only for a week as fallback |
| AI misclassifies orgs at scale | Confidence threshold + human review queue; admin can override and the override sticks |
| IRS XML schema changes between filing years | Build parser with version detection; test against multiple years before full run |
| Duplicate orgs from Phase 3 merge | Flag, don't auto-merge; surface in admin review queue |
| Embeddings cost more than expected | Use a smaller model for embeddings (Voyage or text-embedding-3-small instead of vector-1536); switch if needed |
| MEC starts using the old admin tool mid-cutover | Communicate freeze to MEC; offer them the new tool the moment Phase 5 ships |

---

## Out of scope for this rebuild

- Migrating `media_mentions` data structure beyond the FK rename
- Rebuilding the civic-action-builder integration (it just reads via the existing API; should keep working with the new schema if we update the API endpoint)
- Multi-state expansion (deferred per the proposal's Long-Term Sustainability section)
- Board member network *visualization* (graph UI showing board overlaps) — deferred; the data is collected in Phase 2 but the network visualization tool is a separate build
- Grant flows (Schedule I parsing) — deferred

---

## Timeline summary

| Phase | Duration | Cumulative |
|---|---|---|
| 0. Freeze | 1 day | 1 day |
| 1. New schema, empty | 3-4 days | ~1 week |
| 2. Ingest IRS 990 XML + AI classify | 4-5 days | ~2 weeks |
| 3. Merge curated overlay | 2-3 days | ~2.5 weeks |
| 4. Cutover public site | 3-4 days | ~3 weeks |
| 5. Rebuild admin tool | 3-5 days | ~4 weeks |
| 6. Tier 2 ingestion | (proposal Phase: Data Expansion, Months 4-9) | |
| 7. Tier 3 ongoing | (proposal Phase: Data Expansion + ongoing, Months 4-9+) | |

The 2-3 week overlap window covers Phases 0-4. Phase 5 (admin tool) can run in parallel with the cutover and doesn't block the public site.
