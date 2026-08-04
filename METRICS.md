# Metric Contract — ct-demand-marketing dashboard

**What this is:** the authoritative definition of every number this dashboard shows, every number it
*could* show on request, and — just as importantly — the questions that **cannot** be answered from
current data sources.

**Who it's for:** both humans and AI agents. If you are an agent answering a data question about this
dashboard, read this file first and follow [How to answer a question](#how-to-answer-a-question).
Do not query the warehouse without checking [Traps](#traps--read-before-trusting-any-number) — several
tables will hand you a confident, wrong number.

**Status:** reflects the `exec-summary-mtm-source` preview branch on the public
`kiethuynhphu-ct/ct-demand-marketing` repo. Verified against live BigQuery 2026-08-04. Re-verify
freshness before quoting anything as current.

> ⚠️ **Automation is currently broken.** The daily GitHub Actions refresh (§E) does not run — the
> `GCP_USER_CREDENTIALS` secret is missing from the repo entirely (confirmed 2026-08-04). Treat the
> page's `DATA_AS_OF` header as a manual snapshot date, not a guarantee of daily freshness, until this
> is fixed.

> **Scope note:** this file names datasets and tables only. It deliberately contains no spend figures,
> no spreadsheet IDs, and no individual owners' names — this repository is public. Those live in the
> maintainer's internal notes; see [Escalation](#escalation).

---

## How to answer a question

Route every question through these four cases **in order**. Never skip to SQL.

| # | Case | What to do |
|---|---|---|
| 1 | The number **is on the dashboard** | Quote it and name the card. Don't re-derive it — re-deriving is how two "official" numbers start disagreeing. |
| 2 | **Queryable but not shown** (§B) | Write the query against the table named in §B. Apply the stated grain and aggregation exactly. |
| 3 | **Cannot be answered** (§C) | Say so plainly. Explain the missing dimension. Do **not** substitute a proxy and present it as the answer. |
| 4 | **Out of scope** | Route to the right tool — see [Escalation](#escalation). |

**Every answer must carry:** the source table · the filters applied · the date range · data freshness ·
any applicable trap from §D. A number without provenance is not an answer.

**Aggregation convention** (applies everywhere unless a row says otherwise):
`dau` and `dau_w_lead` are **averaged** over the month; `lead` is **summed** over the month.
Getting this backwards is the single most common way to produce a wrong number here.

---

## §A — Metrics shown on the dashboard

Primary source for the Executive Summary on this branch:
`chotot-dwh.ct_digital.mtm_chotot_vertical_dau_mau`, filtered `vertical='Chotot'` (the full-platform
aggregate row), summed across platform. Fresh through 2026-07-31.

| Metric | Definition | Source | Notes |
|---|---|---|---|
| DAU | Daily active users, averaged over the month | `mtm_chotot_vertical_dau_mau` · `dau` | Verified exact match vs. warehouse for July MTD |
| DAU w/ Lead | Daily actives who generated ≥1 lead, month-averaged | `mtm_chotot_vertical_dau_mau` · `dau_w_lead` | ~0.12% variance vs. the legacy monthly table |
| Total Lead | Lead events, summed over the month | `mtm_chotot_vertical_dau_mau` · `lead_daily` | ⚠️ **Apr/May disputed — see T-1** |
| MAU | Monthly active users | *hardcoded on this branch* | Not yet re-sourced; carried over from the legacy dashboard |
| MAU w/ Lead | Monthly actives with ≥1 lead | *hardcoded on this branch* | Same as above |
| DAU/MAU | Stickiness ratio | derived (DAU ÷ MAU) | Not stored; computed on read |
| New vs Returning | New/returning split for DAU and DAU w/ Lead | `mtm_chotot_vertical_dau_mau` · `new_dau`/`return_dau`, `new_dau_w_lead`/`return_dau_w_lead` | Only these two metrics have the split. Lead and MAU do **not**. |
| Contribution % | Each side's share of the New+Returning total | derived | Rendered as its own card |
| By Channel | DAU / DAU w/ Lead / Lead split across Direct, SEO, Digital, Growth, Others | `chotot-dwh.ct_digital.mtm_chotot_vertical_channel` | ⚠️ **See T-3 (overcounting)** |
| By Vertical | Same metric set, split PTY / JOB / VEH / GDS | `mtm_chotot_vertical_dau_mau`, `vertical IN (pty,jobs,veh,gds)` | **Switched 2026-08-04** from the legacy `chotot_mtm.dashboard__*` tables. PTY/JOB/GDS agree with the old source within ~1-2%. ⚠️ **VEH is not reconciled** — Total Lead is +93% in Apr and +71% in May vs. the old source (see T-1) |
| FC0 / FC1 / FC2 | Three forecast vintages vs. actuals | Marketing plan spreadsheet (manual) | FC0 = annual plan (all 12 mo) · FC1 = post-Q1 (Apr–Dec) · FC2 = post-Q2 (Jul–Dec). Months outside a vintage's window render "—", never a fabricated number. |

**Channel mapping.** Raw channel values are `digital`, `seo`, `direct`, `growth_inapp`,
`growth_outapp`, `others`. `growth_inapp` **and** `growth_outapp` both fold into a single **Growth**
bucket. Preserve this mapping or your channel numbers won't reconcile with the dashboard.

**Vertical normalization.** Verticals appear as text codes (`pty`, `jobs`, `veh`, `gds`) or numeric
category codes: 1000–1050 → PTY · 2000–2080 → VEH · 13000/13010 → JOB · everything else → GDS.

---

## §B — Available but not shown (this is where most requests land)

These are **not** on the dashboard but **can** be answered from the warehouse today.

| You want | Source | Grain | Caveats |
|---|---|---|---|
| Ad spend, impressions, clicks | `chotot-dwh.ct_digital.kiet_digital_campaign_daily` | ad platform (Google/Meta) × campaign × account | No vertical/channel **column** — but see [Spend by vertical](#spend-by-vertical) below |
| **Spend by vertical** | same table, campaign-name parsing | vertical (derived) | ⚠️ **Convention-based, not modeled — read the method below**. Deferred — no UI for this exists on the dashboard yet; if built, ELT rolls into GDS (decided 2026-08-04) and the unattributed remainder must always render explicitly, never hidden |
| Platform split (iOS / Android / MSite / Desktop) | `chotot-dwh.ct_digital.mtm_chotot_vertical_dau_mau` | vertical × platform | Available for every metric in that table. Note: this table's "platform" is **device** (Android/iOS/Desktop/MSite), not acquisition channel — it has no channel column at all, so it cannot cover channel-grain breakdowns (see the By Channel row in §A) |
| Campaign → marketplace attribution | `chotot-dwh.ct_digital.mtm_chotot_vertical_channel_campaign_dau_mau` | vertical × channel × campaign, daily | The only source joining campaigns to vertical/channel outcomes |
| Sub-vertical detail (C2C / ELT / category codes) | `chotot-dwh.ct_digital.mtm_chotot_vertical_dau_mau` | sub-codes `2010`/`2020`/`5010`, `C2C`, `ELT` | Present in the table; not surfaced in any UI |
| Daily (not monthly) figures | most tables above are daily-grain | daily | The dashboard aggregates to months; the underlying data is daily |

### Spend by vertical

`kiet_digital_campaign_daily` has **no vertical column**, but campaign names follow a convention that
embeds the vertical as an underscore-delimited token — `pty`, `job`/`jobs`, `veh`, `gds`, `elt`
(e.g. `digital_dau_pty_sell_vn_pmax_adview_2026`, `brand_job_eng_b2c_0726`). Parsing that token
attributes **~97% of spend**.

Measured coverage, July 2026 (verified 2026-08-01):

| Bucket | Campaigns | Share of spend |
|---|---|---|
| PTY | 337 | 42.9% |
| JOB | 367 | 22.6% |
| VEH | 141 | 14.4% |
| GDS | 96 | 12.1% |
| ELT | 48 | 5.0% |
| Ambiguous (>1 vertical token) | 2 | 1.9% |
| Unclassified (no token) | 785 | 1.1% |

```sql
REGEXP_EXTRACT(campaign, r'(?:^|_)(pty|job|jobs|veh|gds|elt)(?:_|$)')
```

**Read these before quoting a per-vertical cost number:**

- **This is a naming convention, not modeled attribution.** Nothing enforces it. If marketing renames
  campaigns or launches one without the token, spend silently drops into *Unclassified* — the number
  gets quietly wrong rather than visibly broken. Re-check coverage whenever you use this.
- **Always report the unattributed remainder** (~3%) alongside the split. Never present the four
  verticals as if they sum to total spend.
- **ELT is reported separately, not folded into GDS.** The dashboard's vertical set is PTY/JOB/VEH/GDS,
  but the warehouse also carries `ELT` and `C2C` as distinct codes. Whether ELT rolls up into GDS for
  cost reporting is an **open question — confirm with the maintainer** rather than assuming.
- **2 campaigns carry more than one vertical token** and are genuinely ambiguous; they need manual
  classification.
- **Deriving CPA/CPL per vertical compounds two error sources** — this ~3% attribution gap *plus*
  trap T-4 (vertical figures don't reconcile to the platform total). Treat any resulting
  cost-per-lead as directional, not exact.

---

## §C — Cannot be answered today

**Do not improvise around these.** Each is a genuine data gap, not a query-writing problem. Answering
anyway requires inventing an attribution that does not exist in the data.

| Question | Why not | What would be needed |
|---|---|---|
| **Cost / spend per channel** | Spend exists only in `kiet_digital_campaign_daily`, which has **no channel dimension**. Unlike vertical, campaign names carry no reliable channel token. | Channel attribution added upstream, or a campaign→channel mapping |
| CPA / cost-per-lead **per channel** | Numerator (spend) can't be sliced by channel at all | Resolve the channel-attribution gap first |
| **MAU per channel** | No per-channel MAU source exists anywhere. The production dashboard *estimates* it as (channel's share of vertical DAU) × (vertical MAU) — a **proxy, not measured data**. | A real per-channel monthly-unique source |
| New vs Returning for **Lead or MAU** | The split exists only for `dau` and `dau_w_lead` | New/return columns for those measures upstream |
| **Total Lead for Apr or May 2026** (platform total, and VEH specifically) | Two sources disagree — 17-18% at platform grain, +93%/+71% for VEH — and the discrepancy is unreconciled — see T-1 | Owner reconciliation (tracked as O-014) |

If asked one of these, say the number cannot be produced and explain the missing dimension. An honest
"we can't compute that yet, here's why" is a correct answer. A plausible fabricated one is not.

---

## §D — Traps · read before trusting any number

| ID | Trap | Consequence |
|---|---|---|
| **T-1** | `mtm_chotot_vertical_dau_mau` reports platform-total Total Lead **17–18% higher than the legacy source for April and May 2026 specifically**. At **per-vertical** grain (switched onto this table 2026-08-04) the same gap **localizes to VEH**: +93% in Apr (1,481,312 vs 766,108), +71% in May (1,228,473 vs 719,097) — every other vertical/month agrees within ~2%. Two different table owners; unreconciled (**O-014**). | Never quote VEH or platform Apr/May Lead without stating both figures and the dispute. |
| **T-2** | `chotot-dwh.ct_product_analytics.daumaulead_mkt_rp` was **9 days stale** as of 2026-08-01 (max date 2026-07-22). It still feeds production's Detail Table and Campaign Detail. | Recent-period queries against it silently omit the newest ~9 days. Prefer `mtm_chotot_vertical_channel`. |
| **T-3** | Channel data **double-counts** users active in 2+ channels in a period. Production adds an "Unattributed" row (authoritative total − sum of channels), which can legitimately go **negative**. ~1.3–1.5% DWL overcount observed. | Channel rows will not sum to the vertical total. This is by design, not a bug — don't "fix" it. |
| **T-4** | Platform DAU runs a consistent **~18–20%/day higher** than PTY+JOB+VEH+GDS summed. | Vertical figures will not reconcile to the platform total. Unexplained (**O-015**) — flag it, don't paper over it. |
| **T-5** | MAU / MAU w/ Lead / DAU-MAU are **hardcoded** on the preview branch, not live-sourced. | They will not move when the warehouse does. |
| **T-6** | Two-tier authority in production: totals come from the `chotot_mtm` tables; the channel table supplies **breakdown shape only, never totals**. | Don't derive a total by summing channels. |
| **T-7** | Forecast vintages cover different windows (FC0 all year · FC1 Apr–Dec · FC2 Jul–Dec). | Comparing an actual to the wrong vintage produces a meaningless variance. |

---

## §E — Freshness

| Source | Fresh through (checked 2026-08-01) |
|---|---|
| `mtm_chotot_vertical_dau_mau` | 2026-07-31 |
| `mtm_chotot_vertical_channel` | 2026-07-31 |
| `kiet_digital_campaign_daily` | 2026-07-31 |
| `daumaulead_mkt_rp` | 2026-07-22 ⚠️ stale (T-2) |
| Forecast spreadsheets | manual — updated ad hoc |

The dashboard is *supposed* to refresh daily at 11:00 GMT+7 via GitHub Actions, but as of 2026-08-04
this does **not** happen — the `GCP_USER_CREDENTIALS` repo secret is missing entirely, so every run
fails before it can query BigQuery. The `DATA_AS_OF` header on the page is currently a manual
snapshot date, not evidence of a working daily refresh — check it critically rather than trusting it
by default until this is fixed.

---

## Escalation

- **Numbers, definitions, or a gap in §C** → the dashboard maintainer. Sheet IDs, table owners, and
  spend detail are withheld from this public file and held in internal notes.
- **Marketing budget / spend questions** → the marketing budget tooling, not this dashboard.
- **Ads platform performance** (CTR, CPI, creative) → the ads-performance tooling.
- **A number you need repeatedly** → say so. Recurring questions are how this dashboard decides what
  to build next; a question asked three times should become a card.

---

## Maintaining this file

This contract is only useful while it's true. Update it in the **same commit** as any change to what
the dashboard renders or where it sources from. If you add a metric, add its row to §A. If you close a
gap, move it out of §C. If a trap is resolved, delete it and note the resolution.
