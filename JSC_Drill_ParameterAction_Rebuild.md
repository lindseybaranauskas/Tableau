# JSC Won Drill — Parameter-Action Rebuild (Stable Single-Sheet)

## Verdict (read first)
The current design uses **chained Set Actions from one source worksheet**. That is the cause of the
"clicking Region opens too much / Division set too broad" behavior — **sets hold multiple values and
every action fires from the same click**. This cannot be made cleanly two-click.

**Fix:** keep the single worksheet, but drive the drill with **Parameter Actions** instead of Set
Actions. A parameter holds exactly ONE value, so "one Region → one Division" is enforced by design.
The account-level calculation that already works is kept as-is.

Do NOT hand-edit workbook XML. Build this in Tableau by creating fields, parameters, and actions.

---

## 1. Parameters

Add two string parameters (reuse the optional ones already in the workbook if present):

```text
pSelectedRegion    (string, default "")
pSelectedDivision  (string, default "")
```

Keep existing: `pTime View`, `pSelected Year`, `pSelected Quarter`, `pSelected Month`, `pRegion View`.

---

## 2. Helper key fields (so parameter actions pass clean single values)

```tableau
// Region Key
IFNULL([Region], "Unassigned Region")
```
```tableau
// Division Key
IFNULL([Division], "Unassigned Division")
```

These are the fields the parameter actions will read. Because each drill level aggregates to one
value per mark, `ATTR()` resolves to a single string at the level you click.

---

## 3. Period + base measure (unchanged logic, keep what works)

```tableau
// In Selected Period   (JSC dated by Service Start Date)
[Service Start Date] >= [Selected Period Start]
AND [Service Start Date] <= [Selected Period End]
```
```tableau
// JSC Won Current Row
IF ([pRegion View] = "All" OR [Region] = [pRegion View])
   AND [Is Won]
   AND [In Selected Period]
THEN [Amount]
END
```
```tableau
// JSC Won
SUM([JSC Won Current Row])
```

---

## 4. Drill display fields — now driven by PARAMETERS, not sets

Replace the set-based display calcs with these. Structure and the working account logic are preserved;
only the gating changes from `[... Set]` to parameter comparisons.

```tableau
// Division Display
// After a Region is selected, show every Division in that Region (so the user can click one).
IF [pSelectedRegion] <> "" AND [Region Key] = [pSelectedRegion]
THEN [Division Key]
ELSE ""
END
```
```tableau
// Service Line Display
// Only after a single Division is selected, within the selected Region, for contributing won rows.
IF [pSelectedRegion]   <> "" AND [Region Key]   = [pSelectedRegion]
   AND [pSelectedDivision] <> "" AND [Division Key] = [pSelectedDivision]
   AND [Is Won] AND [In Selected Period]
   AND ZN([JSC Won Current Row]) > 0
THEN IFNULL([Service Line], "Unassigned Service Line")
ELSE ""
END
```
```tableau
// Account / Opportunity Display
// Shows all won account/opportunity rows for the selected Division (does NOT depend on a Service Line set).
IF [pSelectedRegion]   <> "" AND [Region Key]   = [pSelectedRegion]
   AND [pSelectedDivision] <> "" AND [Division Key] = [pSelectedDivision]
   AND [Is Won] AND [In Selected Period]
   AND ZN([JSC Won Current Row]) > 0
THEN IFNULL([Opportunity Name], "Unassigned Opportunity") + " | " + STR(DATE([Service Start Date]))
ELSE ""
END
```

---

## 5. Worksheet — "JSC Won Region View Drill Down"

```text
Rows (in order):
   1. Region
   2. Division Display
   3. Service Line Display
   4. Account / Opportunity Display
Columns:
   SUM([JSC Won Current Row])           (i.e. the JSC Won measure)
Marks:
   Bar
   Label:  JSC Won
   Color:  Region   (or neutral)
Filters:
   Keep only broad business filters (e.g., record type).
   DO NOT filter on Division Display / Service Line Display / Account Display.
   REMOVE any generated "Exclusions (...)" filter on the drill-display fields — blank lower
   levels represent collapsed branches and must remain.
```

Blank handling: the `ELSE ""` rows render as empty sub-labels for collapsed branches. Do not add a
`<> ""` worksheet filter on the display fields — that would collapse parent branches too. If stray
"Null"/blank headers look noisy, turn off the null indicator (right-click axis → Hide) rather than
filtering.

---

## 6. Parameter actions (the whole fix)

Two actions, both fired from the same sheet — safe now because each writes ONE value to ONE parameter.

```text
Action 1 — Select Region
   Dashboard > Actions > Add > Change Parameter
   Source Sheet:   JSC Won Region View Drill Down
   Run action on:  Select
   Target Parameter: pSelectedRegion
   Source Field:     Region Key
   Aggregation:      None (or ATTR)

Action 2 — Select Division
   Dashboard > Actions > Add > Change Parameter
   Source Sheet:   JSC Won Region View Drill Down
   Run action on:  Select
   Target Parameter: pSelectedDivision
   Source Field:     Division Key
   Aggregation:      None (or ATTR)
```

**Why this gives clean two-click behavior:**
- Click a collapsed **Region** bar → `Region Key` is one value (set), `Division Key` aggregates to many
  → resolves to `*`/blank → `pSelectedDivision` does not match any division → only Divisions show.
- Click a **Division** bar → `Division Key` is one value → `pSelectedDivision` set → Service Lines +
  Accounts show for that Division only.
- Click a different Region → `pSelectedRegion` updates, `pSelectedDivision` resets to `*`/blank → that
  Region re-expands fresh. No cascade, no stuck multi-Division state.

---

## 7. Reset control (parameters don't auto-clear)

Parameter actions don't clear on deselect, so add a reset:
```text
Option 1: Show pSelectedRegion and pSelectedDivision as parameter controls; user blanks them.
Option 2: A small floating "Reset" sheet (one dummy mark) with two Change-Parameter actions
          (Run on: Select) that set pSelectedRegion = "" and pSelectedDivision = "".
```

---

## 8. If you want ZERO fragility (alternatives to the in-place click drill)
The in-place "click a bar to expand on the same sheet" pattern is the hardest in Tableau. If executive
stability matters more than expanding within one sheet, either is bulletproof:
- **Native hierarchy:** make Region→Division→Service Line a hierarchy; users expand with the +/-
  control. No actions at all. Add one Filter Action to a separate Account list.
- **Two sheets + one Filter Action:** summary (Region→Division→Service Line) → Account detail sheet.

Both behave identically on worksheet and dashboard. The parameter-action version above is the answer
if you specifically want the single-sheet in-place drill.

---

## 9. Test procedure
1. Build the worksheet; confirm clicking Region then Division behaves on the **worksheet**.
2. Put ONLY this sheet on a **blank** dashboard with the two parameter actions; test two-click.
3. If it works on the blank dashboard but not the real one → a **floating object is intercepting the
   click** (check the Layout pane for anything stacked over the sheet). This is the most common
   "works on sheet, dead on dashboard" cause.
4. Confirm no leftover **Set Actions** remain on the sheet — delete them so they don't fire alongside
   the parameter actions.

---

## 10. Specific ask for the assistant
Rebuild the single-sheet JSC drill using **Change-Parameter actions** (pSelectedRegion,
pSelectedDivision) per Sections 1–7, replacing all Set Actions. Keep the existing account-level
calculation. Provide the calculated fields, the exact rows/marks/filter setup, both parameter-action
configurations, and a reset control. Then give the Section 9 test steps, leading with the floating-
overlay check. Do not edit workbook XML; produce field formulas and build instructions only.
