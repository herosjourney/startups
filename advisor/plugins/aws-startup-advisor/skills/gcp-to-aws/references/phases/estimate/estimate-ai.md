# Estimate Phase: AI Workload Cost Analysis

> Loaded by estimate.md when aws-design-ai.json exists.

**Execute ALL steps in order. Do not skip or optimize.**

## Pricing Mode

The parent `estimate.md` selects the pricing mode before loading this file.

**Price lookup order:**

1. **`shared/pricing-cache.md` (primary)** — Look up Bedrock model pricing and source provider pricing by table. Set `pricing_source: "cached"`.
2. **MCP (secondary)** — If a model is NOT in pricing-cache.md and MCP is available, query `get_pricing("AmazonBedrock", ...)` with model filter and the user's target region. Set `pricing_source: "live"`.
3. **Cache after MCP failure** — If MCP was attempted but failed, and the model IS in the cache, use the cached price. Set `pricing_source: "cached_fallback"`.
4. **Unavailable** — If a model is NOT in the cache AND MCP failed, set `pricing_source: "unavailable"` and warn the user.

For typical migrations (Claude, Llama, Nova, Mistral, DeepSeek, Gemma, OpenAI gpt-oss, Gemini source pricing), ALL prices are in `pricing-cache.md`. Zero MCP calls needed.

**Model lifecycle:** When building the model comparison table, check `references/shared/ai-model-lifecycle.md` and apply the 90-day exclusion rule:

- **Excluded** (≤90 days to EOL): omit entirely from `model_comparison`, `recommended_model`, and `backup_model`.
- **Legacy** (>90 days to EOL): include in `model_comparison` with `(Legacy — EOL YYYY-MM-DD)` annotation. Do not select as `recommended_model` unless no Active alternative exists.
- **Active**: no restrictions.

## Prerequisites

Read from `$MIGRATION_DIR/`:

- **`ai-workload-profile.json`** — `current_costs.monthly_ai_spend`, `current_costs.services_detected`, `models[]`, `metadata.profile_source`, `summary.inferred_from_iac`
- **`openai-usage-profile.json`** (if present) — `summary.monthly_cost_usd`, `usage_by_model[]` (real per-model input/output token counts from the OpenAI Admin API)
- **`preferences.json`** — `ai_constraints.ai_token_volume.value`, `ai_constraints.ai_capabilities_required.value`
- **`aws-design-ai.json`** — `metadata.ai_source`, `ai_architecture.honest_assessment`, `ai_architecture.tiered_strategy`, `ai_architecture.bedrock_models[]` (with `source_provider_price`, `bedrock_price`, `honest_assessment`), `ai_architecture.capability_mapping`

**Traditional-AI workloads (`design_blocks[]` entries with `target_aws_service` set):** capability `document_extraction`, `image_analysis`, or `speech_transcription` (→ Textract, Rekognition, Transcribe) are priced per-page/per-image/per-minute, not per-token. Skip these blocks in Parts 1–3 below (which are token-cost logic only) — cost them separately in **Part 3.5**.

---

## Part 1: Establish Current GCP AI Costs

Determine current AI spending from the best available source:

1. **OpenAI usage API data (most preferred)** — Use `summary.monthly_cost_usd` from `openai-usage-profile.json` (real billed spend, captured from the provider). `usage_by_model[]` also supplies actual input/output token counts and ratio for Part 2.
2. **Billing data** — Use `current_costs.monthly_ai_spend` from `ai-workload-profile.json`
3. **Estimated from token volume** — Use `ai_constraints.ai_token_volume.value` from `preferences.json` with Gemini pricing from `pricing-cache.md` (under "Source Provider Pricing"). Apply 60/40 input/output ratio if actual ratio unknown.
4. **None available** — Note in output and present model comparison at multiple volume tiers so user can find their range.

**IaC-only profile:** If `metadata.profile_source` is `iac_vertex` or `summary.inferred_from_iac` is true and billing/token data is missing, state explicitly that **current GCP AI spend is unverified** and widen uncertainty bands (use the same multi-tier comparison approach as in case 3).

---

## Part 2: Build Model Comparison Table

Calculate the monthly Bedrock cost for **every viable model** at the user's token volume.

**Token volume mapping** (from `ai_token_volume` in `preferences.json`):

| `ai_token_volume` | Input tokens/month | Output tokens/month | Ratio |
| ----------------- | ------------------ | ------------------- | ----- |
| `"low"`           | 6M                 | 4M                  | 60/40 |
| `"medium"`        | 60M                | 40M                 | 60/40 |
| `"high"`          | 600M               | 400M                | 60/40 |
| `"very_high"`     | 6B                 | 4B                  | 60/40 |

If design or discover phase has more specific token estimates, use those instead. In particular, when `openai-usage-profile.json` exists, use its `usage_by_model[]` actual monthly input/output token totals (and actual ratio) instead of the tier table — real observed volume always beats a tier midpoint.

**Cost formula:** `Monthly = (input_tokens / 1M × input_rate) + (output_tokens / 1M × output_rate)`

**Long-context surcharge:** If `ai_critical_feature = "ultra_long_context"` in `preferences.json`, Claude models charge 2x the standard input rate for tokens beyond 200K context. Apply the surcharge to the portion of input tokens that exceeds 200K per request. If per-request token counts are unknown, assume 50% of input tokens fall in the long-context tier as a conservative estimate.

**Comparison table columns:** Model, Bedrock Monthly, vs Source Provider ($ and %), vs Current GCP, Quality, Capabilities Match (checked against `ai_capabilities_required`).

Include source provider pricing from `aws-design-ai.json` → `bedrock_models[].source_provider_price`.

If Bedrock is more expensive for the recommended model, flag prominently.

If embeddings are needed, add a separate line (additive to primary model cost).

---

## Part 3: Recommended Model Cost Breakdown

Using the model selected in the design phase, show:

- Input tokens × rate, output tokens × rate, embeddings × rate (if applicable)
- Total monthly cost
- Comparison to current GCP spend (monthly and annual difference)
- Backup model cost for comparison

---

## Part 3.5: Traditional AI Services (Textract, Rekognition, Transcribe)

For every `design_blocks[]` entry with `target_aws_service` set (see Prerequisites above), compute cost using per-unit pricing from `references/shared/pricing-cache.md` → "Traditional AI Services (Non-Bedrock, Non-Token Pricing)". These are NOT token-priced — do not apply Part 2's cost formula to them.

### Step 1: Establish monthly volume (pages / images / minutes)

Volume is not captured by any existing Clarify question (`ai_token_volume` measures LLM tokens, not pages/images/minutes) — derive it in this order:

1. **GCP billing data (preferred)** — If `ai-workload-profile.json` → `current_costs.services_detected` includes "Document AI", "Vision AI", or "Speech-to-Text" (from `discover-billing.md` Pattern 3.4) and a per-service dollar amount is available, back into a volume estimate by dividing the GCP monthly spend for that service by the corresponding GCP per-unit rate in `pricing-cache.md`'s GCP comparison tables. Example: $45/month on GCP Vision Label Detection ÷ $1.50/1,000 units ≈ 30,000 images/month (ignoring the first-1,000-free tier, which is immaterial at this volume). State the assumption: "Volume estimated from GCP billing; actual AWS volume may differ if usage patterns change post-migration."
2. **App-code call-site count (fallback)** — If `workloads[].call_sites` shows the number of distinct call locations but no volume data exists, do NOT fabricate a number. Set `volume_confidence: "unknown"` and present cost at 3 illustrative volume tiers instead of a single figure (same multi-tier approach as Part 1's "None available" case): 10K/mo, 100K/mo, 1M/mo units.
3. **User-supplied (if asked)** — Do not add a new Clarify question for this. If the user volunteers a volume figure in conversation during Estimate, use it and set `chosen_by: "user"`.

### Step 2: Apply the matching rate table

| `target_aws_service` | Rate table to use                                                            | Unit   |
| -------------------- | ---------------------------------------------------------------------------- | ------ |
| `textract`           | `pricing-cache.md` → "AWS Textract (per 1,000 pages, tiered)"                | page   |
| `rekognition`        | `pricing-cache.md` → "AWS Rekognition Image (per 1,000 images, tiered)"      | image  |
| `transcribe`         | `pricing-cache.md` → "AWS Transcribe (per minute, tiered by monthly volume)" | minute |

**Textract API selection:** Use the specific `target_aws_service` sub-variant recorded by Design (`references/design-refs/ai.md` maps GCP processor type → specific Textract API — e.g., `AnalyzeExpense` for invoices, `DetectDocumentText` for plain OCR). If Design only recorded `"textract"` without an API sub-type, default to `AnalyzeDocument` — Forms + Tables (the most capable, and most expensive, tier) and flag: "Textract API variant not specified — costed at the Forms+Tables rate; confirm the actual processor type to refine this estimate, since per-page cost varies up to 47x by API."

**Apply the tier breakpoints** — e.g., for Textract `DetectDocumentText` at 1.5M pages/month: `(1,000,000 × $0.0015) + (500,000 × $0.0006) = $1,500 + $300 = $1,800/month`.

### Step 3: Compare against current GCP cost

Use the matching GCP rate table in `pricing-cache.md` (same section) to compute the current GCP cost at the same volume, using the GCP service actually detected (Document AI processor type / Vision API feature / Speech-to-Text tier). If GCP billing data gave the volume in Step 1, this is the customer's actual current spend — use it directly instead of recomputing, and note any variance if the recomputed figure differs meaningfully (>15%) from billed spend, since that signals the derived volume estimate may be off.

### Step 4: Emit output

Add a `traditional_ai_costs[]` array (see Output schema below) — one entry per `design_blocks[]` traditional-AI workload:

```json
{
  "workload_id": "wl_8b4e91",
  "target_aws_service": "textract",
  "api_variant": "AnalyzeExpense",
  "monthly_volume": 100000,
  "volume_unit": "pages",
  "volume_confidence": "estimated_from_billing",
  "aws_monthly_cost": 1000.00,
  "gcp_monthly_cost": 833.33,
  "cost_comparison_pct": "+20%",
  "note": "AWS is more expensive at this volume for AnalyzeExpense vs GCP Expense parser flat rate; AWS's tiered discount only helps above 1M pages/month."
}
```

**If no volume could be established at all** (no billing, no user input, call-site count only): set `volume_confidence: "unknown"`, omit `aws_monthly_cost`/`gcp_monthly_cost`/`cost_comparison_pct` (do not fabricate a number), and instead populate a `cost_at_illustrative_volumes[]` sub-array with the 3 tiers from Step 1.2.

**These entries do NOT feed into `cost_comparison`, `roi_analysis`, or `recommendation` (Parts 5–7)** — those remain LLM-token-cost-only, matching the Scope Boundary at the bottom of this file. Traditional-AI cost is presented as a separate line item in the summary (see Present Summary, below), not blended into the primary migrate/stay verdict, since the two are priced on fundamentally different bases and a customer with one cheap Bedrock model and one expensive Textract migration should see both facts distinctly rather than a single averaged number.

---

## Part 4: Human One-Time Migration Costs (Out of Scope)

**Do not** present human labor, contractors, professional services, or engineering effort as one-time migration **costs** or budget line items (no dollar figures, no "budget for people work" lists, no "one-time migration cost" categories for implementation).

Populate `migration_cost_considerations.categories` as an **empty array** `[]`. Use `migration_cost_considerations.note` to state that human and professional-services one-time migration costs are intentionally excluded from this advisor.

**Technical integration complexity** (for internal JSON and risk context only — not framed as money):

From `ai-workload-profile.json`, record non-monetary factors in `migration_cost_considerations.complexity_factors[]` as short strings, for example:

- `integration.pattern = "framework"` → lower integration touch surface
- `integration.pattern = "direct_sdk"` → moderate SDK and API pattern changes
- `integration.pattern = "rest_api"` → higher endpoint, auth, and parsing changes
- `summary.total_models_detected` > 3 → multi-model coordination
- `quota_risk = "high"` (from `aws-design-ai.json`) → Bedrock quota increase required before migration; allow 1–5 business days (see `shared/bedrock-quotas.md`)

Do **not** repeat these as "costs" in the user-facing summary.

---

## Part 5: ROI Analysis

Present the monthly and annual cost difference between current GCP AI spend and projected Bedrock cost:

- **If Bedrock is cheaper**: present monthly and annual savings clearly
- **If Bedrock is more expensive**: state clearly, justify with non-cost benefits or note "not justified if cost is the only priority"

Reference `aws-design-ai.json` → `honest_assessment`. If `"recommend_stay"`, present prominently.

**Non-cost benefits to present:** model flexibility (30+ models), prompt caching (Claude, 90% savings), AWS ecosystem (Guardrails, Knowledge Bases, Agents), vendor diversification, multi-model strategy.

**Note:** Human/professional-services one-time migration costs are intentionally out of scope for this advisor and excluded from ROI calculations.

---

## Part 6: Cost Optimization Opportunities

Present applicable optimizations with estimated savings:

| Optimization               | Savings | Applies When                                        |
| -------------------------- | ------- | --------------------------------------------------- |
| Model downsizing / tiering | 60-87%  | High volume, premium model selected                 |
| Prompt caching (Claude)    | ~30%    | Repeated system prompts                             |
| Batch API                  | 50%     | Non-real-time workloads (`ai_latency = "flexible"`) |
| Provisioned throughput     | Varies  | Token volume > 100M/month, predictable traffic      |
| Input token reduction      | 10-30%  | Prompt optimization, shorter context                |
| Multi-model tiered routing | 60-87%  | High/very-high volume, `tiered_strategy` in design  |

For each applicable optimization, calculate before/after monthly cost and show an `optimized_projection` (best-case monthly with all optimizations).

**Post-migration optimization (do not surface during migration):** Model distillation — training a smaller, faster student model from a larger teacher model — can reduce inference costs up to ~75% for high-volume, stable workloads. Requires production traffic, labeled examples, and a teacher/student eval loop. Mention in the estimate summary as: "Once you have 2–4 weeks of Bedrock production traffic, consider model distillation to further reduce costs. See docs.aws.amazon.com/bedrock/latest/userguide/model-distillation.html." Do not recommend distillation before the startup has migrated and validated their workload.

---

## Part 7: Migration Recommendation (REQUIRED)

Produce a clear migrate/stay/optimize verdict for the AI workload migration. This is the AI-only equivalent of `estimate-infra.md` Part 7.

**Decision logic:**

| Condition                                                                                                                         | Verdict             | `recommendation.path` |
| --------------------------------------------------------------------------------------------------------------------------------- | ------------------- | --------------------- |
| Bedrock cheaper AND capabilities match                                                                                            | Migrate             | `migrate_optimized`   |
| Bedrock more expensive BUT non-cost benefits justify (vendor diversification, Guardrails, multi-model) AND user priority ≠ `cost` | Migrate with caveat | `migrate_optimized`   |
| Bedrock more expensive AND user priority = `cost` AND no compelling non-cost reason                                               | Stay                | `stay`                |
| Design `honest_assessment` = `recommend_stay`                                                                                     | Stay                | `stay`                |
| Mixed (some workloads cheaper, some not)                                                                                          | Migrate selectively | `migrate_phased`      |

**Output fields** (add to `estimation-ai.json` top-level):

```json
"recommendation": {
  "path": "migrate_optimized | migrate_phased | stay",
  "path_label": "Migrate to Bedrock | Migrate selectively | Stay on current provider",
  "migrate_if": "Brief condition under which migration makes sense (1 sentence)",
  "stay_if": "Brief condition under which staying makes sense (1 sentence)",
  "confidence": "high | medium | low",
  "rationale": "2-3 sentence justification citing cost delta and non-cost factors"
}
```

**Rules:**

- MUST emit `recommendation` — never omit. If data is insufficient, set `confidence: "low"` and state why in `rationale`.
- If `honest_assessment` from `aws-design-ai.json` says `recommend_stay`, `recommendation.path` MUST be `stay` regardless of cost numbers.
- For multi-workload runs: if some workloads favor migration and others don't, use `migrate_phased` and list which workloads to migrate vs. keep in `rationale`.

---

## Output

Write `estimation-ai.json` to `$MIGRATION_DIR/`.

**Schema — top-level fields:**

| Field                           | Type   | Description                                                                                                                                                                                                                                                                                                                            |
| ------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `phase`                         | string | `"estimate"`                                                                                                                                                                                                                                                                                                                           |
| `timestamp`                     | string | ISO 8601                                                                                                                                                                                                                                                                                                                               |
| `pricing_source`                | string | `"cached"` or `"live"`                                                                                                                                                                                                                                                                                                                 |
| `accuracy_confidence`           | string | `"±5-10%"` or `"±15-25%"`                                                                                                                                                                                                                                                                                                              |
| `current_costs`                 | object | `source`, `gcp_monthly_ai_spend`, `services[]`                                                                                                                                                                                                                                                                                         |
| `token_volume`                  | object | `source`, `monthly_input_tokens`, `monthly_output_tokens`, ratio                                                                                                                                                                                                                                                                       |
| `model_comparison`              | array  | All viable models: `model`, `monthly_cost`, `vs_current`, `quality`, `capabilities_match`, `missing_capabilities[]`                                                                                                                                                                                                                    |
| `recommended_model`             | object | `model`, `monthly_cost`, `breakdown` (input/output/embeddings), `rationale`                                                                                                                                                                                                                                                            |
| `backup_model`                  | object | `model`, `monthly_cost`, `rationale`                                                                                                                                                                                                                                                                                                   |
| `embeddings`                    | object | `model`, `monthly_cost`, `monthly_tokens`, `note` (if applicable)                                                                                                                                                                                                                                                                      |
| `cost_comparison`               | object | `current_gcp_monthly`, `projected_bedrock_monthly`, `monthly_difference`, `annual_difference`, `percent_change`                                                                                                                                                                                                                        |
| `migration_cost_considerations` | object | `categories[]` (always `[]`), `complexity_factors[]` (technical integration only), `note` (must state human/pro costs excluded)                                                                                                                                                                                                        |
| `roi_analysis`                  | object | `monthly_cost_delta`, `annual_cost_delta`, `justification`, `non_cost_benefits[]`                                                                                                                                                                                                                                                      |
| `optimization_opportunities`    | array  | `opportunity`, `potential_savings_monthly`, `implementation_effort`, `description`                                                                                                                                                                                                                                                     |
| `optimized_projection`          | object | `monthly_with_optimizations`, `vs_current`, `note`                                                                                                                                                                                                                                                                                     |
| `recommendation`                | object | `path`, `path_label`, `migrate_if`, `stay_if`, `confidence`, `rationale` (see Part 7)                                                                                                                                                                                                                                                  |
| `traditional_ai_costs`          | array  | Present only if any `design_blocks[]` entry has `target_aws_service` set. Per-workload: `workload_id`, `target_aws_service`, `api_variant`, `monthly_volume`, `volume_unit`, `volume_confidence`, `aws_monthly_cost`, `gcp_monthly_cost`, `cost_comparison_pct`, `note` (see Part 3.5). `[]` or absent if no traditional-AI workloads. |

All cost values are numbers, not strings. Output must be valid JSON.

## Validation Checklist

- [ ] `recommendation` field is present with non-empty `path`, `path_label`, `migrate_if`, `stay_if`, and `rationale`
- [ ] `recommendation.path` is one of: `migrate_optimized`, `migrate_phased`, `stay`
- [ ] If Design `honest_assessment` = `recommend_stay`, then `recommendation.path` = `stay`
- [ ] `model_comparison` includes ALL viable Bedrock models, not just recommended
- [ ] Legacy models in `model_comparison` are annotated with EOL dates (per `shared/ai-model-lifecycle.md`)
- [ ] `recommended_model` is an Active model (not Legacy) unless no Active alternative exists
- [ ] Every model has `capabilities_match` checked against `ai_capabilities_required`
- [ ] `recommended_model.rationale` references user's priority, preference, and volume
- [ ] `roi_analysis` is honest — if migration increases cost, says so
- [ ] `optimization_opportunities` only includes strategies relevant to user's workload
- [ ] No compute, database, storage, or networking costs (those belong in `estimate-infra.md`)
- [ ] `migration_cost_considerations.categories` is `[]` — no human one-time migration costs presented
- [ ] Every `design_blocks[]` entry with `target_aws_service` set has a corresponding `traditional_ai_costs[]` entry (or is covered by `cost_at_illustrative_volumes[]` if volume is unknown) — none silently dropped
- [ ] `traditional_ai_costs[]` entries do NOT factor into `cost_comparison`, `roi_analysis`, or `recommendation` — those remain LLM-token-only per the Scope Boundary
- [ ] If `volume_confidence` is `"unknown"` for a traditional-AI workload, `aws_monthly_cost`/`gcp_monthly_cost` are omitted (not fabricated) and `cost_at_illustrative_volumes[]` is populated instead

## Completion Handoff Gate (Fail Closed)

Before returning control to `estimate.md`, require:

- `estimation-ai.json` exists and passes the Validation Checklist above.

If this gate fails: STOP and output: "estimate-ai did not produce a valid `estimation-ai.json`; do not complete Phase 4."

## Present Summary

After writing `estimation-ai.json`, present under 25 lines:

1. **Pricing source and accuracy**: State whether prices came from cache or live API, and the accuracy range (±15-25% for AI models from cache, ±5-10% from live API). Example: "AI model estimates based on cached pricing (2026-03-07), accuracy ±15-25%."
2. Current GCP AI spend vs estimated monthly Bedrock cost (recommended model)
3. Model comparison table: model name, estimated monthly cost, vs source provider %, capabilities match
4. Recommended model with estimated monthly cost breakdown
5. If migration increases cost: flag honestly with non-cost justification
6. Top 2-3 optimization opportunities with potential estimated monthly savings
7. Optimized projection
8. **If `traditional_ai_costs[]` is non-empty:** present as a separate line item, not blended into the LLM cost comparison above — e.g., "Document extraction (Textract, AnalyzeExpense): Est. $1,000/mo at ~100K pages/mo (vs Est. $833/mo on GCP Expense parser, +20%)." If `volume_confidence` is `"unknown"` for any entry, say so explicitly and present the illustrative-volume range instead of a single figure.

**Cost labeling rule:** All dollar figures presented to the user MUST be labeled as "estimated monthly costs" or prefixed with "Est." — never present raw dollar amounts as if they are exact.

## Generate Phase Integration

The Generate phase uses `estimation-ai.json`:

1. **`recommended_model`** — Which Bedrock model to provision and test
2. **`migration_cost_considerations`** — `complexity_factors[]` only for integration risk context; **never** present human one-time migration **costs** to the user (`categories` stays `[]`)
3. **`optimization_opportunities`** — Which optimizations to implement and when
4. **`cost_comparison`** — Cost monitoring targets and alerts in production
5. **`model_comparison`** — Fallback options if recommended model doesn't meet quality bar

## Scope Boundary

**This phase covers financial analysis ONLY for AI workloads.**

FORBIDDEN — Do NOT include compute, database, storage, networking cost calculations, infrastructure provisioning, code migration examples, or detailed migration timelines.
