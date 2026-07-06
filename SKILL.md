---
name: cap-table-monthly-refresh
description: Refresh the Spot AI cap table using Carta as the source of truth. Runs automatically on the 3rd business day of each month, and also supports forced/manual refreshes on demand.
---

You are running the Spot AI cap table refresh. This task runs in one of two modes:

- **Scheduled mode** — the task is scheduled to fire on every weekday between day 3 and day 7 of the month. In this mode it is YOUR job to short-circuit if today is not the 3rd business day of the current month.
- **Forced mode** — a manual, on-demand run requested directly by the user (e.g. "run the refresh", "force refresh the cap table", "run it now", "rebuild the cap table", "refresh now"). In this mode you skip the business-day gate entirely and always proceed.

## Step 0: Determine run mode

Treat the run as **forced** if ANY of the following is true:
- The user is present and has explicitly asked for the refresh to run now (a manual chat invocation rather than an automated/scheduled trigger).
- The request contains a force/now intent — e.g. "force", "run it now", "run now", "refresh now", "rebuild now", "do it now", "push it again", "re-run".
- The user names a specific month or as-of date to (re)build (e.g. "rebuild the May cap table").

Otherwise treat the run as **scheduled** (e.g. an automated run delivered with a scheduled-task wrapper and no human present).

- If **forced** → skip Step 1 and go straight to Step 2. Note in your final confirmation that this was a forced/manual run.
- If **scheduled** → proceed to Step 1.

## Step 1: Determine whether to proceed (scheduled mode only)

Skip this step entirely on a forced run.

Compute whether today is the 3rd US business day of the current month. Treat Saturdays, Sundays, and US federal holidays (New Year's Day, MLK Day, Presidents' Day, Memorial Day, Juneteenth, Independence Day, Labor Day, Columbus Day, Veterans Day, Thanksgiving Day, Christmas Day — observed when the date falls on a weekend) as non-business days. Iterate calendar day 1, 2, 3... of the current month and count weekdays-that-are-not-holidays; the date where that count first equals 3 is the target.

If today is NOT that date, stop immediately and do nothing else. Do not post to Slack and do not generate files.

If today IS the 3rd business day, proceed.

## Step 2: Pull data from Carta

Carta MCP corporation_id for Spot AI, Inc. is **572610**.

Compute the "as-of" date as the last calendar day of the PREVIOUS month (e.g. if today is in June, as-of = May 31 of the current year). On a forced run, if the user explicitly names a different month or as-of date, use that instead; otherwise default to the last day of the prior month.

Call the Carta MCP tools:
1. `mcp__864fa754-55cc-45dd-8fe2-1804ab1bd1df__welcome` (required initialization)
2. `cap_table:get:cap_table_by_share_class` with corporation_id=572610 and as_of_date=<last day of prior month>
3. `cap_table:get:cap_table_by_stakeholder` with the same params plus page_size=500, detail=full

## Step 3: Update the workbook

The cap table lives at `G:\Shared drives\Financing (Equity and Debt Sensitive)\Cap Table\`. The newest file is named `SpotAI Cap Table YYYY-MM-DD.xlsx` (where the date is the as-of-date of the most recent refresh).

If Cowork has access to that folder, open the latest workbook and ADD three new tabs to it (do not overwrite the existing tabs):
- `Major Investors - <Month>` (e.g. "Major Investors - May")
- `Simple Cap Table - <Month>`
- `Detailed Cap - <Month>`

If a tab with the target month's name already exists (e.g. on a forced re-run of the same month), overwrite/replace that month's three tabs rather than creating duplicates — but never touch tabs from other months.

Each new tab should mirror the structure of the corresponding April tabs from the prior month's workbook. Use the Carta data pulled in Step 2:
- Simple Cap Table: row per share class (Common, Series A-1, Series A, Series B, Series B-1, Series B-2), then Common Warrants, Options Outstanding, Options Available for Grant, Total.
- Major Investors: Founders section (Tanuj Thapliyal, Sudarshan Bhatija, Rishabh Gupta), Key Investors section (Redpoint, Scale, Bessemer, StepStone, Qualcomm, AT&T, GSBackers, Cheyenne, Pledge, Village Global, Ourcrowd), All Others, Total.
- Detailed Cap: full per-stakeholder rollup sorted by fully-diluted shares descending.

Save the workbook as `SpotAI Cap Table YYYY-MM-DD.xlsx` (date = as-of-date) in `G:\Shared drives\Financing (Equity and Debt Sensitive)\Cap Table\`. If that path is not accessible, save to the outputs folder and call this out clearly in the Slack message.

## Step 4: Post to Slack

Search for the channel `#fpa-model-updates` (channel ID `C09MT601JAU`), then call `slack_send_message_draft` (NOT slack_send_message — victor reviews before sending) with a message in this format:

```
**Cap Table — <Month> <Year> refresh**

The cap table has been updated with <Month> <last day>, <Year> data pulled from Carta. New tabs added to the workbook:
- Major Investors - <Month>
- Simple Cap Table - <Month>
- Detailed Cap - <Month>

**Key totals as of <as-of-date>**
| Metric | Prior month | This month | Δ |
|---|---:|---:|---:|
| Total outstanding | ... | ... | ... |
| Total fully diluted | ... | ... | ... |
| Options outstanding | ... | ... | ... |
| Options available for grant | ... | ... | ... |
| Total cash raised | — | $... | — |

<2–3 sentences of commentary on what moved this month>

File: `G:\Shared drives\Financing (Equity and Debt Sensitive)\Cap Table\SpotAI Cap Table YYYY-MM-DD.xlsx`
```

Compare the new month totals to the prior month from the workbook to compute the deltas. If a delta is zero, show "0". On a forced re-run of a month that was already refreshed, compare against the prior month as usual (not against the previous build of the same month).

## Step 5: Confirm

End the run by stating: "Cap table refresh complete for <Month> <Year>. Slack draft posted to #fpa-model-updates for review." On a forced run, append " (forced/manual run)." If you short-circuited in Step 1, state instead: "Today is not the 3rd business day of the month — no action taken."