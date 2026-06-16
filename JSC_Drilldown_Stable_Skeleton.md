# JSC Won Drill-Down — Stable Tableau Skeleton (for ChatGPT)

## Context for the assistant
We are building a Tableau dashboard widget that drills:

```
Region → Division → Service Line → Account Detail
```

It must show only records that contributed to the **selected-period won JSC amount**. The account
level must exclude anything outside the selected period or any record that didn't contribute to the
won measure.

**Problem we are solving:** the drill works on the worksheet but the click does not fire on the
dashboard. Build the **stable** version below (Option A). Do NOT default to the fragile single-sheet
multi-set-action version (Option B) unless explicitly asked — it is included only as a fallback.

This is a **build guide / spec**, not a file to hand-edit. The human will build the sheets in
Tableau Desktop / Cloud by dragging fields. Do not generate or hand-edit `.twb`/`.twbx` XML —
Tableau rejects hand-authored worksheet/dashboard structure. Produce calculated-field formulas and
step-by-step shelf/marks/action instructions only.

---

## Field mapping (real Salesforce fields)

| Role | Salesforce field |
|---|---|
| Top Category | `HHS_Region__c` |
| Subcategory | `Division__c` |
| Detail Category | `Service_Line__c` |
| Account | `Account_Name` (Account.Name) |
| Opportunity (optional) | `Opportunity_Name` / `Name` |
| Service Start Date | `Service_Start_Date__c` |
| JSC row amount | `Annual_Service_Revenue__c` |
| Won flag | `IsWon` |
| Closed flag | `IsClosed` |

> JSC is dated by **Service Start Date**, not Close Date. The period filter below uses
> `Service_Start_Date__c` deliberately.

---

## Parameters this widget depends on

Create these if they don't already exist (string unless noted):

| Parameter | Values | Default |
|---|---|---|
| `pTimeView` | Selected Period, Date Range, YTD, T12M | Selected Period |
| `pPeriodType` | Year, Quarter, Month | Year |
| `pYear` (integer) | 2023–2029 | 2026 |
| `pQuarter` (integer) | 1–4 | 1 |
| `pMonth` (integer) | 1–12 | 1 |
| `pStartDate` (date) | any | 2026-01-01 |
| `pEndDate` (date) | any | 2026-12-31 |
| `pRegionView` (optional) | All, East, West, CS | All |

---

## Calculated fields (copy-paste, exact)

### 1. JSC In Selected Period — boolean (dated by Service Start Date)
```tableau
IF [pTimeView] = "Selected Period" THEN
    IF [pPeriodType] = "Year" THEN
        YEAR([Service_Start_Date__c]) = [pYear]
    ELSEIF [pPeriodType] = "Quarter" THEN
        YEAR([Service_Start_Date__c]) = [pYear]
        AND DATEPART('quarter', [Service_Start_Date__c]) = [pQuarter]
    ELSEIF [pPeriodType] = "Month" THEN
        YEAR([Service_Start_Date__c]) = [pYear]
        AND DATEPART('month', [Service_Start_Date__c]) = [pMonth]
    END
ELSEIF [pTimeView] = "YTD" THEN
    YEAR([Service_Start_Date__c]) = [pYear]
    AND [Service_Start_Date__c] <= MAKEDATE([pYear], DATEPART('month', TODAY()), DATEPART('day', TODAY()))
ELSEIF [pTimeView] = "T12M" THEN
    [Service_Start_Date__c] > DATEADD('year', -1, TODAY())
    AND [Service_Start_Date__c] <= TODAY()
ELSEIF [pTimeView] = "Date Range" THEN
    [Service_Start_Date__c] >= [pStartDate]
    AND [Service_Start_Date__c] <= [pEndDate]
END
```

### 2. In Selected Region — boolean (optional, if the widget should respect region)
```tableau
[pRegionView] = "All" OR [HHS_Region__c] = [pRegionView]
```

### 3. JSC Won - Selected Period Row — row-level measure (the base measure)
```tableau
IF [IsWon]
   AND [JSC In Selected Period]
   AND ([pRegionView] = "All" OR [HHS_Region__c] = [pRegionView])
THEN ZN([Annual_Service_Revenue__c])
END
```
Use `SUM([JSC Won - Selected Period Row])` in views. Aggregate it in the view; never put SUM inside
a dimension calc (causes aggregate/non-aggregate errors).

---

# OPTION A — Recommended stable design (build this)

Two sheets + one filter action. Tiers 1–3 use a native Tableau **hierarchy** (the +/- expand), so
there are no set actions to misfire. Only the account level uses an action.

## A1. Build the hierarchy
In the Data pane, drag `HHS_Region__c` onto `Division__c`, then `Service_Line__c` onto that, to make
a hierarchy named **JSC Drill** (order: Region → Division → Service Line).

## A2. Summary sheet — "JSC Won Drill"
```
Rows:    JSC Drill   (the hierarchy; users expand with the + control)
Columns: SUM([JSC Won - Selected Period Row])
Marks:   Bar
Color:   HHS_Region__c   (color by top level)
Label:   SUM([JSC Won - Selected Period Row])
Filters:
   [IsWon] = True
   [JSC In Selected Period] = True
   SUM([JSC Won - Selected Period Row]) > 0      (filters out non-contributing rows)
Sort:    descending by SUM([JSC Won - Selected Period Row])
```

## A3. Account detail sheet — "JSC Won Accounts"
```
Rows / Text:
   [Account_Name]
   [Opportunity_Name]        (optional)
   [Service_Start_Date__c]
Measure (Text / Columns):
   SUM([JSC Won - Selected Period Row])
Marks:   Text (or Bar)
Filters:
   [IsWon] = True
   [JSC In Selected Period] = True
   SUM([JSC Won - Selected Period Row]) > 0
Sort:    by [Service_Start_Date__c] ascending (soonest service start first)
```

## A4. Dashboard + the one action
1. Put **JSC Won Drill** and **JSC Won Accounts** on the dashboard.
2. Dashboard → Actions → Add Action → **Filter**:
```
Name:               Drill to Accounts
Source Sheet:       JSC Won Drill        (must match the sheet instance on the dashboard)
Target Sheet:       JSC Won Accounts
Run action on:      Select
Selected Fields:    HHS_Region__c, Division__c, Service_Line__c
Clearing the selection:  Exclude all values   (detail list is empty until a bar is clicked)
```
Clicking a bar at any level filters the account list to that node. Expanding tiers 1–3 is handled by
the hierarchy's +/- control — nothing to break.

---

# Why the action fires on the worksheet but NOT the dashboard (diagnosis)

Work through these in order — the first is by far the most common.

1. **A floating object is intercepting the click.** Any floating object stacked over the drill sheet
   (transparent container, blank, background image, floating legend) eats the click before the sheet
   receives it. Open the dashboard **Layout pane** (top-left item hierarchy) and confirm nothing is
   layered on top of the drill sheet. This is the #1 cause in float-heavy layouts.

2. **You're clicking a row header, not a mark.** Actions fire from selecting a **mark** (the bar), not
   the text label on the row shelf. If a level shows a header but no bar (because the value is null or
   zero), there's no mark to click. Ensure every level draws a bar.

3. **Source Sheet mismatch / wrong scope.** The action's Source Sheet must be the exact sheet placed on
   the dashboard. If "Selected Fields" lists a field not in the view, it silently won't fire. Use the
   three category fields that are actually in the view.

4. **Multiple Select set actions colliding** (only in Option B). If several set actions are "Run on:
   Select" on one sheet, they all fire on a single click and open levels too early. Fix by setting the
   deeper-level actions to **Menu**.

5. **Marks not selectable.** Confirm the sheet allows selection (not disabled), and that the marks are
   real bars, not just headers.

### Fastest isolation test
Create a **brand-new blank dashboard**, drop ONLY the drill sheet and ONE action, and test.
- Works there → the problem is your real dashboard's layout (almost always a floating overlay, #1).
- Fails there → the problem is the action config (#3) or no selectable mark (#2).

---

# OPTION B — Single-sheet set-action drill (FRAGILE fallback only)

Use only if in-place expansion without the +/- control is explicitly required. More moving parts,
more ways to break on a dashboard.

## B1. Action-key fields (duplicates so visible dims don't conflict with set targets)
```tableau
// Region Action Key
[HHS_Region__c]
```
```tableau
// Division Action Key
[Division__c]
```
```tableau
// Service Line Action Key
[Service_Line__c]
```

## B2. Sets
```
[Region Action Set]       from [Region Action Key]
[Division Action Set]      from [Division Action Key]
[Service Line Action Set]  from [Service Line Action Key]
```

## B3. Drill calcs
```tableau
// Drill Level 1 - Division
IF [Region Action Set] THEN IFNULL([Division__c], "Unassigned Division") END
```
```tableau
// Drill Level 2 - Service Line
IF [Division Action Set] AND NOT ISNULL([Drill Level 1 - Division])
THEN IFNULL([Service_Line__c], "Unassigned Service Line") END
```
```tableau
// Drill Level 3 - Account Detail
IF [Service Line Action Set]
   AND NOT ISNULL([Drill Level 2 - Service Line])
   AND [IsWon]
   AND [JSC In Selected Period]
   AND NOT ISNULL([JSC Won - Selected Period Row])
THEN IFNULL([Account_Name], "Unassigned Account") + " | " + STR(DATE([Service_Start_Date__c]))
END
```

## B4. Worksheet
```
Rows:    [HHS_Region__c], [Drill Level 1 - Division], [Drill Level 2 - Service Line], [Drill Level 3 - Account Detail]
Columns: SUM([JSC Won - Selected Period Row])
Marks:   Bar
Label:   SUM([JSC Won - Selected Period Row])
Detail:  [Region Action Key], [Division Action Key], [Service Line Action Key]
Tooltip: Account_Name, Service_Start_Date__c, SUM([JSC Won - Selected Period Row])
Filters: [IsWon]=True, [JSC In Selected Period]=True, SUM([JSC Won - Selected Period Row]) > 0
```

## B5. Set actions
```
Action 1 — Select Region:       Source = drill sheet, Run on = Select, Target = [Region Action Set],       Assign values, Clearing = Remove all values
Action 2 — Select Division:     Source = drill sheet, Run on = Menu,   Target = [Division Action Set],      Assign values, Clearing = Remove all values
Action 3 — Show Account Detail: Source = drill sheet, Run on = Menu,   Target = [Service Line Action Set],  Assign values, Clearing = Remove all values
```
Deeper levels are **Menu** (not Select) to avoid the collision in diagnosis #4.

---

## Known pitfalls (apply to both options)
- Keep the won amount as a **measure** on Columns/Label/Tooltip — never inside a text-dimension calc.
- Gate the account level by `[IsWon]`, `[JSC In Selected Period]`, and `SUM(...) > 0`.
- Dashboard actions need a real selectable **mark**, not just a row header.
- If a floating object overlaps the sheet, the click never reaches it.
- Test on a clean blank dashboard with one action before debugging the full layout.

---

## Specific ask for ChatGPT
Build **Option A** (hierarchy + single filter action) for a Region → Division → Service Line → Account
drill on JSC Won, scoped to the selected period (by Service Start Date) and showing only records that
contributed to the won amount. Provide: the calculated fields (above), exact shelf/marks/filter setup
for both sheets, and the exact filter-action configuration. Then give a short troubleshooting checklist
for "action fires on the worksheet but not the dashboard," prioritizing the floating-overlay and
header-vs-mark causes. Do not produce or hand-edit workbook XML.
