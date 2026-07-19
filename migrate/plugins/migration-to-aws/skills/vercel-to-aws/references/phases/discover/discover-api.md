---
_fragment: vercel-api
_of_phase: discover
_contributes:
  - discovery.json (deployments, env_var_names, domains, storage_integrations, usage_metrics, peripherals, api_routes, backend_service_detected sections)
---

# Discover Phase: Vercel API Capture Parsing (Always Runs)

> Self-contained PARSE-ONLY fragment. Reads the API captures that
> `discover-capture.md` wrote to `$MIGRATION_DIR/capture/api/` (indexed by
> `capture/manifest.json`) — this fragment runs in the dispatched worker, which
> has no network access and never sees the token. It parses everything the
> capture exposes: projects, deployments, env var NAMES (never values), domains,
> cron jobs, Edge Config, KV/Postgres/Blob store integrations, FOCUS billing
> charges / `vercel usage` CLI spend (when captured), and residual coarse usage
> metrics. Process only manifest entries with `status: "ok"`; carry every
> `failed`/`skipped` entry into the corresponding section as
> `"unavailable: <note>"`.

**Execute ALL steps in order. Do not skip or optimize.**

---

## Step 1: Parse Deployments

From `capture/api/deployments-<project>.json` for each in-scope project
(`tier1-signals.json.project_list`): recent deployments, production vs. preview,
timestamps, status.

---

## Step 2: Parse Env Var Names (Never Values)

Read `capture/api/env-keys-<project>.json` — already reduced to KEY names by the
capture step's projection rule. Per Requirement 1.6, values are NEVER fetched,
logged, or persisted anywhere in the pipeline; if this file unexpectedly
contains objects with value-like payloads instead of a plain key-name array,
DISCARD it, record a warning, and continue — never copy any part of it. (The
"infrastructure-pointing env var hostnames" Tier 3 input remains scoped to
hostnames the FOUNDER explicitly shares out-of-band.)

---

## Step 3: Parse Domains and Crons

From `capture/api/domains-<project>.json` and `capture/api/crons-<project>.json`:
custom domains and Cron jobs (supplementing what `vercel.json` already declared
in `discover-configs.md` — the API capture may show crons configured outside
`vercel.json` too).

---

## Step 4: Parse Storage Integrations

From `capture/api/stores.json`: Vercel KV, Postgres, Blob, and Edge Config
integrations attached to the project(s). For each, record the integration type
and any AWS-relevant metadata (e.g. approximate size, region) the capture
exposes.

Contribute these to `peripherals[]` — this feeds the Recommendation Engine's
separability check (Requirement 7.1 rule 1) and the Scaffold phase's
peripheral-mapping lookup (`knowledge/peripheral-mappings.json`).

---

## Step 5: Detect a Separate Backend/API Service

Inspect the project structure and API route enumeration for signs of API routes
that could stand alone as a separable backend (e.g. a `pages/api/` or `app/api/`
tree substantial enough to be its own service, or an explicitly separate Vercel
project already serving as a backend API). Contribute `api_routes[]` and a
`backend_service_detected: true|false` flag — both feed the Recommendation
Engine's separability check directly.

---

## Step 6: Billing + Usage Metrics (spend baseline for Estimate)

Build `usage_metrics` from the best available spend capture (first match wins),
then attach any residual coarse aggregates.

### 6a. FOCUS billing charges (preferred)

When the manifest marks `billing-charges.jsonl` (or
`billing-charges-summary.json`) as `ok`:

1. Read FOCUS v1.3 charge lines (JSONL) or the pre-aggregated summary.
2. Prefer `EffectiveCost` for totals (fall back to `BilledCost` when
   `EffectiveCost` is absent). Currency is USD per the API.
3. If multiple projects are in `tier1-signals.json.project_list`, filter charge
   rows whose `Tags.ProjectId` / `Tags.ProjectName` match an in-scope project
   when those Tags are present; if Tags are missing on a row, keep it in the
   **team** total and note `project_filter: "partial_tags"`.
4. Emit `usage_metrics.billing_data`:

```json
{
  "source": "focus_billing_charges",
  "window_days": 30,
  "currency": "USD",
  "monthly_total": 123.45,
  "by_service": { "Fluid Compute": 80.0, "Fast Data Transfer": 20.0 },
  "by_project": { "prj_acmeshop01": 100.0 },
  "charge_count": 42,
  "contract_commitments_present": false
}
```

Normalize the 30-day window total to a monthly figure as
`monthly_total = sum(EffectiveCost) * (30.437 / window_days)` when the captured
window is not ~30 days; for the default 30-day capture, use the sum directly.
Set `usage_metrics.confidence` to `HIGH` when `billing_data` is present from
FOCUS (line-item triangulation). Soft-read
`contract-commitments.json` when `ok` — set
`contract_commitments_present: true` only if the file contains at least one
commitment; never invent amounts from commitments alone.

### 6b. `vercel usage` CLI fallback

When 6a is unavailable and `usage-cli.json` is `ok`, parse
`totals` / `services[]` (`Effective Cost` / `Billed Cost` fields as emitted by
`--format json`). Emit the same `billing_data` shape with
`"source": "vercel_usage_cli"` and `confidence: "MEDIUM"` (service rollups, not
FOCUS line items).

### 6c. Coarse residuals (never a spend baseline alone)

Legacy / optional `usage-<project>.json` aggregates (invocation counts,
bandwidth) may still appear; parse them into `usage_metrics.invocation_counts`
etc. Per Requirement 4.4, a finding resting SOLELY on this coarse usage data
(with no log-drain and no `billing_data`) is LOW confidence — record it as
such, with `upgrade_input: "7-14 day log drain/observability export"`. Coarse
residuals must **not** fabricate `billing_data`.

When no billing capture succeeded, omit `billing_data` entirely (Clarify Q6
asks for spend; Estimate falls through). Do not invent zeros.

---

## Step 7: Output Contribution for Parent Orchestrator

```json
{
  "deployments": [...],
  "env_var_names": ["DATABASE_URL", "REDIS_URL", ...],
  "domains": ["<hostname>", ...],
  "crons": [...],
  "storage_integrations": [{ "type": "kv" | "postgres" | "blob" | "edge_config", "metadata": {...} }],
  "peripherals": [{ "type": "...", "source": "vercel_api" }, ...],
  "api_routes": ["<route>", ...],
  "backend_service_detected": false,
  "usage_metrics": {
    "confidence": "HIGH" | "MEDIUM" | "LOW",
    "billing_data": {
      "source": "focus_billing_charges" | "vercel_usage_cli",
      "window_days": 30,
      "currency": "USD",
      "monthly_total": 0,
      "by_service": {},
      "by_project": {},
      "charge_count": 0
    },
    "invocation_counts": {}
  }
}
```

Findings sourced purely from the Vercel API (no log drain) carry
`computed_from_inputs: ["vercel_api_token"]`. Usage-metric findings that lack
`billing_data` additionally list `"log_drain_export"` in `computed_from_inputs`
even though log drain data was NOT used this run — this is intentional: it means
a future re-invocation with a log drain export newly present will correctly
trigger recomputation of this specific finding (per the recompute short-circuit
mechanism in `discover-coupling.md`/`discover-preflight.md`, and analogously
here). When `billing_data.source` is `focus_billing_charges`, also list
`"vercel_billing_charges"` in `computed_from_inputs`.

---

## Error Handling

| Error Category                                                            | Behavior                                                                          |
| ------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Capture file named in the manifest is missing or malformed                | Record a warning, mark that section `"unavailable: capture_unreadable"`, continue |
| Manifest records `failed: rate_limited` / `insufficient_token_scope` etc. | Mark the corresponding section `"unavailable: <note>"`, continue with the rest    |
| Zero deployments found for an in-scope project                            | Record as a LOW-confidence finding, note the project may be dormant/unused        |

---

## Scope Boundary

**This fragment covers PARSING the API captures ONLY.**

FORBIDDEN — Do NOT include ANY of:

- Any network call or `curl` — capture already happened in the main window
  (`discover-capture.md`); this worker has no network and no token
- Fetching or persisting env var VALUES (names only, ever)
- Header-probe parsing (that is `discover-probe.md`'s job)
- Computing the Recommendation Engine's separability verdict (this fragment
  contributes the raw signals; `recommend-rules.md` makes the decision)
- AWS service names or recommendations

**Your ONLY job: enumerate the Vercel API surface and record raw facts. Nothing
else.**
