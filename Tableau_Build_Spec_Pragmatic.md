# Command View — Tableau Build Spec (Pragmatic Edition)

**What this is:** the bridge document. It translates every widget in the HTML mock dashboard into concrete, Tableau-native build steps — calculated fields (with formulas against your real Salesforce fields), sheet layouts, and dashboard actions. It is **medium-independent**: it's instructions, not a file, so it doesn't break on upload. A person in Desktop *or* an assistant generating the workbook follows this one source of truth instead of improvising.

**Pragmatic philosophy:** we reproduce the *intent and ~80% of the feel* using native Tableau (hierarchies, dashboard filter actions, BANs) rather than cloning the prototype's custom JavaScript interactions. Where the prototype does something Tableau can't do natively, the pragmatic substitute is called out explicitly.

**Build order (do not skip):**
1. Connection + parameters → 2. Calculated fields → 3. Individual sheets → 4. Assemble dashboard → 5. Wire actions. Confirm each layer publishes before moving on.

---

## 0. Conventions & field mapping

Real Salesforce fields this spec uses (from your workbook's live connection):

| Concept | Salesforce field |
|---|---|
| Region | `HHS_Region__c` |
| Division | `Division__c` |
| Service Line | `Service_Line__c` |
| Stage | `StageName` |
| Amount | `Amount` |
| Probability | `Probability` |
| JSC (annual service rev) | `Annual_Service_Revenue__c` |
| Won / Closed | `IsWon`, `IsClosed` |
| Close date | `CloseDate` |
| Service start date | `Service_Start_Date__c` |
| Days in stage | `LastStageChangeInDays` |
| Push count | `PushCount` |
| Push dates | from `OpportunityFieldHistory` (or `SSD_Change_1/2/3__c` if materialized) |
| Rep | `Owner_Name` (Owner.Name) |
| Account | `Account_Name` (Account.Name) |

> Naming in this doc uses friendly captions. Internal field names don't matter as long as the formulas reference the real SF fields above.

---

## 1. Parameters (12)

These already exist in the scaffold; recreate if building fresh.

| Parameter | Type | Allowed values | Default |
|---|---|---|---|
| `pTimeView` | string | Selected Period, Date Range, YTD, T12M | Selected Period |
| `pPeriodType` | string | Year, Quarter, Month | Year |
| `pYear` | int | 2023–2029 | 2026 |
| `pQuarter` | int | 1–4 | 1 |
| `pMonth` | int | 1–12 | 1 |
| `pStartDate` | date | any | 2026-01-01 |
| `pEndDate` | date | any | 2026-12-31 |
| `pRegionView` | string | All, East, West, CS | All |
| `pPipelineView` | string | Weighted, Open | Weighted |
| `pJSCGroup` | string | Region, Division | Region |
| `pCoverageGoal` | float | any | 13,000,000 |
| `pIncludeMissingSSD` | string | Include, Exclude | Include |

---

## 2. Calculated fields

### 2.1 Period & region scoping (foundation — everything depends on these)

**`In Selected Period`** (boolean, by Close Date)
```
IF [pTimeView]="Selected Period" THEN
  IF [pPeriodType]="Year"    THEN YEAR([CloseDate])=[pYear]
  ELSEIF [pPeriodType]="Quarter" THEN YEAR([CloseDate])=[pYear] AND DATEPART('quarter',[CloseDate])=[pQuarter]
  ELSEIF [pPeriodType]="Month"   THEN YEAR([CloseDate])=[pYear] AND DATEPART('month',[CloseDate])=[pMonth]
  END
ELSEIF [pTimeView]="YTD"  THEN YEAR([CloseDate])=[pYear] AND [CloseDate] <= MAKEDATE([pYear],DATEPART('month',TODAY()),DATEPART('day',TODAY()))
ELSEIF [pTimeView]="T12M" THEN [CloseDate] > DATEADD('year',-1,TODAY()) AND [CloseDate] <= TODAY()
ELSEIF [pTimeView]="Date Range" THEN [CloseDate] >= [pStartDate] AND [CloseDate] <= [pEndDate]
END
```

**`JSC In Selected Period`** — identical but replace `[CloseDate]` with `[Service_Start_Date__c]`. (JSC is dated by Service Start Date in your source.) If `pIncludeMissingSSD`="Exclude", AND it with `NOT ISNULL([Service_Start_Date__c])`.

**`In Prior Period`** (boolean, by Close Date — powers all YoY arrows)
```
IF [pTimeView]="Selected Period" THEN
  IF [pPeriodType]="Year"    THEN YEAR([CloseDate])=[pYear]-1
  ELSEIF [pPeriodType]="Quarter" THEN YEAR([CloseDate])=[pYear]-1 AND DATEPART('quarter',[CloseDate])=[pQuarter]
  ELSEIF [pPeriodType]="Month"   THEN YEAR([CloseDate])=[pYear]-1 AND DATEPART('month',[CloseDate])=[pMonth]
  END
ELSEIF [pTimeView]="YTD"  THEN YEAR([CloseDate])=[pYear]-1 AND [CloseDate] <= MAKEDATE([pYear]-1,DATEPART('month',TODAY()),DATEPART('day',TODAY()))
ELSEIF [pTimeView]="T12M" THEN [CloseDate] > DATEADD('year',-2,TODAY()) AND [CloseDate] <= DATEADD('year',-1,TODAY())
ELSEIF [pTimeView]="Date Range" THEN
   [CloseDate] >= DATEADD('day', -1*(DATEDIFF('day',[pStartDate],[pEndDate])+1), [pStartDate]) AND [CloseDate] < [pStartDate]
END
```
**`JSC In Prior Period`** — same, on `[Service_Start_Date__c]`.

**`In Selected Region`** (boolean)
```
[pRegionView]="All" OR [HHS_Region__c]=[pRegionView]
```

### 2.2 Scoped measures (period+region baked in → sheets need NO worksheet filters)

| Field | Formula |
|---|---|
| `Amount Won (cur)` | `IF [In Selected Period] AND [In Selected Region] AND [IsWon] THEN [Amount] END` |
| `Amount Won (prior)` | `IF [In Prior Period] AND [In Selected Region] AND [IsWon] THEN [Amount] END` |
| `JSC Won (cur)` | `IF [JSC In Selected Period] AND [In Selected Region] AND [IsWon] THEN [Annual_Service_Revenue__c] END` |
| `JSC Won (prior)` | `IF [JSC In Prior Period] AND [In Selected Region] AND [IsWon] THEN [Annual_Service_Revenue__c] END` |
| `Revenue Expected (cur)` | `IF [In Selected Period] AND [In Selected Region] AND [IsWon] THEN [ExpectedRevenue] END` |
| `Won Flag (cur)` | `IF [In Selected Period] AND [In Selected Region] AND [IsWon] THEN 1 END` |
| `Lost Flag (cur)` | `IF [In Selected Period] AND [In Selected Region] AND [IsClosed] AND NOT [IsWon] THEN 1 END` |
| `Won Flag (prior)` | `IF [In Prior Period] AND [In Selected Region] AND [IsWon] THEN 1 END` |
| `Lost Flag (prior)` | `IF [In Prior Period] AND [In Selected Region] AND [IsClosed] AND NOT [IsWon] THEN 1 END` |
| `Win Rate (cur)` | `SUM([Won Flag (cur)]) / (SUM([Won Flag (cur)]) + SUM([Lost Flag (cur)]))` |
| `Win Rate (prior)` | `SUM([Won Flag (prior)]) / (SUM([Won Flag (prior)]) + SUM([Lost Flag (prior)]))` |
| `Pipeline $ (cur)` | `IF [In Selected Region] AND NOT [IsClosed] THEN (IF [pPipelineView]="Weighted" THEN [Amount]*[Probability] ELSE [Amount] END) END` |
| `Weighted Open $` | `IF [In Selected Region] AND NOT [IsClosed] THEN [Amount]*[Probability] END` |
| `Coverage Ratio` | `SUM([Weighted Open $]) / [pCoverageGoal]` |
| `Is Stalled` | `NOT [IsClosed] AND [LastStageChangeInDays] > 60` |
| `Stalled Count` | `IF [In Selected Region] AND [Is Stalled] THEN 1 END` |
| `Is Pushed` | `[PushCount] > 0` |
| `Pushed Open $` | `IF [In Selected Region] AND [Is Pushed] AND NOT [IsClosed] THEN [Amount] END` |
| `JSC Group Dimension` | `IF [pJSCGroup]="Region" THEN [HHS_Region__c] ELSE [Division__c] END` |
| `Close Quarter` | `STR(YEAR([CloseDate]))+" Q"+STR(DATEPART('quarter',[CloseDate]))` |

### 2.3 YoY arrow fields (the supporting KPI info ChatGPT dropped)

For each KPI, three fields. Example for Amount Won (repeat pattern for JSC, Revenue, Win Rate, Pipeline):

**`Amount Won YoY %`** (number)
```
(SUM([Amount Won (cur)]) - SUM([Amount Won (prior)])) / ABS(SUM([Amount Won (prior)]))
```
**`Amount Won Arrow`** (string — the ▲/▼ + %)
```
IF [Amount Won YoY %] >= 0 THEN "▲ " ELSE "▼ " END
+ STR(ROUND(ABS([Amount Won YoY %])*100,0)) + "% vs " + STR([pYear]-1)
```
**`Amount Won Trend Color`** (string — drives green/red)
```
IF [Amount Won YoY %] >= 0 THEN "Up" ELSE "Down" END
```
For the **Amount Won pace badge** (% of target) instead of YoY, use:
`SUM([Amount Won (cur)]) / ([pCoverageGoal] ...)` — note targets above rep level don't exist in SF yet; see Open Decisions.

---

## 3. Per-widget build specs

### Widget A — Global controls (top bar)
- Show parameter controls on the dashboard: `pTimeView`, `pPeriodType`, `pYear`, `pQuarter`, `pMonth`, `pStartDate`, `pEndDate`, `pRegionView`, `pPipelineView`, `pJSCGroup`.
- Pragmatic note: the prototype hides/shows Quarter vs Month pickers contextually. Tableau can't conditionally hide a parameter control — **leave all visible**, or group them in a collapsible container. Acceptable trade-off.

### Widget B — KPI strip (5 cards, each with arrow + sparkline)
For each KPI build **two stacked sheets**, then place them together on the dashboard:

**B1. The BAN sheet** (e.g. "KPI - Amount Won")
- Marks: Automatic/Text.
- Text shelf: `SUM([Amount Won (cur)])` (big), and on a second text line `[Amount Won Arrow]`.
- Color shelf: `[Amount Won Trend Color]` (map Up=green, Down=red) — colors only the arrow text.
- No filters (scoping is in the measure).

**B2. The sparkline sheet** ("Spark - Amount Won")
- Columns: `MONTH([CloseDate])` or a `Quarter` continuous date (last 8 quarters).
- Rows: `SUM([Amount Won (cur)])` — but for trend you want it *unscoped by period*; use a trend-specific measure `IF [In Selected Region] AND [IsWon] THEN [Amount] END` so the line shows history regardless of the period filter.
- Marks: Line. Hide axes, headers, gridlines, tooltips off. Size small.
- Place directly under the BAN on the dashboard.

Repeat B1/B2 for: **JSC Won**, **Revenue Expected**, **Win Rate**, **Weighted/Open Pipeline**. For **Pipeline Health** show `[Coverage Ratio]` as the number and a Healthy/Watch/At-risk label (`IF [Coverage Ratio]>=1 THEN "Healthy" ELSEIF >=0.75 THEN "Watch" ELSE "At risk" END`) instead of an arrow.

### Widget C — JSC Won (drill)
**Pragmatic substitute for the custom branch widget: a Tableau hierarchy + a detail action.**
1. Create hierarchy **JSC Drill**: drag `HHS_Region__c` → onto `Division__c` → onto `Service_Line__c` (order: Region, Division, Service Line). *(If `pJSCGroup`=Division desired as top, build a second hierarchy Division→Service Line, or just use the hierarchy and let users collapse.)*
2. Sheet "JSC Won": Rows = **JSC Drill** hierarchy; Columns = `SUM([JSC Won (cur)])`; Marks = Bar; Color by the top level. Users expand with the **+** control — native drill.
3. Sheet "JSC Accounts" (detail): Rows = `Account_Name`, `Owner_Name`; Text = `SUM([Annual_Service_Revenue__c])`, `CloseDate`. Filtered later by dashboard action.
4. Target marker: add a reference line at `[pCoverageGoal]`-derived target, or a `JSC Target` calc once targets exist.

### Widget D — Pipeline by Stage
1. Hierarchy **Pipeline Drill**: `StageName` → `Division__c` → `Service_Line__c`.
2. Sheet "Pipeline by Stage": Rows = **Pipeline Drill**; Columns = `SUM([Pipeline $ (cur)])`; Marks = Bar; sort stages in funnel order via a manual sort or a `Stage Sort` calc (`CASE [StageName] WHEN "Qualify" THEN 1 … END`).
3. Sheet "Pipeline Accounts": Rows = `Account_Name`,`Owner_Name`,`Service_Start_Date__c`; Columns/Text = `SUM([Pipeline $ (cur)])`; **sort ascending by `Service_Start_Date__c`** (soonest service start first). Filtered by dashboard action from the bar.
4. `pPipelineView` (Weighted/Open) already drives the measure — no extra work.

### Widget E — Win Rate drill
1. Hierarchy **Win Rate Drill**: `HHS_Region__c` → `Division__c` → `Service_Line__c`.
2. Sheet "Win Rate": Rows = hierarchy; Columns = `[Win Rate (cur)]`; Marks = Bar; Color by a band calc:
   `IF [Win Rate (cur)]>=0.55 THEN "Good" ELSEIF [Win Rate (cur)]>=0.45 THEN "Watch" ELSE "Low" END` (green/amber/red).
3. Label with `[Win Rate (cur)]` (%) and `SUM([Won Flag (cur)])`/`SUM([Lost Flag (cur)])` in tooltip.

### Widget F — Pipeline Risk (stalled + pushed)
- **F1 "Stalled BAN":** Text = `SUM([Stalled Count])` and `SUM(IF [Is Stalled] AND [In Selected Region] THEN [Amount] END)`.
- **F2 "Pushed BAN":** Text = `SUM(IF [Is Pushed] AND NOT [IsClosed] AND [In Selected Region] THEN 1 END)` and `SUM([Pushed Open $])`.
- **F3 "Stalled by Quarter" chart:** Columns = `[Close Quarter]`; Rows = `SUM([Stalled Count])`; Marks = Bar. (This is the chart you asked to bring back — native and trivial.)
- **F4 "Pushed Deals" detail:** Rows = `Account_Name`,`Owner_Name`,`Service_Start_Date__c`; Text columns = `SSD_Change_1__c`, `SSD_Change_2__c`, `SSD_Change_3__c` (the push dates), `SUM([Amount])`. Filter: `Is Pushed`=True, `IsClosed`=False, `In Selected Region`=True. Sort ascending by Service Start Date.
- **F5 "Stalled Deals" detail:** Rows = `Account_Name`,`StageName`,`Owner_Name`; Text = `LastStageChangeInDays`,`SUM([Amount])`. Sort descending by `LastStageChangeInDays`.
- Pragmatic note: the prototype toggles the lists open/closed by clicking the BANs. In Tableau, use a **dashboard filter action** (select stalled BAN → show stalled list) or just place both lists below the BANs permanently.

### Widget G — Rep Performance
- **G1 "Rep Ranked":** Rows = `Owner_Name` (sorted desc by the chosen measure); Columns = `SUM([JSC Won (cur)])`; Marks = Bar. Add a parameter `pRepMetric` (JSC Won / Win Rate / Quota %) and a `Rep Metric` calc that returns the selected measure to make it switchable.
- **G2 "Rep All Metrics" table:** Rows = `Owner_Name`; Columns = Measure Names (or individual text columns) showing `SUM([JSC Won (cur)])`, `[Win Rate (cur)]`, quota %, `SUM([Weighted Open $])`, `SUM([Stalled Count])`. This is the "all metrics at once" view — a native text table.
- Visibility: the prototype hides reps at All-Healthcare. Pragmatic: use a **container with a Show/Hide button**, or a filter action, or simply leave it and let `pRegionView` scope it.

### Widget H — Region / East-vs-West
- Single region: `pRegionView` already scopes everything. Done.
- **East vs West (pragmatic):** Tableau can't do the prototype's dynamic split from one parameter cleanly. Two options:
  - (a) Build a duplicate set of the key sheets, one filtered `HHS_Region__c`="East", one ="West", placed side by side on a **separate dashboard** "East vs West". Lower effort, clear result.
  - (b) Add `HHS_Region__c` to Columns on the relevant sheets to get small multiples. Quick but less controllable.
  Recommend (a) for the 2–3 widgets execs compare (JSC Won, Pipeline, Rep).

---

## 4. Dashboard assembly & action wiring

Layout: title bar; parameter controls (top/left); KPI strip (5 BAN+sparkline pairs) across the top; Performance left (JSC Won, Win Rate, Rep), Opportunity right (Pipeline, Risk).

**Actions (Dashboard → Actions):**

| # | Type | Source sheet | Target sheet(s) | Run on | Effect |
|---|---|---|---|---|---|
| 1 | Filter | JSC Won | JSC Accounts | Select | Click a JSC bar → account list for that node |
| 2 | Filter | Pipeline by Stage | Pipeline Accounts | Select | Click a stage/division → accounts (soonest SSD first) |
| 3 | Filter | Win Rate | (tooltip or detail) | Select | Optional drill on win rate |
| 4 | Filter | Stalled BAN | Stalled Deals | Select | Show stalled list |
| 5 | Filter | Pushed BAN | Pushed Deals | Select | Show pushed list + dates |
| 6 | Filter | Rep Ranked | Rep All Metrics | Select | Click a rep → their detail |

Native hierarchy +/- handles the in-chart drill; the actions above handle the master-detail "click to see accounts" behavior.

---

## 5. Build checklist

- [ ] 12 parameters created
- [ ] Foundation calcs: In Selected Period, JSC In Selected Period, In Prior Period (+JSC), In Selected Region
- [ ] Scoped measures (§2.2)
- [ ] YoY arrow fields for the 5 KPIs (§2.3)
- [ ] 3 hierarchies: JSC Drill, Pipeline Drill, Win Rate Drill
- [ ] 5 KPI BAN sheets + 5 sparkline sheets
- [ ] JSC Won + JSC Accounts
- [ ] Pipeline by Stage + Pipeline Accounts
- [ ] Win Rate
- [ ] Risk: Stalled BAN, Pushed BAN, Stalled by Quarter, Pushed Deals, Stalled Deals
- [ ] Rep Ranked + Rep All Metrics
- [ ] East vs West dashboard (duplicate sheets)
- [ ] Dashboard assembled + 6 actions wired
- [ ] Publish to Tableau Cloud, confirm live connection refresh

---

## 6. Open decisions (carry from the sign-off doc)

1. **Targets above rep level don't exist in Salesforce** — every pace/coverage/target marker needs a real source. Until then `pCoverageGoal` is a single placeholder knob.
2. **JSC date basis = Service Start Date** (confirm).
3. **`pIncludeMissingSSD`** rule for null SSD.
4. **Win rate** by count (current) vs dollars.
5. **Stalled threshold** = 60 days (confirm, maybe per-stage).
6. **Region field** `HHS_Region__c` vs `Affiliation_Region__c`.
7. **Service line override** precedence (`ACR_Service_Line_Override__c`).

---

*Pragmatic trade-offs accepted in this version: native hierarchy drills instead of the prototype's branch/pop-out animation; parameter controls always visible; East-vs-West as a duplicate-sheet dashboard rather than a dynamic split. All reproduce the information and the drill paths; they differ only in interaction polish.*
