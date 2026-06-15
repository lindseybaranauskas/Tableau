# Command View — Design Spec & Executive Sign-Off Questions

**Purpose:** This document captures how the consolidated sales dashboard currently behaves and the decisions we need executives to confirm before it goes live. Each section states *what the prototype does today* (our working assumption) and the *open question* for the execs. The goal is to replace guesses with explicit sign-off so the live version shows exactly what leadership wants.

How to use this in the walkthrough: for each "❓ Decision needed," capture the chosen option and any nuance. Items already agreed are marked "✔ Assumed — confirm."

---

## 1. Time periods & the comparison basis (most important)

The dashboard has a global period control: **Year, Quarter, Month, YTD, Trailing 12 Months (T12M)**. Every metric, widget, and comparison reacts to this control.

**How each period defines "current" and "prior" today:**

| Period | Current window | Compared against (prior) |
|---|---|---|
| Year | Full calendar year (e.g. all of 2026) | Full prior year (all of 2025) |
| Quarter | Selected quarter | Same quarter, prior year |
| Month | Selected month | Same month, prior year |
| YTD | Jan 1 → today | Jan 1 → **the same date** last year |
| T12M | Last 12 months | The 12 months before that |

**❓ Decision needed — the headline comparison question you raised:** For the default headline numbers (e.g. JSC Won, Amount Won), should the "vs last year" comparison be:
- (a) **Full year vs full year** (current "Year" behavior), or
- (b) **Same-point-in-time** — this year-to-date vs last year through the same date (the "YTD" behavior)?

Right now both are available via the toggle, but we should pick the **default** the dashboard opens on, and which comparison execs consider the "real" one for board reporting. Our recommendation: default to **YTD same-point** for fairness, with Year available for full-year retrospectives.

**❓ Decision needed:** Is the fiscal year the calendar year, or does the company use a non-calendar fiscal year? (The prototype assumes calendar year.)

**❓ Decision needed:** For Quarter/Month, is the comparison "same period last year" (current) or "vs the immediately prior quarter/month"? Both are reasonable; execs should pick one.

---

## 2. KPI scorecards — definitions & comparisons

Five cards across the top. For each: what it measures, and how the comparison is computed today.

| KPI | Current definition | Comparison shown |
|---|---|---|
| **Amount Won** | Sum of `Amount` for won opportunities in the period | **% of target** (pace badge), not YoY |
| **Revenue Expected** | Sum of expected revenue for won opps | YoY (vs prior period per the table above) |
| **Win Rate** | Won count ÷ (Won + Lost count) | YoY |
| **Weighted / Open Pipeline** | Open opps; weighted = `Amount × probability`, open = raw `Amount` | YoY |
| **Pipeline Health** | Weighted open ÷ (a coverage goal) shown as a ratio (e.g. 1.2×) | Health label (Healthy / Watch / At risk) |

**❓ Decisions needed:**
1. **Win rate basis:** by **count of deals** (current) or by **dollar value** won vs lost? These can tell very different stories.
2. **Amount Won card:** should it show **% of target** (current) or **YoY %**, or both?
3. **Pipeline Health "coverage goal":** today it's a placeholder (40% of the annual won target). What is the real coverage target? (e.g. "3× quarterly quota," or a sufficiency number finance already uses.)
4. **Should the 5 KPIs be exactly these five?** Any KPI to add (e.g. average deal cycle, # of new logos) or remove?
5. Each card has an 8-quarter **sparkline** trend — confirm that trend window (8 quarters) is right.

---

## 3. JSC Won (left / performance widget)

Branchable bar widget. At "All Healthcare," it can group by **Region → Division → Service Line** or **Division → Region → Service Line** (toggle). Drilling to the deepest level reveals the **won-account list**. A ▼ marker shows the target.

**❓ Decisions needed:**
1. **What exactly is "JSC"?** The prototype maps JSC Won to *annualized won contract value* (Annual Service Revenue of won deals). Confirm the real definition and the exact field. ("JSC" appeared throughout the source workbook but was never defined to us.)
2. **Default grouping:** open on By Region or By Division?
3. **Target line source:** what is the JSC target by region and by division, and where does it come from? (See §6.)

---

## 4. Pipeline by Stage (right / opportunity widget)

Opens as a simple **stage funnel** (stage + amount). Clicking a stage pops out a detail panel: that stage's **divisions**, then a click opens the **service-line drawer** plus an **account list** sorted by **soonest Service Start Date**.

**❓ Decisions needed:**
1. **Stage list:** the prototype uses Qualify → Discovery → Proposal → Negotiation → Verbal. Confirm the real Salesforce stage names and order, and whether to include any pre-Qualify or post-Verbal stages.
2. **Weighted vs Open default:** open on Weighted or Open pipeline?
3. **Weighting source:** weighted = `Amount × stage probability`. Use Salesforce's probability per stage, or a custom weighting leadership prefers?
4. **Account list sort:** confirmed soonest Service Start Date first — is that the right priority sort, or should it be by amount?

---

## 5. Win Rate drill (from the Win Rate KPI)

Clicking the Win Rate card opens a full-width branch panel: win rate by Region → Division → Service Line, color-coded (green/amber/red), with W/L counts.

**❓ Decisions needed:**
1. Win-rate color thresholds are currently green ≥ 55%, amber 45–55%, red < 45%. Confirm the bands leadership wants.
2. Should this drill also be available by **Rep**?

---

## 6. Targets & quotas

Currently the prototype uses **mock targets** derived from last year's actuals × a growth factor, at company / region / division level, plus per-rep quotas. Targets pro-rate to the selected period (quarter = ¼, month = 1/12, YTD = elapsed-fraction).

**❓ Decisions needed (this is a real data gap):**
1. **Company and division targets don't exist yet** — only rep quotas were in the source. Who owns the official targets, and at what granularity (company / region / division / service line / rep)?
2. **Proration:** is straight-line proration acceptable (¼ per quarter), or are targets seasonally weighted (e.g. Q4 heavier)?
3. **"On pace" definition:** the pace badge turns green when attainment ≥ elapsed time fraction. Confirm that's how leadership judges "on track," or supply the real pacing rule.

---

## 7. Pipeline Risk — stalled & pushed

Two clickable cards plus a stalled-by-quarter chart.
- **Stalled:** open deals with **> 60 days in current stage**. Click to see the full list (sorted by days stuck).
- **Pushed:** open deals whose Service Start Date moved. Click to see the full list sorted by Service Start Date, **showing the date of each push in the cycle**.

**❓ Decisions needed:**
1. **Stalled threshold:** 60 days is a guess. What's the real "stalled" cutoff, and should it vary by stage (e.g. Negotiation stalls faster than Qualify)?
2. **"Pushed" definition:** any change to Service Start Date? Only pushes beyond a threshold (e.g. > 30 days)? Only pushes that cross a quarter boundary?
3. The stalled chart buckets by **close quarter** — is that the right axis, or should it bucket by how long they've been stalled?

---

## 8. Rep performance

Visible only when a single region or East-vs-West is selected (hidden at All Healthcare). Ranks reps by JSC Won / Win Rate / Quota Attainment, plus an **"All metrics" table** view. Click a rep for detail.

**❓ Decisions needed:**
1. **Visibility rule:** confirm reps should be hidden at the All-Healthcare level and only appear for regional views. Should CS leadership see CS reps the same way?
2. **Default sort metric:** JSC Won, Win Rate, or Quota %?
3. Any rep metrics to add (e.g. activities, new logos, average cycle)?

---

## 9. Regions & East-vs-West

Views: **All Healthcare, East, West, CS, East vs West.** Company structure modeled as Healthcare → 3 regions (East, West, CS) → 4 divisions (Post Acute, Healthcare, Senior Living, TSOL) → service lines (CNS, EVS, LUM, FAC, PTX).

**❓ Decisions needed:**
1. Confirm the region / division / service-line lists and names are complete and correct.
2. "East vs West" intentionally excludes CS. Is a "compare any two regions" option wanted, or is E-vs-W the only comparison leadership cares about?
3. In E-vs-W, the widgets are independent collapsible trees per region (not full drill-to-accounts). Is comparison-only sufficient there, or do they want full drill in each column?

---

## 10. Visual / UX sign-off

- Performance on the left, Opportunity on the right — confirm this split matches how they read it.
- Color system: teal = primary, amber = caution, red = risk, green = good. Confirm no brand-color requirements.
- Default landing state: All Healthcare, Weighted pipeline, YTD (recommended) — confirm.

---

## 11. Data source & field mapping (for the build team)

The live version connects to Salesforce opportunity data. Fields the dashboard depends on (confirm these exist / map them):

`Region, Division, Service Line, Stage, Amount, Probability/Weighted Amount, Annual Service Revenue (JSC), Expected Revenue, Is Won, Is Closed, Close Date, Service Start Date, Created Date, Days In Stage (or Last Stage Change Date), Service Start Date change history (push dates), Sales Rep, Fiscal Year, Fiscal Quarter, Targets/Quotas.`

**❓ Decisions needed:**
1. Which of these are **not currently captured** in Salesforce and would need to be added? (Likely candidates: days-in-stage / last-stage-change timestamp, SSD change history, official targets.)
2. Refresh cadence — live connection or daily extract?

---

## Consolidated decision checklist

- [ ] Default comparison basis: full-year vs YTD same-point
- [ ] Fiscal year = calendar year?
- [ ] Quarter/Month comparison: prior-year vs prior-period
- [ ] Win rate: by count or by dollars
- [ ] Amount Won card: target % vs YoY
- [ ] Real Pipeline Health coverage target
- [ ] Final KPI set (the five — add/remove?)
- [ ] Definition of "JSC" + exact field
- [ ] JSC default grouping (Region vs Division)
- [ ] Real Salesforce stage names & order
- [ ] Weighted vs Open default + weighting source
- [ ] Win-rate color bands
- [ ] Target ownership & granularity (company/division gap)
- [ ] Target proration (straight-line vs seasonal)
- [ ] "On pace" definition
- [ ] Stalled threshold (and per-stage?)
- [ ] "Pushed" definition
- [ ] Rep visibility rule + default sort
- [ ] Region/division/service-line lists confirmed
- [ ] E-vs-W scope (only E/W? full drill?)
- [ ] Brand colors
- [ ] Missing Salesforce fields to add
- [ ] Refresh cadence
