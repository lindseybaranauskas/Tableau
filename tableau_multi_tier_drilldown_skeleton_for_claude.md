# Sanitized Tableau Multi-Tier Drill-Down Skeleton

## Goal
Build a Tableau dashboard widget that supports a multi-tier drill-down using Set Actions or Parameter Actions. The intended drill path is:

```text
Top Category → Subcategory → Detail Category → Account Detail
```

The widget should show only records that contribute to the current selected-period won amount. The final account/detail level should not include records outside the selected period or records that did not contribute to the won measure.

## Generic field mapping
Use these generic field names in place of private/company-specific fields:

| Generic field | Meaning |
|---|---|
| `[Top Category]` | Highest-level grouping, for example Region |
| `[Subcategory]` | Second-level grouping, for example Division |
| `[Detail Category]` | Third-level grouping, for example Service Line |
| `[Account]` | Account/customer/affiliation name |
| `[Opportunity Name]` | Optional opportunity-level detail |
| `[Service Start Date]` | Start date tied to the won record |
| `[Won Amount Row]` | Row-level amount contributing to won revenue/JSC |
| `[Is Won]` | Boolean flag for closed/won records |
| `[In Selected Period]` | Boolean flag for records inside the selected date period |

## Base measure
Use a row-level measure first, then aggregate it in the view.

```tableau
// Won Amount - Selected Period Row
IF [Is Won]
AND [In Selected Period]
THEN ZN([Won Amount Row])
END
```

In the worksheet, use:

```text
Columns: SUM([Won Amount - Selected Period Row])
```

Add a filter to remove non-contributing rows:

```text
SUM([Won Amount - Selected Period Row]) > 0
```

## Recommended scalable design
The most stable scalable version is not a single self-expanding worksheet. Use either:

1. A summary worksheet with `Top Category → Subcategory → Detail Category`, plus a separate Account Detail worksheet filtered by dashboard actions; or
2. A single worksheet with carefully separated action-key fields and sets.

The single-sheet version is more fragile because all dashboard actions can fire from the same click.

---

# Option A: Recommended linked-sheet design

## Summary sheet
Rows:

```text
[Top Category]
[Subcategory]
[Detail Category]
```

Columns:

```text
SUM([Won Amount - Selected Period Row])
```

Filters:

```text
[Is Won] = True
[In Selected Period] = True
SUM([Won Amount - Selected Period Row]) > 0
```

## Account detail sheet
Rows/Text:

```text
[Account]
[Opportunity Name]
[Detail Category]
[Service Start Date]
```

Measure:

```text
SUM([Won Amount - Selected Period Row])
```

Filters:

```text
[Is Won] = True
[In Selected Period] = True
SUM([Won Amount - Selected Period Row]) > 0
```

## Dashboard action
Use a normal Filter Action:

```text
Source: Summary sheet
Target: Account detail sheet
Run action on: Select
Clearing selection: Show all values or Exclude all values, depending on UX preference
Selected Fields: [Top Category], [Subcategory], [Detail Category]
```

This is the recommended approach if the dashboard must be stable and executive-friendly.

---

# Option B: Single-sheet set-action drill skeleton

## Create action-key fields
These are duplicate fields used only for set actions, so the visible drill fields do not conflict with the fields driving the actions.

```tableau
// Top Category Action Key
[Top Category]
```

```tableau
// Subcategory Action Key
[Subcategory]
```

```tableau
// Detail Category Action Key
[Detail Category]
```

## Create sets from action-key fields
Create these sets:

```text
[Top Category Action Set] from [Top Category Action Key]
[Subcategory Action Set] from [Subcategory Action Key]
[Detail Category Action Set] from [Detail Category Action Key]
```

## Drill calculated fields

### Level 1: show Subcategory only after Top Category is selected

```tableau
// Drill Level 1 - Subcategory
IF [Top Category Action Set] THEN
    IFNULL([Subcategory], "Unassigned Subcategory")
END
```

### Level 2: show Detail Category only after Subcategory is selected

```tableau
// Drill Level 2 - Detail Category
IF [Subcategory Action Set]
AND NOT ISNULL([Drill Level 1 - Subcategory])
THEN
    IFNULL([Detail Category], "Unassigned Detail Category")
END
```

### Level 3: show Account Detail only after Detail Category is selected

```tableau
// Drill Level 3 - Account Detail
IF [Detail Category Action Set]
AND NOT ISNULL([Drill Level 2 - Detail Category])
AND [Is Won]
AND [In Selected Period]
AND NOT ISNULL([Won Amount - Selected Period Row])
THEN
    IFNULL([Account], "Unassigned Account")
    + " | "
    + STR(DATE([Service Start Date]))
END
```

Do not include `SUM([Won Amount - Selected Period Row])` inside this dimension calc. That causes an aggregate/non-aggregate error. Put the amount on Columns, Label, and Tooltip instead.

## Worksheet setup
Rows:

```text
[Top Category]
[Drill Level 1 - Subcategory]
[Drill Level 2 - Detail Category]
[Drill Level 3 - Account Detail]
```

Columns:

```text
SUM([Won Amount - Selected Period Row])
```

Marks:

```text
Mark type: Bar
Label: SUM([Won Amount - Selected Period Row])
Tooltip: Account, Service Start Date, SUM([Won Amount - Selected Period Row])
Detail: [Top Category Action Key], [Subcategory Action Key], [Detail Category Action Key]
```

Filters:

```text
[Is Won] = True
[In Selected Period] = True
SUM([Won Amount - Selected Period Row]) > 0
```

## Dashboard set actions
Action 1:

```text
Name: Select Top Category
Source Sheet: Drill worksheet
Run action on: Select
Target Set: [Top Category Action Set]
Running action: Assign values to set
Clearing selection: Remove all values from set
```

Action 2:

```text
Name: Select Subcategory
Source Sheet: Drill worksheet
Run action on: Menu or Select
Target Set: [Subcategory Action Set]
Running action: Assign values to set
Clearing selection: Remove all values from set
```

Action 3:

```text
Name: Show Account Details
Source Sheet: Drill worksheet
Run action on: Menu
Target Set: [Detail Category Action Set]
Running action: Assign values to set
Clearing selection: Remove all values from set
```

## Important known pitfalls
- If the worksheet works but the dashboard does not, test with a brand-new dashboard containing only the worksheet and one action.
- Dashboard actions generally need a real selectable mark, not just a row header.
- If multiple set actions are set to `Select` on the same sheet, they may all fire on the same click and open levels too early.
- If a lower level opens too early, change that lower-level action to `Menu`.
- Avoid using the exact same field as the visible drill dimension and the action target set field. Use duplicated action-key fields.
- Keep amount as a measure on Columns/Label/Tooltip, not inside text-dimension calculations.
- Final account/detail rows must be gated by `[Is Won]`, `[In Selected Period]`, and non-null/positive won amount.

## Tooltip template
Use generic labels only:

```text
┌────────────────────────────┬────────────────────────────────────┐
│ Field                      │ Value                              │
├────────────────────────────┼────────────────────────────────────┤
│ Top Category                │ <Top Category>                     │
│ Subcategory                 │ <Drill Level 1 - Subcategory>      │
│ Detail Category             │ <Drill Level 2 - Detail Category>  │
│ Account Detail              │ <Drill Level 3 - Account Detail>   │
│ Current Won Amount          │ <SUM(Won Amount - Selected Period Row)> │
└────────────────────────────┴────────────────────────────────────┘
```

## Specific ask for Claude
Please diagnose why dashboard set actions may not fire even when worksheet interactions appear to work. Recommend the most stable implementation for a Tableau dashboard widget that drills from Top Category to Subcategory to Detail Category to Account Detail, while only showing account records that contributed to the selected-period won amount.
