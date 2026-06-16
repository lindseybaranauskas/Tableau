# JSC Drill Widget Issue Brief and Sanitized Workbook Skeleton

Prepared for external troubleshooting. All company, person, account, opportunity, and Salesforce ID details have been stripped or generalized.

## 1. Current symptom

The goal is a single Tableau worksheet that behaves like a controlled drill:

```text
Click Region
→ show Divisions

Click Division
→ show Service Lines and all won account/opportunity rows for the selected Division
```

The current issue is that when the second action is set to Select, the click behavior becomes too broad. Instead of only opening the selected Division, Tableau can show all available Division / Service Line / Account detail beneath the selected Region, or it can retain multiple Divisions in the middle drill set.

## 2. Main diagnosis

This is primarily an action-state / single-sheet set-action problem, not a data-calculation problem.

The workbook is using chained Set Actions on the same source worksheet. That source worksheet already contains all drill-level fields on Rows:

```text
Region
Division Drill Display
Service Line Drill Display
Account / Opportunity Drill Display
```

Because all levels are present on the same worksheet, Tableau can evaluate the same click against multiple drill levels at once. When both the Region action and Division action run on Select from the same source sheet, the second action may fire from the same click or capture more child values than intended.

The behavior becomes worse when the middle set action uses Add values instead of Assign values. Add values lets multiple Divisions remain selected at the same time, which makes later levels appear broader than expected.

## 3. Current state observed in workbook XML

The workbook currently stores these relevant drill states:

```text
Region Set = selected Region A
Division Drill Set = Division A + Division B
Service Line Drill Set = empty
```

The middle action is configured as:

```text
Action: Division drill action
Target Set: Division Drill Display Set
Running action: Add values to set
Activation: not cleanly present in XML after switching between Select/Menu
```

This explains why details can feel stuck or overly broad. Tableau is not holding exactly one selected Division.

## 4. Important note about the excluded blank row

At one point a blank/container row was excluded. That can break this kind of drill because blank rows are not always cosmetic. In a drill worksheet, a blank lower-level value often represents a closed branch.

A generated Exclusions group exists in the workbook metadata. It should not be used as an active worksheet filter for the drill fields. Remove any active worksheet filter named like:

```text
Exclusions (...Region Set..., Account Drill, Division Drill, Service Line Drill, Region...)
```

or any filter on:

```text
Division Drill Display
Service Line Drill Display
Account Drill Display
```

unless it was intentionally added.

## 5. Why the current account detail finally works

The account drill was fixed when it stopped depending on the final Service Line Set. The working logic is:

```text
If a Service Line is visible, and the row is won, in selected period, and has positive JSC value,
then show the account/opportunity label.
```

This allows one-opportunity service lines to display. The final Service Line Set should not be required for the main display.

## 6. Recommended stable action layout for current design

For the current single-sheet set-action design, the safest stable setup is:

```text
Action 1: Region → Division
Type: Change Set Values
Run on: Select
Target Set: Region Set
Running: Assign values to set
Clearing: Remove all values from set

Action 2: Division → Service Lines + Accounts
Type: Change Set Values
Run on: Menu for stability, Select only if accepted as fragile
Target Set: Division Drill Display Set
Running: Assign values to set
Clearing: Remove all values from set

Action 3: Service Line → Accounts
Disable for now, or keep as Menu only for optional narrowing
Target Set: Service Line Drill Display Set
Running: Assign values to set
Clearing: Remove all values from set
```

If Action 2 must be Select, expect the risk that Region click may trigger downstream opening because both actions are sourced from the same worksheet.

## 7. Best-practice alternative

A parameter-action drill is the cleaner architecture for true click-click behavior:

```text
pSelectedRegion
pSelectedDivision
pSelectedServiceLine
```

Each click stores a single selected value in a parameter, and calculated fields decide which level to display. Parameters hold one value, while sets can hold multiple values. This makes the drill state easier to control.

## 8. Sanitized workbook skeleton

### Data source

```text
Data Source: Salesforce Opportunity Extract
Primary grain: opportunity row
Related objects: account, opportunity history, opportunity stage
Extract type: packaged extract / Hyper
```

### Core raw fields

```text
Region Source Field / Owner Mapping Field
Division
Service Line
Account Name
Opportunity Name
Service Start Date
Amount / Annual Revenue
Stage
Is Won
Is Closed
Opportunity ID
Account ID
Owner ID
```

### Parameters

```text
pTime View              = Year / Quarter / Month
pSelected Year          = integer year
pSelected Quarter       = 1-4
pSelected Month         = 1-12
pRegion View            = All / Region A / Region B / Region C
pJSC Target             = numeric target
pSelected Metric        = optional KPI selection
pSelected Division      = optional panel selection
pSelected Service Line  = optional panel selection
```

### Period calculations

```tableau
// Selected Period Start
CASE [pTime View]
WHEN "Year" THEN MAKEDATE([pSelected Year], 1, 1)
WHEN "Quarter" THEN MAKEDATE([pSelected Year], 1 + 3 * ([pSelected Quarter] - 1), 1)
WHEN "Month" THEN MAKEDATE([pSelected Year], [pSelected Month], 1)
END

// Selected Period End
CASE [pTime View]
WHEN "Year" THEN DATEADD('day', -1, DATEADD('year', 1, MAKEDATE([pSelected Year], 1, 1)))
WHEN "Quarter" THEN DATEADD('day', -1, DATEADD('month', 3, MAKEDATE([pSelected Year], 1 + 3 * ([pSelected Quarter] - 1), 1)))
WHEN "Month" THEN DATEADD('day', -1, DATEADD('month', 1, MAKEDATE([pSelected Year], [pSelected Month], 1)))
END

// In Selected Period
[Service Start Date] >= [Selected Period Start]
AND [Service Start Date] <= [Selected Period End]
```

### JSC value calculations

```tableau
// JSC Won Current Row
IF [Region View] = "All" OR [Region] = [Region View]
AND [Is Won]
AND [In Selected Period]
THEN [Amount]
END

// JSC Won
SUM([JSC Won Current Row])
```

### Current set-action drill calculations

```tableau
// Division Drill Display
IF [Region Set] THEN
    IFNULL([Division], "Unassigned Division")
ELSE
    ""
END

// Service Line Drill Display
IF [Division Drill Display Set]
AND [Division Drill Display] <> ""
AND [Division Drill Display] = IFNULL([Division], "Unassigned Division")
AND [Is Won]
AND [In Selected Period]
AND NOT ISNULL([JSC Won Current Row])
AND [JSC Won Current Row] > 0
THEN
    IFNULL([Service Line], "Unassigned Service Line")
ELSE
    ""
END

// Account / Opportunity Drill Display
IF [Service Line Drill Display] <> ""
AND [Is Won]
AND [In Selected Period]
AND NOT ISNULL([JSC Won Current Row])
AND [JSC Won Current Row] > 0
THEN
    IFNULL([Opportunity Name], "Unassigned Opportunity")
    + " | "
    + STR(DATE([Service Start Date]))
ELSE
    ""
END
```

### Main drill worksheet

```text
Worksheet: JSC Won Region View Drill Down

Rows:
1. Region
2. Division Drill Display
3. Service Line Drill Display
4. Account / Opportunity Drill Display

Columns:
- JSC Won

Marks:
- Bar
- Label: JSC Won
- Color: In/Out of Region Set, or neutral region color

Filters:
- Keep broad business filters only
- Do not use generated Exclusions filters on drill display fields
```

### Dashboard actions

```text
Action 1: Region Set Action
Source: Main drill worksheet
Run on: Select
Target Set: Region Set
Action behavior: Assign values
Clear behavior: Remove all values

Action 2: Division Drill Action
Source: Main drill worksheet
Run on: Menu for stability, Select for fragile two-click attempt
Target Set: Division Drill Display Set
Action behavior: Assign values, not Add values
Clear behavior: Remove all values

Action 3: Service Line Drill Action
Source: Main drill worksheet
Run on: Menu only or disabled
Target Set: Service Line Drill Display Set
Action behavior: Assign values
Clear behavior: Remove all values
```

## 9. Recommended question for another model or Tableau expert

```text
I have a single Tableau worksheet used as a multi-level drilldown. Rows contain Region, a calculated Division drill display, a calculated Service Line drill display, and an Account/Opportunity drill display. The view uses set actions from the same source worksheet: Region Set and Division Drill Display Set. When both actions run on Select, clicking a Region immediately opens too many downstream levels or populates the Division set too broadly. The middle action also previously used Add values, which caused multiple Divisions to remain in the set. I need true two-click behavior: click Region to show Divisions, then click one Division to show Service Lines and all account rows for that Division. Is this achievable with chained set actions on one worksheet, or should this be rebuilt using parameter actions or separate sheets?
```

## 10. Recommendation

Do not keep rewriting the account calculations. They are now the part that is closest to correct.

The issue to solve is the interaction architecture:

```text
Single source worksheet + chained set actions + Select activation = fragile cascading behavior.
```

The clean path is either:

```text
A. Keep current set-action version but use Select / Menu / Off for stability.
```

or:

```text
B. Build a duplicate parameter-action version for true controlled click-click behavior.
```
