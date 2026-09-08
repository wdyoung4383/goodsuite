# GoodSuite License Reconciliation App: Status and Roadmap

Prepared 2026-09-08. Target: functional by Friday 2026-09-12.

## References

- Rewst app: https://app.rewst.ai/orgs/goodsuite/apps/1693bca0-5469-4a87-81c3-28d4ae93b552
- Rewst App Builder docs: https://docs.rewst.help/documentation/app-builder/getting-started
- Rewst App Builder components (Data Table load methods): https://docs.rewst.help/documentation/app-builder/components
- Rewst new-platform announcement (Agent, new App Builder, MCP server, beta status): https://docs.rewst.help/flow-2026-announcement/coming-soon-to-rewst
- Will/Amy 2026-08-12 (PAX8 reconciliation demo, Amy as future owner): https://app.fireflies.ai/view/01KZSE5W61XBQXXY6K1B6P7XDZ?t=938
- L10 2026-07-13 (Thomas/Brent requirements for reconciliation): https://app.fireflies.ai/view/01KX9YTVVB3JA4C7BPFHYQ0S8W?t=3222
- Rewst check-in 2026-08-24 (new platform GA late Sep/early Oct, pricing TBD): https://app.fireflies.ai/view/01M06GDH970VV5209PEW1G73XP
- L10 2026-07-27 (seven-month Azure billing sync miss, Cloud Olive evaluation): https://app.fireflies.ai/view/01KXYDQ3J4Q1FKCWPC3RSFSBJH

## What this app is

PAX8 subscription reconciliation against ConnectWise agreement additions, built on Rewst's new agent-driven App Builder (beta). Pull PAX8 subscription counts per client, compare to the quantity on the matching CW addition, show deltas, and flag anything that does not tie back to an agreement. AI-assisted mapping suggests which PAX8 product corresponds to which CW product or addition. Amy Beaver is the intended day-to-day owner once built. Stated end state is "all tools" (PAX8, Datto, Blackpoint, Acronis, and so on); PAX8 is the first and only integration in scope right now.

## Current status

What I could verify:

- This repo contains only the static Agreement Gross Profit report (`index.html`, Jan to Mar 2026 data). No app code, workflows, or notes for the Rewst app live here. Last commit was 2026-03-31.
- The app exists on the new Rewst platform, which is still beta. Rewst told you on 2026-08-24 that GA is late September or early October with pricing announced then.
- Last hands-on work I can find is the 2026-08-12 session with Amy, where the app was a work in progress ("this is what I'm working on now").
- Amy owes you a list of tools to integrate (action item from 2026-08-12). I found no evidence it was delivered. [unverified]
- Business pressure is real: Brent cited a seven-month Azure billing sync failure and a $1,700/month Datto line item nobody could attribute to a client. Thomas has put a Cloud Olive evaluation in motion as a paid alternative. This app is your answer to "we don't need a $1,000/month tool."

What I could not verify:

- Anything inside the Rewst app itself. The app URL is login-gated and no Rewst connector is attached to this session. I cannot see the pages, workflows, or run history. Everything under "Likely causes" below is inference from the Rewst docs and how these apps are normally wired, and is tagged accordingly.

To go further I need either the Rewst MCP server connected (Rewst now ships one for authenticated tenant access, per the announcement page above), or an export of the app's workflows plus a screenshot of the home page in edit mode and one failed run of the mapping workflow.

## Likely causes of the two symptoms

### Blank landing page

Ranked by how often this is the cause in Rewst apps. [inferred]

1. The home page's Data Table or Chart is on "Use Latest Workflow" and that workflow has never completed successfully in the GoodSuite org context, so there is nothing to render. Switch to "Run Workflow on Load" or run the backing workflow once manually and reload.
2. The app was edited but not published, or the page you are viewing is the auto-generated default home page rather than the one you built. Only one home page is allowed per app; confirm which page is flagged as home.
3. The backing workflow errors before it publishes its output variable (PAX8 or ConnectWise integration auth expired, or a Jinja error in the transform). A failed workflow renders as an empty table with no error surfaced on the page.
4. Access control on the page or app is set to owner-only or a suborg you are not viewing as. This renders blank rather than denying access in some cases.
5. Beta platform regression. The new App Builder was mid-rewrite in late August. If the page was built on a component that changed, it may need to be re-added. Check the Rewst Discord beta channel and your Rewst rep before spending hours here.

Fifteen-minute triage: open the app in edit mode, open the home page, click each data component, note its load method and bound workflow, then open that workflow's results and read the last run. That tells you which of 1 through 5 it is.

### AI mapping not working

[inferred] The mapping step is a workflow that hands the PAX8 product list and the CW product catalog to an LLM action and asks for matches. Typical failure modes, in order of likelihood:

1. Payload too large. GoodSuite's CW product catalog plus the full PAX8 subscription list will blow past the context or output limits of a single LLM call, and the action returns truncated or invalid JSON. Fix is to batch by client and only send unmatched rows.
2. Output parsing. The LLM returns prose or fenced JSON and the downstream Jinja `from_json` or equivalent fails silently. Force a strict JSON schema in the prompt and validate before use.
3. The LLM integration itself: expired key, wrong model name, or the action moved between old-platform and new-platform action names.
4. Workflow timeout. Long LLM calls in a synchronous "Run Workflow on Load" path will time out and the page shows nothing.

Whichever it is, my recommendation below is to take AI off the critical path this week.

## Recommendation: scope for this week

Simplicity wins here. Get a boring, deterministic PAX8 reconciliation in front of Amy by Friday, and treat AI mapping as an assist for leftovers, not the engine.

**Cut to an MVP:**

- PAX8 only. No Datto, Blackpoint, or Acronis until PAX8 is in weekly use.
- Deterministic matching first: exact match on a persisted mapping table (PAX8 product ID to CW product ID), then exact name match, then vendor SKU match where PAX8 exposes one. Most of the catalog will map this way and never needs AI.
- Persist mappings in a Rewst table (or the app's data layer), keyed by PAX8 product ID. Do not store them on the CW addition record; a separate table survives platform changes and lets Amy edit without touching CW.
- AI only runs on the unmatched remainder, per client, in small batches, and writes suggestions to a "needs review" list. Amy confirms or rejects. A confirmed suggestion becomes a row in the mapping table. Nothing auto-applies.
- Output is a single delta table: client, CW agreement, CW addition, CW quantity, PAX8 quantity, delta, status (matched, unmapped PAX8 item, unmapped CW addition). Unmapped rows are the "does not tie back to an agreement" cases Brent asked for.
- Read-only against ConnectWise this week. Writing quantity updates back is phase 2 and needs a confirm step regardless.

What this gives up: nothing Thomas or Brent asked for. They asked for deltas and orphan detection. AI-first mapping was your optimization, not their requirement.

## Build order

Dates assume you start Tuesday 2026-09-09.

| # | Day | Work | Done when |
|---|-----|------|-----------|
| 01 | Tue AM | Triage the blank page using the checklist above. Fix or rebuild the home page with a single Data Table on "Run Workflow on Load" bound to a stub workflow that returns three hard-coded rows. | Home page renders rows. |
| 02 | Tue PM | Workflow A: pull PAX8 subscriptions for one client (Schifrin, since Amy already chose it for onboarding). Publish as a flat list: pax8_company_id, product_id, product_name, sku, quantity. | Workflow run shows the list. |
| 03 | Wed AM | Workflow B: pull CW agreements and additions for the same company. Flatten to: company_id, agreement_id, agreement_name, addition_id, product_id, product_identifier, quantity, cancelled_flag. Filter out cancelled additions. | Workflow run shows the list. |
| 04 | Wed PM | Mapping table and Workflow C: deterministic match (mapping table, then exact name, then SKU). Emit the delta table. Bind it to the home page. | Home page shows real Schifrin deltas. |
| 05 | Thu AM | Client dropdown on the home page that passes company into Workflows A through C. Then run it for all clients in one pass and check the unmapped count. | Any client selectable; unmapped list visible. |
| 06 | Thu PM | AI assist for unmapped rows only: batch of at most 25 rows per call, strict JSON output, validate, write to a review table. Review page with confirm/reject that upserts the mapping table. | Amy can confirm a suggestion and see it disappear from unmapped. |
| 07 | Fri AM | Walk Amy through it live. Have her map ten items herself. Note every place she hesitates; that is the fix list. | Amy runs it unassisted. |
| 08 | Fri PM | Schedule a weekly run and a Teams or email summary (total delta dollars, unmapped count). Commit workflow exports and a one-page runbook to this repo. | Runbook in repo; schedule on. |

If day 01 turns into a platform-bug rabbit hole, stop at noon, build the same MVP on the classic App Builder (which is stable and documented), and revisit the new platform after GA. Losing a morning is acceptable; losing the week to beta software is not.

## Risks and pushback

- **Beta platform under a hard deadline.** You committed a "this week" delivery on software that is pre-GA with pricing unknown. Keep every workflow independent of the App Builder so the data layer survives if the app layer breaks or repricing forces a move. The app should be a thin view over workflow outputs.
- **AI mapping is the shiny part and the least reliable part.** Deterministic matching plus a human review loop will map 80 to 90 percent of PAX8 within a session and never regress. [inferred from typical PAX8 to CW catalog overlap; verify on Schifrin's data on Wednesday]
- **Ownership.** Amy asked on 2026-09-01 who owns agreement data accuracy and you pointed to Thomas and Bing. The reconciliation tool will surface agreement errors immediately. Decide before Friday who fixes what it finds, or it becomes a report nobody acts on. Suggest: Amy runs it and maps products; Thomas owns agreement corrections; Brent gets the weekly delta summary.
- **Cloud Olive.** Thomas has a vendor evaluation running for the same problem. Showing a working PAX8 delta table on Friday is what keeps that from becoming a purchase. Book ten minutes on the next L10 for a demo.
- **Repo hygiene.** Nothing about this app is versioned anywhere. Rewst told you on 2026-08-28 to use GitHub for workflow version control. Export the workflows into this repo as part of day 08 so the next "pick this back up" does not start from zero.

## Phase 2 (after PAX8 is in weekly use)

1. Write-back: propose CW addition quantity updates with an explicit confirm step, never automatic.
2. Second integration: pick whichever tool Amy's list ranks highest by dollar exposure. Datto is the obvious candidate given the $1,700/month orphan.
3. Dollarize deltas using CW unit price so the weekly summary reads as missed revenue, not unit counts.
4. Fold the delta summary into the Agreement Gross Profit report in this repo so leadership sees billing leakage next to margin.
