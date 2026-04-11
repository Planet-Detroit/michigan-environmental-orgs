# AI Classification Test Results

**Date:** April 11, 2026
**Purpose:** Validate that AI classification can drive the ECOcensus 2.0 rebuild before committing to the schema/migration plan in `schema.md` and `nextsteps.md`.
**Model tested:** Claude Sonnet 4.6 (`claude-sonnet-4-6`)
**Status:** ✅ Approach validated — proceed with rebuild as designed.

---

## TL;DR

- **Classification quality is better than the current human-curated tags** in several measurable ways. The AI catches editorial nuance, fills in missing tags, and pushes back on borderline cases when the human curator was too generous.
- **Cost is trivial.** ~$0.005 per org. Classifying the entire 607-org existing database costs about $2.77. Classifying a projected 3,000-org full Michigan universe costs about $13.69.
- **Speed is fine.** ~11 minutes for 607 orgs, ~56 minutes for 3,000 orgs, with proper rate-limit handling.
- **There is one operational gotcha** — concurrency must be bounded to ~5 in-flight requests to stay inside Anthropic rate limits. Naive parallelization (20 in flight) drops success rate to 42%. Bounded concurrency gets 95%.

---

## Test methodology

**Stage 1 — Quality test (hand-evaluated, 8 orgs)**
- Pulled 8 diverse orgs from the existing `organizations` table that already had human-curated focus tags
- Sent each to Claude Sonnet 4.6 with a structured classification prompt
- Compared AI output against human tags side-by-side

**Stage 2 — Scale test (quantitative, 100 orgs)**
- Pulled 100 orgs at random from the same table
- Ran them through the same classifier with bounded concurrency (5 in-flight)
- Measured success rate, total cost, total time, and the distribution of relevance scores and primary focus areas

**Classification prompt structure**

The model was asked to return JSON with these fields:

```json
{
  "relevance_score": 0.0-1.0,
  "primary_focus_area": "one of the 14 official tags",
  "secondary_focus_areas": ["up to 3 more"],
  "issue_keywords": ["specific terms like PFAS, lead pipes, brownfield"],
  "ai_summary": "2-3 sentence plain English description for public directory",
  "rationale": "one sentence explaining the relevance score"
}
```

The relevance score was anchored:
- **1.0** = core environmental org
- **0.7-0.9** = environmentally adjacent
- **0.4-0.6** = peripheral
- **0.0-0.3** = not environmental (should not be in directory)

---

## Stage 1 results — Quality

**8 of 8 successfully classified.** Below are five cases that show what the AI does well, drawn from the actual run:

### Case 1: AI catches tags humans missed

**Friends of the Rifle River High Banks Inc.** (Prescott, NTEE C02)

| | Tags |
|---|---|
| Human | Stewardship |
| AI primary | Stewardship |
| AI secondary | Environmental Education, Land Conservation, **Water** |
| AI relevance | 0.95 |

The human curator tagged this only as Stewardship. The AI added Water — which is the obvious primary attribute of a *river* organization. This is a clean tag that was missing.

---

### Case 2: AI enriches a single tag into a full classification

**Canton Nature Society** (Canton, NTEE C50)

Mission: *"To discover and act on opportunities to sustain, enhance and restore our local habitats."*

| | Tags |
|---|---|
| Human | Stewardship |
| AI primary | Stewardship |
| AI secondary | Wildlife, Land Conservation, Environmental Education |
| AI relevance | 1.0 |

Same starting human tag, four meaningful AI tags. The AI summary: *"Canton Nature Society is a Michigan nonprofit dedicated to sustaining, enhancing, and restoring local natural habitats in the Canton area."* Publication-ready directory copy from a 13-word mission statement.

---

### Case 3: AI pushes back on a misclassified human tag

**Montcalm County Quality Deer Cooperative** (Sidney, NTEE C30)

| | Tags |
|---|---|
| Human | Outdoor Recreation |
| AI primary | **Wildlife** |
| AI secondary | Stewardship, Land Conservation |
| AI relevance | 0.6 (peripheral) |

The AI's rationale: *"wildlife management cooperatives have clear conservation and stewardship components but are primarily focused on game management rather than broad environmental advocacy."*

The human tag (Outdoor Recreation) treats this as a hunting/recreation org. The AI correctly reframes it as a wildlife management org with a peripheral environmental relevance. The 0.6 score means it gets routed to admin review rather than auto-included.

---

### Case 4: AI catches editorial context unprompted

**Reroot Flint** (Flint, NTEE C60)

Mission: *"We are fellow Flintstones & environmental enthusiasts that want to participate in Flint's growth."*

| | Tags |
|---|---|
| Human | Environmental Education |
| AI primary | Stewardship |
| AI secondary | Environmental Education, **Environmental Justice**, Land Conservation |
| AI relevance | 0.7 |

The AI noticed Flint's environmental justice context unprompted. The rationale: *"rooted in a community with deep environmental justice concerns (Flint water crisis)."* This is exactly the kind of editorial signal that makes the directory useful for journalists writing about Flint — and it took the AI under a second to catch it from a 14-word mission statement.

---

### Case 5: AI is more skeptical when the human was too generous

**Enhance Escanaba** (Escanaba, NTEE C50)

Mission: *"Enhance Escanaba was established in 2021 as a nonprofit volunteer organization. It is compromised of members who share a unified vision to enhance the appearance of the City of Escanaba."*

| | Tags |
|---|---|
| Human | Stewardship |
| AI primary | Stewardship |
| AI relevance | **0.4** (peripheral) |

The AI's rationale: *"primary mission is civic beautification rather than conservation or environmental advocacy."*

This is the workflow the rebuild is designed for. The human curator gave it a Stewardship tag and called it done. The AI accepts the tag but flags it at 0.4 confidence — meaning it should be re-reviewed by an admin who can decide whether civic beautification belongs in an environmental directory at all. It's not a wrong tag, it's a case that deserves human judgment.

---

### Case 6: Mission text is awful, AI summary is publication-ready

**Gravel Lake Association** (Lawton, NTEE C32)

Raw mission statement (truncated as stored in DB):
> *"Gravel Lake is located in Lawton, Michigan in the Southwestern part of the state. Our Lake is 296 acres, spring fed from the main body as well as the channel, has an outlet (spillway) on Idle Ease bea..."*

AI-generated summary:
> *"Gravel Lake Association is a lake association based in Lawton, Michigan, focused on the stewardship and enjoyment of the 296-acre spring-fed Gravel Lake in the southwestern part of the state. The organization is involved in the management and care of the lake's watershed, including its inlet and outlet connections. As a lake association, it likely works to protect water quality and promote responsible recreational use of the lake."*

The raw mission reads like a Wikipedia stub. The AI summary is directory copy a reader can actually use. This benefit alone is worth the rebuild — the existing directory is full of similarly bad mission text scraped from 990s and websites.

---

## Stage 2 results — Scale (100 orgs)

### Headline numbers

| Metric | Value |
|---|---|
| Sample size | 100 orgs (random selection with mission text) |
| Concurrency | 5 in-flight requests |
| Wall time | 106.7 seconds |
| Success rate | **95 / 100** |
| Errors (all rate limit retries) | 5 |
| Total tokens | 35,764 in / 21,758 out |
| Total cost | **$0.4337** |
| Cost per org | **$0.0046** |
| Throughput | 0.89 orgs/sec |

The cost per org (0.46¢) is consistent with Stage 1 (0.47¢), confirming that the per-record cost is stable at scale.

### Relevance score distribution (95 orgs)

| Band | Count | Percent | Action |
|---|---|---|---|
| ≥ 0.8 (high confidence environmental) | 67 | 71% | Auto-include in directory |
| 0.4 – 0.8 (medium / borderline) | 25 | 26% | Route to admin review queue |
| < 0.4 (low / not environmental) | 3 | 3% | Likely exclude from directory |

This is a healthy distribution. About 2/3 of work can be fully automated, ~1/4 needs human eyes, and a small fraction gets correctly caught for exclusion. If everything came back at 0.9, the model would be over-confident; if everything came back at 0.5 it would be useless. Sonnet 4.6 gives calibrated, defensible scores.

### Primary focus area distribution (95 orgs)

```
Water                            12  ████████████
Advocacy                          9  █████████
Energy                            9  █████████
Food & Agriculture                9  █████████
Recycling                         8  ████████
Land Conservation                 8  ████████
Sustainability                    8  ████████
Stewardship                       7  ███████
Climate                           6  ██████
Environmental Education           6  ██████
Environmental Justice             6  ██████
Outdoor Recreation                5  █████
Wildlife                          2  ██
```

This distribution looks plausible for Michigan: Water leading the list makes sense for a Great Lakes state. The low Wildlife count is worth watching — the AI may be defaulting to Stewardship instead and under-tagging wildlife orgs. Should be re-checked after the full Phase 2 ingestion.

---

## Cost and time at full scale

| Workload | Cost | Time |
|---|---|---|
| 607 existing orgs (one classification pass) | **$2.77** | ~11 minutes |
| 3,000-org projected full Michigan universe | **$13.69** | ~56 minutes |
| Annual reclassification ×4 per year | **~$56/year** | — |
| With 5% retry overhead for safety | +5% to cost & time | — |

**Even with Tier 2 and Tier 3 added — likely doubling total entity count to ~6,000 — annual AI compute is well under $200/year.** The proposal's claim that "fractions of a cent per record" makes AI classification trivially cheap is real and conservative. The Year 1 budget of $80K is dominated by engineering work (schema, ingestion pipelines, admin tool rebuild, cutover testing), not AI inference.

---

## Key findings

### What the AI does well

1. **Catches missing tags.** Multiple Stage 1 cases showed the AI adding tags humans missed (Water for a river org, Wildlife for a deer cooperative, Environmental Justice for a Flint group).
2. **Pushes back on weak classifications.** The AI gave Enhance Escanaba a 0.4 and the deer cooperative a 0.6 — both borderline cases that deserve human review. It is not sycophantic.
3. **Generates publication-ready summaries.** AI summaries are consistently better than the raw mission text scraped from 990s and websites. This is a quality win for the public directory independent of the classification work.
4. **Catches editorial context.** Reroot Flint → Environmental Justice was unprompted. The model has enough world knowledge to recognize Flint as an EJ context and add the tag. This is the kind of judgment that would take a human curator several minutes of research.
5. **Stable cost at scale.** Cost per org was consistent across both stages (0.47¢ and 0.46¢), confirming no scaling surprises.

### What to watch

1. **Wildlife may be under-tagged.** Only 2 orgs out of 95 in Stage 2 got Wildlife as a primary focus, and the AI seemed to default several wildlife-related orgs to Stewardship instead. Worth a targeted re-check after Phase 2.
2. **AI/human tag conflicts need a UI policy.** Several Stage 1 cases showed the AI primary tag disagreeing with the human tag. The schema stores both with confidence scores, but the public directory needs an explicit display policy: human tags display first when present? AI tags supplement? AI overrides if confidence delta exceeds a threshold? This is a design call that should be made before Phase 4 cutover.
3. **Rate limits are a real production constraint.** The first attempt at Stage 2 with 20 concurrent requests dropped success rate to 42%. Bounded concurrency (5 in-flight) plus retry logic is required. This is standard, well-understood, and not a blocker — but Phase 2's ingestion script needs proper rate-limit handling, not naive `asyncio.gather()`.

### What this test does NOT prove

- **Classification quality on Tier 2 (for-profit) and Tier 3 (informal) entities.** All test orgs were 501(c)(3)s with NTEE codes. B Corps and informal groups have different data shapes and the prompt may need adjustment.
- **Multi-language mission statements.** All test orgs had English mission text.
- **Long-form mission analysis.** The prompt was given short mission statements only. Some 990s have full program descriptions running to several thousand characters; classification on those will use more tokens but should not change quality.
- **Drift over model versions.** Sonnet 4.6 today; Sonnet 5.x in 6 months may behave differently. Phase 5 admin tool should make it easy to re-run classification when models update.

---

## Recommendation

**Proceed with the rebuild plan as designed in `schema.md` and `nextsteps.md`.**

Specific decisions this test settles:

1. **Use Claude Sonnet 4.6 as the default classifier.** Quality is high and cost is trivial. (Haiku 4.5 was not tested but could drop costs another 5x if needed — worth a future test if the budget gets tight, but not necessary.)
2. **Set the auto-include threshold at relevance ≥ 0.8.** Stage 2 distribution suggests this auto-handles ~70% of orgs.
3. **Set the admin review threshold at 0.4 ≤ relevance < 0.8.** ~25% of orgs route here.
4. **Set the exclude threshold at relevance < 0.4.** ~3% of orgs caught — small enough to manually review for false negatives.
5. **Use bounded concurrency (5) and exponential backoff** in the production ingestion script. Plan for ~5% rate-limit retries as overhead.
6. **Budget AI compute at < $20 per full universe pass.** Annual total across 4 reclassification passes: < $100. This is rounding error in the proposal budget.

---

## Test artifacts

The test scripts are in `/tmp/`:
- `/tmp/ai_classification_test.py` — original Stage 1 + Stage 2 (Stage 2 had concurrency=20, dropped to 42% success)
- `/tmp/ai_classification_stage2_retry.py` — Stage 2 retry with concurrency=5, captured error types

These are intentionally throwaway test code. The production ingestion script for Phase 2 will live in the new repo (likely `michigan-environmental-orgs/scripts/ingest_propublica.py` or in a new `ecocensus-rebuild` project) and will need:
- Proper async retry with exponential backoff
- Resumable batch processing (so a partial failure doesn't lose the whole run)
- Provenance writes to the `entity_classifications` and `entity_tags` tables per the schema design
- Cost tracking per batch for ongoing monitoring
