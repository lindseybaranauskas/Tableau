# Command_View_v2.twbx — build notes (for the Tableau developer)

A **scaffold**, not a finished dashboard. It opens against a bundled mock CSV that uses
your real Salesforce field names, so when you repoint at the live SF connection the
calcs should carry over with minimal renaming. I can't run Tableau here, so expect to
re-drag a field or two and re-wire actions on first open.

## What's inside
- **Data:** `sf_opportunities.csv` (1,850 mock rows) bundled in. Columns use SF API names:
  `Id, Account_Name, HHS_Region__c, Division__c, Service_Line__c, StageName, IsWon, IsClosed,
  Amount, Probability, Annual_Service_Revenue__c, ExpectedRevenue, Owner_Name, CreatedDate,
  CloseDate, Service_Start_Date__c, LastStageChangeInDays, PushCount, SSD_Change_1/2/3__c`.
- **12 parameters:** `pTimeView` (Selected Period / Date Range / YTD / T12M), `pPeriodType`
  (Year/Quarter/Month), `pYear, pQuarter, pMonth, pStartDate, pEndDate, pRegionView, pPipelineView`
  (Weighted/Open), `pJSCGroup` (Region/Division), `pCoverageGoal`, `pIncludeMissingSSD`.
- **16 calculated fields** (real formulas, the substance to reuse):
  `In Selected Period` (Close-date based), `JSC In Selected Period` (SSD-based — JSC is dated by
  Service Start Date, matching the source), `In Selected Region`, `Won/Lost Flag`, `Amount Won`,
  `JSC Won`, `Open Amount`, `Weighted Amount`, `Pipeline Dollars` (toggle), `Win Rate`,
  `Is Stalled` (>60d), `Is Pushed`, `JSC Group Dimension`, `Coverage Ratio`, `Period Label`.
- **11 worksheets:** 5 KPI BANs (Amount Won, JSC Won, Win Rate, Pipeline, Coverage) + JSC Won by
  Group, Pipeline by Stage, Win Rate by Division, Stalled by Quarter, Pushed Deals, Rep Performance.
- **1 dashboard** ("Command View"), performance-left / opportunity-right.

## What it deliberately does NOT do
- The prototype's custom interactions (nested branch-tree expand, pop-out master/detail pipeline,
  inline drawers) are not natively expressible — reproduce the *intent* with set/parameter actions
  in Desktop. Worksheets here are simple bars/BANs to refine.
- No East-vs-West split (Tableau single-value param can't do two-region side-by-side cleanly —
  consider a duplicate set of sheets or a second dashboard).
- Targets/coverage use `pCoverageGoal` (a single parameter) as a placeholder — there is no
  company/division target field in Salesforce yet (see the tech spec).

## Likely first-open fixups
- A worksheet may show a field error if a mark encoding detail is off — drag the named measure
  to the indicated shelf; all calcs exist in the data pane.
- Confirm Tableau reads `IsWon`/`IsClosed` as Boolean and the date columns as dates.
- Swap the CSV connection for the live Salesforce connection; map calc fields onto the real
  `Opportunity`/`Account`/`OpportunityFieldHistory`/`OpportunityStage` model.
- `JSC In Selected Period` uses `Service_Start_Date__c`; honor `pIncludeMissingSSD` where SSD is null.

This file is meant to be handed to an assistant or finished directly in Tableau — the parameters
and calculated-field library are the reusable core.
