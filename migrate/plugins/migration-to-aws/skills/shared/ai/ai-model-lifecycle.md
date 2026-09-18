# Bedrock Model Lifecycle Awareness

> Canonical Bedrock model Active/Legacy/EOL registry and the 90-day exclusion
> rule. Source-cloud agnostic: it is a property of Bedrock's model catalog.
> Vendored into each consuming skill as
> `references/vendored/ai/ai-model-lifecycle.md` and kept byte-identical by
> `shared:check`; edit HERE, then run `shared:sync`.

References:

- [Models launched on or after 2026-09-07](https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html)
- [Models launched before 2026-09-07](https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle-legacy.html)

Models on Bedrock move through three states: **Active** → **Legacy** →
**End-of-Life (EOL)**. After EOL, the model is unavailable and requests fail.

The notice policy depends on launch date:

- Models launched **before 2026-09-07** follow the original policy: at least six
  months in Legacy before EOL. For EOL dates after 2026-02-01, public extended
  access starts after at least three months in Legacy and provider pricing may
  increase.
- Models launched **on or after 2026-09-07** use their model card: each card
  declares an `EOL no sooner than` date and either a six-month or **45-day**
  Legacy period. When Legacy begins, the card gains the actual EOL date.

Do not apply the old six-month assumption to a post-2026-09-07 model.

---

## Lifecycle States (Not the Same Thing)

| State      | What it means                                                                                                                                                                                                               | Usable?                |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| **Active** | Provider is actively maintaining the model. Full feature access.                                                                                                                                                            | Yes                    |
| **Legacy** | Deprecated. Still works for existing users, but new Provisioned Throughput cannot be created, new customers cannot onboard, pricing may increase during public extended access, and the model is on a countdown to removal. | Yes, with restrictions |
| **EOL**    | Model is removed. All inference requests fail.                                                                                                                                                                              | **No**                 |

**Legacy does not mean unavailable** — it means the model still functions today but has a firm expiration date. EOL means unavailable.

---

## Selection Rules

### Rule 1: Active models only for new migrations

**New migrations must target Active models only.** Do not recommend a Legacy or EOL model as the primary selection for any new migration, even if it is cheaper.

### Rule 2: 90-day exclusion zone

**Models within 90 days of their EOL date must be excluded from all recommendation and comparison tables.** A migration takes weeks or months to plan, test, and deploy. Recommending a model that will be unavailable before the migration is production-ready is harmful.

- **Excluded** = do not list in "Best Bedrock Match" columns, tiered strategy tables, or `recommended_model` / `backup_model` fields.
- These models may still appear in the pricing cache (for reference by users already on them) but must be marked `excluded (EOL YYYY-MM-DD)` in the Status column.

### Rule 3: Legacy models outside the 90-day zone

Legacy models with >90 days until EOL may appear in comparison tables **with annotation** (`Legacy — EOL YYYY-MM-DD`), but never as `recommended_model` or "Best Bedrock Match" when an Active alternative exists.

### Applying the rules

On each run, compute `days_to_eol = EOL date − today` for every model in the Legacy/EOL table below. Then:

1. `days_to_eol ≤ 0` → EOL. Remove from all tables.
2. `0 < days_to_eol ≤ 90` → **Exclusion zone.** Remove from recommendation/comparison tables. Mark `excluded` in pricing cache.
3. `days_to_eol > 90` and Legacy → Annotate, never recommend as primary.
4. Active → No restrictions.

---

## Legacy / EOL Models (as of 2026-09-17)

For models launched before 2026-09-07, check the
[legacy lifecycle table](https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle-legacy.html).
For newer models, check the model card and runtime `modelLifecycle.status`. The
table below captures pre-policy-change models referenced elsewhere in this
plugin. Recompute `days_to_eol = EOL date − today` on every run.

| Model              | Model ID                                  | EOL Date   | Days to EOL | Status       | Active Replacement      |
| ------------------ | ----------------------------------------- | ---------- | ----------- | ------------ | ----------------------- |
| Nova Canvas v1     | `amazon.nova-canvas-v1:0`                 | 2026-09-30 | 13          | **excluded** | Stability AI (see note) |
| Nova Reel v1       | `amazon.nova-reel-v1:0` / `v1:1`          | 2026-09-30 | 13          | **excluded** | —                       |
| Claude Sonnet 4    | `anthropic.claude-sonnet-4-20250514-v1:0` | 2026-10-14 | 27          | **excluded** | Claude Sonnet 5 / 4.6   |
| Jamba 1.5 Large    | `ai21.jamba-1-5-large-v1:0`               | 2026-11-26 | 70          | **excluded** | —                       |
| Jamba 1.5 Mini     | `ai21.jamba-1-5-mini-v1:0`                | 2026-11-26 | 70          | **excluded** | —                       |
| Marengo Embed v2.7 | `twelvelabs.marengo-embed-2-7-v1:0`       | 2026-11-30 | 74          | **excluded** | Marengo Embed 3.0       |
| Claude Opus 4.1    | `anthropic.claude-opus-4-1-20250805-v1:0` | 2027-01-08 | 113         | legacy       | Claude Opus 4.8 / 4.6   |

Jamba 1.5 Large / Mini and Marengo Embed v2.7 are in public extended
access; provider pricing may increase. They are now inside the 90-day
exclusion zone and must not appear in new-migration recommendation or
comparison tables.

**Removed (past EOL as of 2026-09-17):**

- Titan Image Generator v2 (`amazon.titan-image-generator-v2:0`) — EOL 2026-06-30
- Llama 3.2 all sizes (`meta.llama3-2-*-instruct-v1:0`) — EOL 2026-07-07
- Llama 3.1 405B Instruct (`meta.llama3-1-405b-instruct-v1:0`) — EOL 2026-07-07
- Claude 3 Sonnet (`anthropic.claude-3-sonnet-20240229-v1:0`) — EOL 2026-07-30
- Claude 3.5 Sonnet v1 (`anthropic.claude-3-5-sonnet-20240620-v1:0`) — EOL 2026-07-30
- Claude 3.5 Sonnet v2 (`anthropic.claude-3-5-sonnet-20241022-v2:0`) — EOL 2026-07-30
- Command R / R+ (`cohere.command-r-v1:0` / `cohere.command-r-plus-v1:0`) — EOL 2026-08-19
- Claude 3 Haiku (`anthropic.claude-3-haiku-20240307-v1:0`) — EOL 2026-09-10
- Nova Premier v1 (`amazon.nova-premier-v1:0`) — EOL 2026-09-14
- Nova Sonic v1 (`amazon.nova-sonic-v1:0`) — EOL 2026-09-14

> **AWS page lag:** As of 2026-09-17, the legacy lifecycle page still lists
> some rows whose published EOL date has passed. Treat the EOL date as
> authoritative. Keep those models only in this Removed list and catalog
> warnings; never recommend or invoke them.

**Status key:** `excluded` = ≤90 days to EOL, must not appear in any recommendation. `legacy` = >90 days to EOL, annotate but do not recommend as primary.

**⚠️ Image generation — Active successor is Stability AI:** Nova Canvas v1 is
Legacy and excluded from recommendations (EOL 2026-09-30). The Active image
generation models on Bedrock are **Stability AI** models:

| Model                      | Model ID                            | Pricing       | Tier     | Use case                              |
| -------------------------- | ----------------------------------- | ------------- | -------- | ------------------------------------- |
| Stable Image Ultra         | `stability.stable-image-ultra-v1:0` | ~$0.08/image  | premium  | Photorealistic, high-end visuals      |
| Stable Diffusion 3.5 Large | `stability.sd3-5-large-v1:0`        | ~$0.065/image | flagship | High volume creative assets           |
| Stable Image Core          | `stability.stable-image-core-v1:0`  | ~$0.04/image  | fast     | Rapid, affordable generation at scale |

When `image_generation` capability is detected:

1. Recommend **Stability AI** models as the primary Active target (not Nova Canvas).
2. Note the pricing model difference: Stability AI charges **per image**, not per token. Direct cost comparison with source provider (DALL-E, Imagen) requires converting to per-image equivalents.
3. If the user's source workload is DALL-E or Imagen, map to Stable Image Ultra (quality-first) or Stable Image Core (cost-first) based on `quality_vs_cost` preference in `preferences.json`.
4. Nova Canvas may appear as a Legacy fallback annotation but must not be `recommended_model`.

---

## Integration Points

### Design Phase (`design-ai.md`)

After selecting a Bedrock model for each workload:

1. Check the Legacy/EOL table above (or the lifecycle page).
2. If the model is in the **exclusion zone** (≤90 days to EOL) or EOL: reject it. Use the Active replacement.
3. If the model is Legacy but >90 days from EOL: replace with Active replacement if one exists. If no Active replacement exists, note the EOL date and recommend the user plan a follow-up migration.
4. If Active: proceed normally.

### Estimate Phase (`estimate-ai.md`)

When building the model comparison table:

- **Exclusion zone models**: omit entirely from `model_comparison`. Do not include in `recommended_model` or `backup_model`.
- **Legacy (>90 days)**: include with `(Legacy — EOL YYYY-MM-DD)` annotation. Never use as `recommended_model` if an Active alternative exists.
- **Active**: no restrictions.

### Pricing Cache (`pricing-cache.md`)

The multi-provider quick reference table includes a `Status` column:

| Status value                | Meaning                                                                                 |
| --------------------------- | --------------------------------------------------------------------------------------- |
| `active`                    | No restrictions                                                                         |
| `legacy (EOL YYYY-MM-DD)`   | Legacy, >90 days from EOL. Listed for reference, annotated.                             |
| `excluded (EOL YYYY-MM-DD)` | ≤90 days from EOL. Kept for existing users but must not be selected for new migrations. |

When refreshing the cache, recompute `days_to_eol` and update the Status column from the [model lifecycle page](https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html).

### Mapping Guides (`ai-openai-to-bedrock.md`, `ai-anthropic-to-bedrock.md`, and any source-cloud-specific guide the consuming skill ships)

- "Best Bedrock Match" columns must only contain Active models.
- Exclusion-zone models must not appear in any recommendation row.
- Legacy models (>90 days) may appear in notes or legacy-source mapping rows but never as the primary recommendation.

---

## Refresh Cadence

**On every design run:** Query `GetFoundationModel` or
`ListFoundationModels` for each candidate and inspect `modelLifecycle.status`.
`LEGACY` and `EOL` are never valid new-migration targets. For a model launched
on or after 2026-09-07, read its model card for the `EOL no sooner than` date
and whether its Legacy period is six months or 45 days. Absence from the
pre-2026-09-07 Legacy table does not prove that a newer model is Active.

When live/API evidence is unavailable, recompute
`days_to_eol = EOL date − today` for every pre-2026-09-07 row above and apply
the four selection rules. Treat an uncataloged newer model's lifecycle as
unverified; do not silently infer `active` from a stale pricing cache.

**Periodic table refresh:** Update this canonical file and the applicable
pricing caches together whenever AWS adds a Legacy/EOL date, a model card
changes, or a date passes. Then run `shared:sync`; never edit vendored copies
independently.

**Past-EOL rows:** Once `days_to_eol ≤ 0`, remove the model from the live
Legacy table on the next refresh and retain it only in Removed/catalog warnings
long enough for CI to catch stale targets.
