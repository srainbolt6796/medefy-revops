---
name: "deal-handoff"
description: "Generate a Medefy-branded deal handoff as a portrait PDF (+ HTML) from HubSpot activity and Gong calls — for a single deal, or one PDF per deal for all deals owned by a rep within a time period. Use for rep-to-rep handoffs, owner changes, or closed-lost win-back prep."
---

# Deal Handoff

Generate a **Medefy-branded handoff** (portrait PDF + HTML) from HubSpot activity and Gong calls, so a new owner can pick up a deal fast. Runs for a single deal, or for all of a rep's deals in a time period (one PDF per deal by default).

## When to use
Rep-to-rep handoff on owner change, a departing rep's book, or closed-lost win-back prep. Triggers: "handoff notes," "deal handoff," "handoff for [deal]," "handoff all of [rep]'s deals."

## Inputs
- **Single deal:** deal name, HubSpot deal ID, or deal URL.
- **Batch:** a deal owner (rep) + time window (e.g., "Q2 2026"). Default output = **one PDF per deal**; make a single combined PDF only if asked. List the deal set and confirm before generating many.

## Key HubSpot facts (portal 8313828)
- Deal record URL: `https://app.hubspot.com/contacts/8313828/record/0-3/{dealId}`.
- **Gong recordings sync as MEETING engagements** — Gong meetings have `hs_meeting_title` prefixed `[Gong]` and the call summary + recording link inside `hs_meeting_body`. Always pull MEETINGs for Gong content; extract the `gong.io/call?id=...` link.
- Activity-type prefixes are in `hs_activity_type` (e.g., "Broker: Meeting", "Client: Call") — use them to label the timeline.
- **Owners:** resolve `hubspot_owner_id` via `search_owners`; flag **inactive** owners — a departed rep is usually the handoff reason. Capture the transition (prior → current owner).
- **Stage label:** resolve `dealstage` id via `get_properties` DEAL `[dealstage]` (labels are pipeline-specific; e.g., 147534882 = "Demo").
- **Closed-lost:** `closed_lost_category` (Category — priority), `closed_lost_reason_s_` (Reason(s) — fallback), `closed_lost_reason` (Reason Notes — always include). Flags: `hs_is_closed_won`, `hs_is_closed_lost`.
- **Deal fields:** dealname, dealstage, pipeline, dealtype, amount, hs_arr, hs_mrr, createdate, closedate, hubspot_owner_id, broker_organization, broker_name, level_of_broker_involvement, number_of_associated_broker_contacts.

## Data-integrity principles
- Include contaminated/mislabeled deals with honest notes — never silently drop one.
- Flag: activity cross-associated from unrelated accounts; closed-lost vs. active discrepancies; CRM status that disagrees with recent email/meeting activity.
- A referral/relationship counts regardless of a stale record.

## Procedure
1. **Resolve the deal(s).** Single: `search_crm_objects`/`get_crm_objects` DEAL by id/name (parse id from a URL). Batch: `search_owners` (name → ownerId) → DEAL where `hubspot_owner_id`=ownerId + date window (closed-lost win-back → `closedate BETWEEN`; active handoff → open deals). Confirm the set.
2. **Gather per deal:** deal properties (resolve stage + pipeline labels); owner name + active/inactive (+ prior owner if transitioned); associated **company** (name, industry, u_s_state, contract_status, lifecyclestage) and **contacts** (name, jobtitle, email, broker_program_status) via `associatedWith`; **activities** via `associatedWith` deals — MEETING (incl. Gong), NOTE, EMAIL, CALL, TASK. For Gong meetings pull `hs_meeting_title/body/start_time`. Emails can be huge — take subjects/dates/direction, NOT full bodies (search may exceed token limits; if so, read the saved file for subjects only). Sort activities chronologically.
3. **Synthesize:** a plain-English status narrative; champion / gatekeeper / broker / internal roles; risks & open items; recommended next steps. Closed-lost → loss reason (Category → Reason(s) → Notes) + a realistic win-back angle.
4. **Build the branded HTML** using the **`medefy-brand`** skill (dark header banner + reversed logo base64, cyan eyebrow, white title, pink subtitle; page tinted `#F4F6F9`; white cards with soft shadow). **Section titles: 16px, ink `#202035`, bold, uppercase, with a 2px `#E9EBEE` underline divider** separating title from content. Sections, in order:
   - **Snapshot cards (4):** Stage · Annual Value ($ARR / $MRR) · Target Close (or Close date) · Owner (note "prev. {prior owner}" if transitioned).
   - **Deal Snapshot** (key-value table): company, industry, location, population/size, funding, lifecycle, consultant/broker, created.
   - **Key Contacts & Roles:** champion highlighted with a pink left-border card; gatekeeper/consultant; broker; internal team (current owner + prior owner flagged as departed). Role chips.
   - **Where It Stands:** the status narrative.
   - **Gong Call Summaries:** one card per Gong meeting — date · duration · attendees, a condensed brief + key points, next steps, and an "Open Gong recording →" link (cyan left-border card). Note summaries are Gong-generated.
   - **Activity Timeline:** chronological table (When · Type w/ `hs_activity_type` · Summary).
   - **Risks & Open Items** and **Recommended Next Steps** (two cards). Closed-lost → include loss reason + win-back angle.
   Add a "Save as PDF" button (`window.print()`, hidden via `@media print`). **Print CSS = A4 PORTRAIT**, `print-color-adjust:exact`, tint `#F4F6F9` on BOTH `@page` and `html,body`, keep card shadows; reflow the snapshot cards to 2-up and stack any two-column rows.
5. **Render & deliver:** generate the PDF with **weasyprint** (`pip install weasyprint --break-system-packages` if needed; `weasyprint.HTML(html_path).write_pdf(pdf_path)`; `pdftoppm -png` to verify portrait/tinted). Name `Deal Handoff - {Deal Name}.pdf`. **Present BOTH the PDF and the HTML** (the in-app viewer sometimes blocks the print button / PDF preview, so always ship the file). Batch → one PDF per deal, present all with a one-line count.

## Notes
- Gong summaries are auto-generated by Gong — attribute them.
- Watch email payload size; itemize subjects/dates, not bodies.
- Keep the honest data-quality flags visible in Risks & Open Items.

