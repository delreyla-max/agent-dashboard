# Relevance Scoring Model & Algorithm Admin Panel

_Last updated: 2026-06-17. Lives in `dashboard.html`. Every constant below is tunable
live from the in-app admin panel and persists in the Supabase `scoring_config` table._

## Principle

The board is **relevance-ranked, recency-aware** — not date-sorted. A perfect-fit job
from three weeks ago should outrank a fresh-but-weak one. The old behavior sorted by
posting date and only used score as a tiebreak, so relevance never actually reordered the
list. Now a single `RelevanceScore` (0–100) is the sort key, and posting date only breaks
ties. We would rather show *nothing* than noise, so a floor hides weak matches.

## The formula

```
RelevanceScore = clamp( CoreFit × respMult × warmMult × locMult + freshnessBonus , 0, 100 )

CoreFit = weighted blend of normalized résumé / pref / tier,
          with the weights renormalized over only the signals a job actually has.
```

### Layer 1 — CoreFit (base 0–100)

Raw pipeline scores are compressed (résumé tops out ~70–75, pref ~40, often 0), so each
component is first **normalized to a true 0–100** against a tunable calibration max, then
blended 40 / 40 / 20.

| Component | Field | Normalization | Weight |
|---|---|---|---|
| Preference fit | `prefScore` | `min(100, prefScore / pref_max · 100)`, `pref_max = 40` | 0.40 |
| Résumé match | `score` | `min(100, score / resume_max · 100)`, `resume_max = 75` | 0.40 |
| Tier (quality) | `tier` | Tier 1 = 100, Tier 2 = 60, Tier 3 = 30, none = 0 | 0.20 |

**Graceful re-weighting:** a component only participates if it has signal (`prefScore`
counts only when `> 0`). The weights are renormalized over whatever is present, so a user
with no preferences set yet (e.g. an empty `prefScore`) is scored on résumé + tier alone
(0.67 / 0.33) instead of being dragged toward zero by an absent signal.

Calibration is **absolute, not relative** — deliberately not scaled against each user's own
top job, so a genuinely weak week genuinely shows fewer jobs.

### Layer 2 — Candidate-strength multipliers (scale CoreFit)

| Multiplier | Source | Range | Notes |
|---|---|---|---|
| `respMult` | years-required parsed from the posting vs `his_years` (15) | ×0.85 – ×1.15 | strong applicant ↑, stretch ↓; neutral 1.0 if no years parsed |
| `warmMult` | company vs `warm_companies` list | ×1.20 (or ×1.0) | boost only; a connection can't manufacture relevance |
| `locMult` | `isLocalOrRemote()` heuristic | ×1.0 local/remote, ×0.80 out-of-area | **soft**, replaces the old hard LA filter |

Multipliers swing a job by at most ~±20 %, so they refine the order but can't rescue a bad
match.

### Layer 3 — Freshness (additive nudge, decays)

`freshnessBonus = fresh_bonus · max(0, 1 − daysOld / fresh_window_days)` → **+6 → 0 linearly
over 14 days**. "Discovered in the latest run" (via `_latestIds`) counts as day 0; otherwise
it decays from the `posted` date. Capped at +6 absolute, so recency can only break near-ties
— it never lifts a weak job over a strong one.

### Layer 4 — Display gate (no effect on order)

- **Floor:** the Outlook slider / `floor` (default **60**) hides anything below it.
- **Match level** (label only): 90–100 Excellent · 80–89 Strong · 70–79 Good ·
  60–69 Possible · below 60 hidden.
- **Max age:** postings older than `max_age_days` (14) are dropped outright.

## Signals feeding preference learning

- **Save (★)** — strong positive: adds the full set of the job's title words to
  `learned_keywords` and moves the job off the board.
- **Click (job title link)** — weak positive: logged to `job_clicks`, and adds at most **2**
  of the title's words to learning (a save adds its full set, so a click counts for less).
- **Thumbs-down (👎)** — negative: removes the title's words and hides the job.

## Verified behavior (real data, 2026-06-17)

At floor 60 with location soft (hard LA filter off by default): **Garman 5 visible**
(1 Excellent, 3 Good, 1 Possible — Apex Legends "Lead AI Designer" tops at ~92),
**Chris 3 visible**. Dropping the floor to 50 → Garman 10 / Chris 3.

## Storage

`scoring_config` (single global row, id = 1, `config` JSONB) — readable by all authenticated
users so both boards run the same tuned algorithm; **writable only by the admin** account
(`gyipla@gmail.com`) via RLS. `job_clicks` (per-user, RLS-owned) logs clicks.

## Admin panel

A gear button (bottom-right, **admin only**) opens a panel of number inputs for every
constant above plus the warm-companies list. Editing any field **re-ranks the visible board
instantly** (the math runs client-side). **Save** upserts to `scoring_config` for everyone;
**Reset defaults** restores the built-in values.

## Known limitations / follow-ups

- `scoring_config` is **global**, but a few constants are personal (`his_years`,
  `warm_companies`). They currently apply to both users using the admin's values. Per-user
  overrides are a future enhancement. `warmMult` only ever boosts, so a mismatched list for
  the non-admin user is low-harm.
- `respMult` uses years-required only; `seniorityLevel` is too sparse in the data to use yet.
- Salary fit is not yet folded into `locMult` (string salary parsing deferred).
- The **14-day cap is enforced on the board**; for a true effect the daily search SKILL's
  lookback should also be set to 14 days (separate pipeline change).
