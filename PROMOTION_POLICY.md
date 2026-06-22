# Model promotion policy

How a new LLM gets from "flagged on the radar" to "in the recommender". Designed
to surface meaningful releases within 24 hours (Ryan-from-Intelligent-Noise test
case: GLM-5.2 dropped at noon, dashboard has comparative data by tomorrow's daily
refresh).

There are three tiers. `build.py` handles tier promotion automatically; a human
maintainer handles tier-1 → tier-2 graduation by moving an entry into
`aliases.json`.

---

## Tier 0 — Radar

**Status:** Detected upstream, not in the dashboard.

**Visibility:** Appears in the "N new model(s) flagged" popover at the top of the
page. Not in the scatter, table, or recommender.

**How it gets here:** `new_model_radar()` in `build.py` flags any Artificial
Analysis model that:
- has `intelligence_index ≥ 35`
- comes from a creator in `TRACKED_CREATORS`
- has a base name not already in `aliases.json`
- is newer than our newest tracked model from that creator (so back-catalog
  doesn't churn the radar)

This step runs every daily refresh.

---

## Tier 1 — Preview *(auto-promoted)*

**Status:** Showing as `staging: true` in `models.json`. Appears in the scatter
chart, Full Comparison table, and the radar popover — but **NOT in the
Routing recommender top-3**, because thin upstream data shouldn't drive routing
decisions.

**Badge:** Each preview model carries a `PREVIEW` chip in the UI so users see
which numbers are provisional.

**Promotion gates (all must hold):**
1. Listed in Artificial Analysis with a non-null `intelligence_index`
2. Creator has a `providers.json` entry (so train policy and pricing tier are
   inheritable)
3. Canonical not in `radar_blocklist.json` (maintainer's veto list)

**Time from upstream visibility:** 0 — once a model meets the gates, it
auto-appears in the next daily refresh. No staying-power requirement, because
the use case is "ship Ryan something before the Twitter cycle moves on."

**Failure mode:** If a model auto-promotes with bad data and the maintainer
notices, they add its canonical to `radar_blocklist.json` and the next refresh
strips it. This is the safety valve.

---

## Tier 2 — Tracked *(maintainer-graduated)*

**Status:** First-class entry in `aliases.json`. Counts toward the 21+ tracked
models. Appears in the recommender top-3.

**Graduation criteria (human review):**
- Model has been Tier 1 for ≥7 days OR has data in 3+ upstream sources
  (AA + OpenRouter pricing + LMArena rank)
- Maintainer has eyeballed the data and trusts the name-string mappings
- Provider entry exists with confidence (train policy known, multilingual tier
  set, fine-tuning policy filled in)

**How to graduate:**
1. Open `aliases.json`
2. Copy the Tier 1 entry's `canonical` + upstream name strings from
   `radar_auto.json` (machine-generated suggestions)
3. Verify and adjust each name string against the actual upstream catalog
4. Confirm `providers.json` has a clean entry for the creator
5. Commit. Next daily refresh drops the model from `radar_auto.json` and
   processes it as a normal tracked model.

---

## Review cadence

**Daily (automatic):** `build.py` re-derives radar + Tier 1 list.

**Weekly (maintainer):** Scan the preview models. For each:
- Looks healthy and has full data → graduate to Tier 2
- Looks like noise or a back-catalog model → add to `radar_blocklist.json`
- Looks healthy but data is still thin → leave at Tier 1 another week

**Monthly (maintainer):** Audit `providers.json` for any creators that have
appeared in Tier 1 but don't yet have a clean entry. Fill in missing fields.

**Ad-hoc (anyone):** If a stakeholder asks "where's model X?" — check radar,
check blocklist, check whether creator is in `TRACKED_CREATORS`. Surface the
gap.

---

## Files

| File | Owner | Purpose |
|---|---|---|
| `aliases.json` | human | Tier 2 — the curated list |
| `providers.json` | human | Train policy, pricing tier, multilingual support per provider |
| `radar_blocklist.json` | human | Canonicals to NEVER auto-promote (typos, duplicates, abandoned releases) |
| `radar_auto.json` | machine | Generated each refresh — Tier 1 promotion suggestions |
| `data/radar.json` | machine | Generated each refresh — Tier 0 radar list |

---

## Honesty about confidence

The dashboard labels every preview model so users don't confuse provisional
data with verified data. The point is to surface comparison *quickly* without
pretending we have full coverage. If you ship a screenshot to a stakeholder
and a preview model is the answer, mention it's preview.

---

## How to add a new closed-weights model (Claude, GPT, Gemini, Grok)

These models are the only manual case. Anthropic and OpenAI don't publish to
Hugging Face, Epoch tracks them only after benchmarking, and AA usually takes
1–3 weeks to score a frontier closed model. Until AA picks it up, you have to
hand-add a stub. Sixty seconds of work.

### 60-second flow

1. **See the announcement.** Anthropic blog, OpenAI / xAI / Google announcement,
   X post, etc.
2. **Open `aliases.json`**, copy the closest existing entry from the same
   creator (e.g. copy `claude-opus-4-8` when adding `claude-opus-4-9`).
3. **Update these fields**:
   - `canonical` — kebab-case id, e.g. `claude-opus-4-9`
   - `display` — what shows in the UI, e.g. `Claude Opus 4.9`
   - `aa_name` — the EXACT string AA will use once they score it
     (best guess; the maintainer fixes it later if wrong)
   - `epoch_name`, `arena_name`, `openrouter_slug`, `salesevals_name` — same
     pattern, all best-guess strings
   - `provider_key` — almost always the same as the previous-generation entry
     (e.g. `anthropic-opus`), unless pricing changed
   - **`release_date_override`** — ISO date `YYYY-MM-DD`. This makes the model
     appear in "Recently fielded" even before AA scores it.
   - `_note` — one line for future-you: "AA hadn't scored this yet on
     2026-MM-DD, remove override when intelligence_index populates"
4. **Update `providers.json`** ONLY if pricing changed from the previous
   generation. Otherwise skip.
5. **`python3 build.py`** to confirm no errors. The new model will appear in
   the next daily GitHub Action refresh.

### Stub template

```json
{
  "canonical":           "claude-opus-4-9",
  "display":             "Claude Opus 4.9",
  "aa_name":             "Claude Opus 4.9",
  "epoch_name":          "Claude Opus 4.9",
  "arena_name":          "claude-opus-4-9",
  "openrouter_slug":     "anthropic/claude-opus-4.9",
  "vectara_name":        null,
  "salesevals_name":     "Claude Opus 4.9",
  "provider_key":        "anthropic-opus-4.8",
  "_note":               "AA pending. Released 2026-MM-DD per Anthropic blog. Remove override when scored.",
  "release_date_override": "2026-MM-DD"
}
```

### Auto-graduation

Once AA publishes a score for that exact `aa_name`, `merge_one()` overwrites
the stub's `release_date` with AA's real value, fills in `intelligence_index`,
`task_scores`, `cost.aa_blended_per_1m`, etc. The `preliminary` tag drops off.
At that point the maintainer can delete `release_date_override` and `_note` —
the entry becomes a normal tracked model.

### When a stub goes stale

If a stub sits for >45 days without AA picking it up:
- AA name guess was probably wrong. Open
  [artificialanalysis.ai/models](https://artificialanalysis.ai/models)
  and find the exact name, update `aa_name`.
- Or the model was renamed / cancelled. Remove the stub.
