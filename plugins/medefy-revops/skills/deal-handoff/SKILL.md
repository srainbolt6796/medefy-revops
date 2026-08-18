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
- **Closed-lost:** `closed_lost_category` (Category — priority), `closed_lost_reason_s_` (Reason(s) — fallback), `closed_lost_reason` (Reason Notes — always include). Flags: `hs_is_closed_won`, `hs_is_closed_lost`. Treat these fields as a **cross-check, not the source of truth** — they are frequently stale or mislabeled; the calls and emails are the primary evidence for the loss story.

## Data-integrity principles
- Include contaminated/mislabeled deals with honest notes — never silently drop one.
- Flag: activity cross-associated from unrelated accounts; closed-lost vs. active discrepancies; CRM status that disagrees with recent email/meeting activity.
- A referral/relationship counts regardless of a stale record.
- **For closed-lost deals, the loss story is reconstructed from Gong + emails and reconciled against the CRM loss reason — a mismatch is surfaced, not smoothed over.**

## Procedure
1. **Resolve the deal(s).** Single: `search_crm_objects`/`get_crm_objects` DEAL by id/name (parse id from a URL). Batch: `search_owners` (name → ownerId) → DEAL where `hubspot_owner_id`=ownerId + date window (closed-lost win-back → `closedate BETWEEN`; active handoff → open deals). Confirm the set.
2. **Gather per deal:** deal properties (resolve stage + pipeline labels); owner name + active/inactive (+ prior owner if transitioned); associated **company** (name, industry, u_s_state, contract_status, lifecyclestage) and **contacts** (name, jobtitle, email, broker_program_status) via `associatedWith`; **activities** via `associatedWith` deals — MEETING (incl. Gong), NOTE, EMAIL, CALL, TASK. For Gong meetings pull `hs_meeting_title/body/start_time`. Emails can be huge — take subjects/dates/direction, NOT full bodies (search may exceed token limits; if so, read the saved file for subjects only). Sort activities chronologically.
3. **Synthesize:** a plain-English status narrative; champion / gatekeeper / broker / internal roles; risks & open items; recommended next steps.
   - **Closed-lost → build the Closed-Lost Story from the evidence, not just the CRM field.** Read the Gong call summaries and the email thread and reconstruct *why the deal was lost*: the turning point (which call/email it went sideways), the specific objection or blocker (price, timing, competitor, no budget, champion left, lost to incumbent, went silent, etc.), **who drove the no**, and any quoted signal that supports it (cite the call date / email subject). Then **reconcile with the CRM loss fields** (`closed_lost_category` → `closed_lost_reason_s_` → `closed_lost_reason` notes): if the calls/emails tell a different story than the CRM category, say so plainly and flag the mismatch. Finish with a **realistic win-back angle grounded in that evidence** (what changed or would need to change to re-open). If there are no Gong calls or emails to draw on, state that the story is CRM-only and lower-confidence.
4. **Build the branded HTML** using the **`medefy-brand`** skill (dark header banner + reversed logo base64, cyan eyebrow, white title, pink subtitle; page tinted `#F4F6F9`; white cards with soft shadow). **Section titles: 16px, ink `#202035`, bold, uppercase, with a 2px `#E9EBEE` underline divider** separating title from content. Sections, in order:
   - **Snapshot cards (4):** Stage · Annual Value ($ARR / $MRR) · Target Close (or Close date) · Owner (note "prev. {prior owner}" if transitioned).
   - **Deal Snapshot** (key-value table): company, industry, location, population/size, funding, lifecycle, consultant/broker, created.
   - **Key Contacts & Roles:** champion highlighted with a pink left-border card; gatekeeper/consultant; broker; internal team (current owner + prior owner flagged as departed). Role chips.
   - **Where It Stands:** the status narrative.
   - **Closed-Lost Story** *(closed-lost deals only)* — a distinct card, placed right after "Where It Stands". Give the loss narrative reconstructed from the Gong calls + emails (turning point, objection/blocker, who drove the no, cited call date / email subject); then a **"CRM says"** line showing `closed_lost_category` / `Reason(s)` / Notes with a **⚠ mismatch flag** if the evidence disagrees; then a **Win-back angle** line grounded in the evidence. If no calls/emails exist, label it CRM-only / lower-confidence.
   - **Gong Call Summaries:** one card per Gong meeting — date · duration · attendees, a condensed brief + key points, next steps, and an "Open Gong recording →" link (cyan left-border card). Note summaries are Gong-generated.
   - **Activity Timeline:** chronological table (When · Type w/ `hs_activity_type` · Summary).
   - **Risks & Open Items** and **Recommended Next Steps** (two cards). Closed-lost → next steps = the win-back play from the Closed-Lost Story.
   Add a "Save as PDF" button (`window.print()`, hidden via `@media print`). **Print CSS = A4 PORTRAIT**, `print-color-adjust:exact`, tint `#F4F6F9` on BOTH `@page` and `html,body`, keep card shadows; reflow the snapshot cards to 2-up and stack any two-column rows.
5. **Render & deliver:** generate the PDF with **weasyprint** (`pip install weasyprint --break-system-packages` if needed; `weasyprint.HTML(html_path).write_pdf(pdf_path)`; `pdftoppm -png` to verify portrait/tinted). Name `Deal Handoff - {Deal Name}.pdf`. **Present BOTH the PDF and the HTML** (the in-app viewer sometimes blocks the print button / PDF preview, so always ship the file). Batch → one PDF per deal, present all with a one-line count.


## Render with the bundled builder (do NOT hand-write HTML/CSS)
The handoff's look is produced by a fixed Python builder — the model supplies only data, so the design can never drift.
1. Write the **Handoff builder** code block below **verbatim** to `build_handoff.py` (do not modify it — logo, CSS, and layout are baked in).
2. Write your assembled data (DATA SCHEMA below) to `data.json`.
3. Run: `pip install weasyprint --break-system-packages` (if needed), then `python3 build_handoff.py data.json "Deal Handoff - {Company}.html" "Deal Handoff - {Company}.pdf"`.
4. Verify with `pdftoppm -png` and **present BOTH** files. Batch → one JSON+PDF per deal.

## DATA SCHEMA (the JSON you pass to the builder)
```
{
  "eyebrow": "Deal Handoff · Closed-Lost Win-Back",   // or just "Deal Handoff" for active deals
  "company": "Eastern Shawnee Tribe of Oklahoma",
  "subtitle": "Closed Lost · Aug 13, 2026  •  Owner: {name}  •  Broker: {firm}",
  "deal_url": "https://app.hubspot.com/contacts/8313828/record/0-3/{dealId}",
  "snapshot": [ {"label":"Stage","value":"Closed Lost","lost":true}, {"label":"Annual Value","value":"$88,200 ARR"}, {"label":"Close Date","value":"Aug 13, 2026"}, {"label":"Owner","value":"{name}"} ],
  "deal_snapshot": [ ["Company","…"], ["Industry","…"], ["Location","…"], ["Deal value","…"], ["Lifecycle","…"], ["Broker / consultant","…"], ["Created","…"], ["Closed","…"] ],
  "contacts": [ {"name":"…","role":"…","note":"…","role_class":"buyer|broker|internal"}, … ],
  "status_narrative": "…HTML prose…",
  "closed_lost": {                                    // include ONLY for closed-lost deals, else omit / null
     "narrative":"…", "quote":"“…” — Name, Title (email, date)",
     "crm":"<b>CRM says:</b> Closed Lost — Category …",
     "flag":"<b>⚠ …</b> evidence-vs-field reconciliation …",
     "winback":"<b>Win-back angle.</b> …" },
  "gong": [ {"title":"[Gong] …","meta":"date · duration · attendees","brief":"…","points":["…","…"],"next":"…","url":"https://…gong.io/call?id=…"} ],
  "timeline": [ ["Jul 9","Gong · Broker sync","…"], ["Aug 13","Email · Decision","…"] ],
  "timeline_note": "96 email activities on record; grouped by thread.",
  "risks": ["…HTML…","…"],
  "next_steps": ["…","…"],
  "footer": "Generated by Medefy RevOps · Deal Handoff skill · Source: HubSpot portal 8313828 + Gong"
}
```
Notes on the schema:
- `snapshot` = exactly 4 cards; mark the Stage card `"lost": true` on closed-lost deals (renders magenta).
- `contacts` `role_class`: `buyer` = economic decider (magenta), `broker` = blue, `internal` = cyan.
- Omit `closed_lost` entirely for active/won deals — the card only appears when present.
- If there are no Gong calls, pass `"gong": []` (the builder prints a "no Gong calls" line).

## Handoff builder — write this to `build_handoff.py` VERBATIM
```python
#!/usr/bin/env python3
# Deal Handoff deterministic builder. Usage: python build_handoff.py data.json out.html out.pdf
import json, sys, html as _h
LOGO = "iVBORw0KGgoAAAANSUhEUgAAA0gAAACqCAYAAACXg2/PAAAACXBIWXMAAAsSAAALEgHS3X78AAAgAElEQVR4nO3dT2wcWX7Y8SdRJCVh1q1dj1e7OxM35V0jUS6kDon2Jk4OQXIS5zCIc2rq4AU2BiwKQQIDxkBkdLKDRBSQwxoBIjYMw1joIDJOsPEitkhgDUMbxGoGHswhjsVKgAQDazFqAxbV3dNi8Lp/NSqVusn+835V71V9PwCx42SmWazqqnq/936/3zt1dHRkAAAAABjTancvGGOWjDHLxpiFxE91hNOzb4x5bozZNcY07M/83MwBpzUsBEgAAAAotVa7awOiVQmKFh2fi0gCpu35uZntsp/rEJz6O/+qvfDpx7NEtgAAwDkZeF5w+Ll2Rv45VwrTarW7CxIUrY64OuRC0xizZYzZZGXJXzZAsg+ZTfvz6cezPHAAAIAzrXbXzpxfc/iRH8zPzexyhTApCYzWjTG1nE/ijfm5ma2cjwEDnDbGVIwxt+2MzOU7nVVOEgAAAIrG1ha12l0bkDz1IDgyUqMED51OHJJdWrx/+U5n9/KdzjIXCwAAAEXQanfXjDEHngRGVjQ/N0OA5KnTAw7LLoM/unyns3X5TsdlzjAAAACQGVk1so0R7krWlC9IE/XYoAApZiPsg8t3OutlP0kAAAAIizQIsYHIdQ8PnG52HjsuQDJxfdLlOx0bKK2U+kwBAAAgCIngyHXLbido9+23kwKkmK1Peij1SQslPl8AAADwWCI48imlLmnHn0PBIKMGSDFbn/T08p3OJvVJAAAA8EkAwZGh/sh/4wZIsZtSn7RWsvMFAAAAD9mGDFLb43NwZKg/8t+kAZKRL99dqU+iLTgAAADytC1lIT7bn5+bOeBb4rdpAqRYVdqCb1OfBAAAgKzJPkfXAjjxpNcFwEWAFLsu9Unr1CcBAAAgC612107Qh7ItDel1AXAZIMVuS33SahFPGAAAALyyHkDdkdWcn5thBSkAGgGSkS/p/ct3Og3qkwAAAKCh1e7acWYtkJNLcBSIM8qHuSj1SXUb3X/68SxFaQAAAHAlj47K+8aY58aYhvyvZduL2xKThWMaRZBeFwjtAClmI/sVu3+SMWbz049nn4/2nwEAAABvk9qj6xmcmqYxZssGOKOmyMnKVvwTN48gQAqEVordIBWpT7JpdytFOokAAADInPbqkQ2MbszPzVyYn5tZG6d+yP6783Mz6/NzMzZA+qp8DgsEgTgtFz9Ldtnx4eU7nd3LdzpLZb8AAAAAmIhmQ7Admy43PzezNe0H2cDIxecgO6clfzIPdrnxyeU7nS3aggMAAGBUrXZ3RbFz3cb83MwKKz7ldTpRXJaXmrQFD6V/PQAAAPKl1SX5nk2N49qWW54rSEm9+qRf/ned/119QFtwAAAAHEtjvLhna4047fAiQHp11piXl8xn7Yvmb9m24NUHnd3qg85C6a8OAAAA3tBqdy/IVjIuNZVrmhAQGyDltjfR0YwxrffNZy8XjHk1by4m/r9sfdLT6oPOZvUB9UkAAAD4kkaTr835uRn260TP6U8/nm3k0MnOdH7BNA+/Yw6777wRGKXdtAFc9UGH5U4AAAAYpfS6Tc4sYvE+SJml2XW/Yl4c/rJpdn7eVMwpc26E/8TWJ92tPug0qE8CAAAoPddlGDt0rENSHCCp7+x7NNurM3rWes+cP5qZqC3jotQnbVOfBAAAUFqux4Hq42CEJQ6QRt4ZeFy2zqj9DfPs8Nu9OqN3HXzkdalPWqc+CQAAoHRcB0hq42CEqRcgSR1S5Pov6HzNvLR1Rl9ccBIYpd2W+iQ6jgAAAJRH1eVfSnMGpJ1O/N/Olhe75405/LZ51vm6OTtindGkbKrefWkLTn0SAAAAxrHP2UJaMkDamvbs2Dqj1i+aZ61f7P2zxqrRMNekPmmL+iQAAACMiOYMeMuXAdI0aXa2zqjXtvvbvdWjLAOjtJrtyGfrk3I8BgAAAACBOp067LFXkb64YA4Pv2Ne9tp2+8Eex+3qg46tT1rhiwkAAABgVOkAaeRNsmyd0ctL5rP2N8w5c8qc9fCM2wK+h1KfpLHjMgAAAICCeSNA+vTjWZuHWT/uT+zVGb1vPrN1Rq/mzcUAToetT3oi9Um0BQcAAAAwVHoFyRpYv/NlndEvmRfdd4IIjNJq0hZ8za/DAgAAAOCLtwKkTz+ePUivInW/Yl68/CXT7NUZnTLnA756tj7prtQn0RYcAAAAwBsGrSCZeBXp1dl+nVHrPXP+aMabJgwuVKUt+C5twQEAAADEBgZIdhWpfdH8/suFYOqMJmXrk55WH3Q2qU8CAAAAMGwFyXzxVfPPjDHNkpyhm1KftOrBsQAAAADIyamjo6Ohv1kaGtwt2cXZN8asRR/N7npwLCiRVrtrVzGX5Mf+c7JObkFSQ2P7id2/D+THbvZ8MD830wj5rLXa3SX5e+PzkGzTvyS1hLG9xD/vyjlpzM/NcP9OodXuLgy4BvEqu/3fxcSnR/L9i+0m/td+Hw9cHBOGa7W7y3K9kj+xa4l/bspzItaI7xnNZ0er3d1NHce0PijaPS7X8MIEz72D5E9Znn2tdnf44HV8e/NzM9Sl4w3HBkimHyS5frCFYkcCJV7uUCGDUPtQXpEXYNXR72nK4NT+bPs+QG21uytyHpYcP2v2Eucg6KBRmwSl8fdwOTUYm1by+7jLtZhO4rkR3zOLjn/FngRM8fV6PsJ/cywCpDfJZNhy4sf1NdxP3XNTX0PfECBB2ygB0oI8LIvUpGF0F1/eMfOv/m10tVK4B0zZyAydK88nGejJi3FVfly/FIexL8st++PLi7LV7q7KgPx6Rr8ySpwDJj1eB6bxT5bPd3sttuVaECyNQIKilYyfG7EduV7bkz4/CJDCv4auJTImJvXI4SH1ModyOhW2M/QLhc8tzeq947Hdl04MkEw/SLI39H2NA/DW3Ktn5t3Wu+ZM7/zYF/p6dLWyVapzUDB5zjjJy3Fd9uPKk23hv57Hg1POwZoMEPKccLHnYLOMg3NZKVrLISgaxj5bN30K3n0hA8gVuV5ZD6iH2ZFrtT3Of1TWAMnTa1iXa5jr+ZNBrcsgJ1S/b4z5pwrHXopVMcXvUTRSgGT6QdKWB4M7faePmuZr7Vlzvjtov6c9CZSobwhQHgGSR4FRWmaBEucgf/ISWfc8Xbo01+M4Hk0kHGeswLZsAZIERmvy4/M1tPdbLhO/BEg9TakZbDhMsU+6UvSJwFa7u62UiXJraBe7AdZkGbKYTplD83Odpnn/sDIkODLygH9Ufdzcqj5u0hYcQ9kXZKvdtQOIp55OLNhjetpqd9e1foGcg60yn4O82UFIq909kIGI77Wk8fXYkgFmqdjAKHG/3PQ8rb0qDZwO7P1Txus1SOK5/7kx5nYA1/C+fT5IyjOyF6c8agWpeaUNZkImkzSCIxu4bo0cIEUfzT6XgsLitf4+1/3MvHd4zlzojPowq/Xagj9uFnZghcnJzFhDBjm+u91qdxuSeuVMq91dk85KIaw6q5yDPNm/RWbtHynNTGqqxQPvwI57IjKoXpdnRmhZGhUJBBplH2TLNTwI5LmfFAdKu0V6BgYifsZtKo2taxJEFJXWO6K3Mj7OClLxgqTZV5+Zb7w05hdaF83psbOvei+G6uOmDZRWdA4QoZGXZGiDUpsb/8TFAEdmwXdldjmkxi72HOyGPshLzGA/Cbz7aCURuBY2j14aZTQCWG04SWkH2YlV2tCv4TV5D2yyIpiJvTidWFaRxqrrG0MhJy4S9X0a7Dt0+Eaxw0QfzTaCD5L6dUaH5psvL5q5V9N+mn0xPKw+bu5WHzeZfSkpGZhuy0syVPclxWciicFeqAPzipyDTQ+OZWyBrVyOygauj4q2mpR4XjwMcIXvOPEgu/Crf4nJiBBXaY9zUyaLGM/oSr9ntO6ZoqbZadVo1uPAdewAyYQcJJ0yL3p1Rt96WTHvfHHO8af3XgzVx81N6pPKRWYydjNsWa2pNkmQJAOihwXZDuDmNIFiHgo6UEuKV5OCf7ZKIHtQkOfFMPH1KmR6jwQPRZuMSHKWVYCBonQnSBmU7yicrkpBr6NW4Pflu3+iAMmEGCTZOqNvHp7v1RmNn043jptSn1To4jj0JYIjX1q4ujBWkCT/bsgrZ4NMFChmTWaxdws8UEtalNqkYGe2Eym4ZdhXcFFqkwqVgi6Dzd0CT0YkTZVVgKGGZSloZS8Uajwq96DG/beX7I45cYBkQgmSZo6ema+3+nVGZ1QDoyT78rsr9UnszlxQBQ2OYiMFCPLvFLX9v9dBUmIWO+Rao3FVQpzZTnR0LNpEwkkqsrJciHQtWam9X7KN82tFWb31RHNY1zoZnGt0i14sWC2n1vP/jesyVYBkXgdJC961AD9lXvbqjN47fNec7eZ1FFVpC27rk4rcSaSsihocxWrH1RIUPDiK1aQjn1ckOCrLLPYg90MJkhITKcXfR3C44AMKed6VYaV2kLiJDUHS9E7aO0xrFakQaXYS6GlMCkbpPcGmDpDMm93t6i4+b2q2zui9w7MKdUaTshfzqW0LTn1SYVwreHAUuz1o5qkkwVHsrk+zb4ngqEyz2IN4HyQVfJW5NEr2vBuGIMmNYwMgGaRHCr+3KC2/tZ75b10XJwGSkSAp+mjWHviN3FLuznafmW8dmgzqjCZ1W+qTKHxESLaTL0VJMynbYGHbh4EBwdFbvA2SCI6KgeDoDQRJ09mJO6SdgI1jB5AAT+NeHJj26CxAikUfzW5JvvGe688eyrbttnVGX2+9m2Gd0aR6rYSrj5sN6pMQiEr88JDBaBnTTCqKL62REBwNdd+3RgAER8VAcDQQQdLkRk2fU0uzC/y6qXWuG5T26DxAMv0g6SD6aNYO/m+priadMofmQuelef+wkmOd0aQWpT5pi/okBOC61OIEuUeQI9fzGojLzBnB0XBbvnS3IzgqhpKulI9qseTvgklEyQ5px5HBukbJSkVxc1VV8lzNLL3OOnV0pLviUn3QuSC/3O2D5p0vnpkLnXc9TaUbV1PO0WZ0tXJc8R6m0Gp3C/FlQa7sSy7TCQ3PBtzJzIBdadATn4+FnJtG2Lz9pRMKoNVJ2/UydRbMwwejDjYnISvl9z34OyPZM8vI/x5IvXdsKedJk1vzczMTBUpS1/nI/SF560a6CcAJ58c+T58q/DGZv8NckAnauwofbTeGHRh4qQdIseqDzrLsFDzdi2P21Wfm59sXzdwrl4fnC/swXI+uVth3QIHnAdK+vPwa8n8/l3+2L8B4SXzZgxeipqb8zQ35+03in+NBQTwgz3MAOtaLblqtdnc7p01F7XdyWwKhxqiBh6zkLCd+svy+2n0scktd9iAlK0rcQweJwXVa/FwJ9ZmiFiDJ9/eJxmefoCn3m712u/NzM41R/iOZQEneb1lPpEx0LUoWINlruzDu5I3iZIvqBIOGVrt7oDQBN/RcZBYgxaoPOqsSKI33h9o6o6+1Z8357nnN4/OEnaVdi65WRnpAYjSeBUiRvAy3x31QJQagawVo87wvtT0jDwhiku4W/2Q5wMtsBk5x1myYSFazt0csJj6RXKfVDIO8iWe1p5HTqkMzEcTuTnrNUkFtHsH4uFQGeBJsNDJ+rtblftt28WGy8rAqP1n8Hbms3Dp+n+c6sTKMYhBpm0UEk2qn+Gw99rpnHiCZ12l3ayNtmmfrjL7Safc605VPXQIl0u4c8CRAssHvpsOXoZuV2WzFHWM2XQzCZVCzMtHEy+TUV5Eybspgv5frymlLC3KNtFdYmjJgcxLgjUL+tkaGgfp+IpB1+n6Qv2XF8wkYrQBpM6MmNF+m1WsGFjKwzOK5ODRNSUsZAiSju3JyKctn5DQUV9I+PG4slkuAFKs+6CzIQ2LwjNW5bj+drhh1RpNqSm3S0A07MZqcAyQ7y7bmKjBKk1n6zQBWlOpyHpwPCiRQGm3iZXr783Mzqk0BMqpl2ZfrkVm6hQT1m8qpQJkOeFrtbiOj1Cb1QDYpwwH2uJwHSBmmfG1oB0ZJGT4XM03bKlGApLV6cm9+bsb7tt+K9+WJmSC5BkgxqU96/cK0dUZf7VwMsDOdpkhWk1QG2GWQY4CkFhQkyYtwy9MUGfv9Xc3iBSorL9sZDOqujJsWOKoMUuuaMtDOrRNVq91dVx60ZVIrlsHfYbK8fwaR7+O6R7VKTgfjGaXW7ck1zGXWXp6LW4qBfKbF/2UJkEz/b32ucO9NVBeVNcW6zhPfDyptvscVfTS7G300u9TbZPZr7Z+Zb74kOHpb9f2/evFw/x/8x5+aap224OGwN+FqFg8h+zskr1ijPeg09iXlKZPBnQQtS/J7NamklMhgTXPF2J6X5TyDI9O/TvZv/FBxK4hN7T0/JB1Nexb2Xpb3zyDyXbF/605ex6BMO51www7A80xpkufisuI1rMpkAdzTeFZXFNtmO6G8MeyJiw1eBEix3iaz73zxHXkhQPzc33TMb//gybM/+bUfm8X/9fnf67V+rNY3TbXORm1+y7TbWUxywX0JkmxueuYFvPL7lpWDJK0iV82Z+roER140gJGU02WlIKmSQfCypXitmrJSor76PIrEBMytvI/FJeUgN76GXgQOGUyirbGBrAqtcYTvKXZaxzdSiqtXAZJlGxJEVyv2pFxK7blRSt/7g794+We/+qPDf/LH0bupv/9mr4Vrte59DmlJ5RIcxTwJknayLtxNSgRJkdKvqLrenFQGa1pF4vWsVjPHkZjZ1giS1AZskhuvVSO2L+kv3rXildWkK6qbwGdLa0KiKZMRPl5DrfdDFpMSpSMrjxrXq5rX5ucnUd4YdqSxmXcBUiy6WjmIrlbsC+gDxQGOt777yTPzp9//w2e/+bt/fnb2i1fnhhxnpVenUK3bQMnb/NkSupVncJSwlkGa2TD7PizfSzCg+QJwfd9pzTRn3mVqHBIkaVwnzQGbVopinALpbW1AIqjN6/nihHIKjzcrtYPI80BjEppVJB1azxtfA9pVpYmL+qiprt4GSLHoamU3ulpZkGX9osxYDfX+X70wP9z4ybMfbvzEfOtnh+lVo2GqvS4f1fo29Um528m7tiMmA6w8BsX2Pl3xZYAng5QNpY93FiApDta8Do5iMtOukb7lfAAgnaU0it138khJnURBgiSNwaH3wVHCisIEtPe1LSGS75NGQHtN3j2+yX1iy/sAKRZdrcRFor4VoDth64x+4/c+ado6o+9+8mzUwCjtutQnrVOflIumby8GeahmXdO37tv+ClIDoLES7TLFTuO7sx9SyotMLrgeBFQkoHFJ61oFNbBMTMIEN3mpmMKzFkhwpDmJRpqdDq3JV6+aa8jzWqNpyt4492YwAZJ5XZ+0KvnPhalP+pU/ig7/+/d+9PL7O//T1XLibalPYhYnW+uezvyuZziA2fNlBW0AjZeAy4e460GFVyt5Y9AYcDs7t0q1R5HvaXXDKNeQaVpRSOG550l69chk5db1JJq3tS0hk6Y2GhN9K56lRWqNXccamwQVIMWiq5WG1Cd9GHJ9kq0z+vE//+PPfut3npyb67w66/jjK73Nxar1XeqTMhH5GhjIoCur/bO8bfMqAxfnzwsZME/7GRqDNe9W8kYhx+z6Xlp0mEZCIJsiQVJoqwaujzfy+fl3Ao1JNCZodWh8x7xprqHY/CYad6P+IAOkmN00VeqTNkKavbJ1Rv/+Xz/+zNYZ/e3/89cXlX/dNalP2qI+SZXvL8Ysgrc9Hzs2pWgEii5m3lwPJnxeyRvFpo+rSBJkud6IeT2UlKzjyAREEPskSfdJ1zVkXrRjn4Qct+vnxXWaNajYVhrv+hLQah3H2GO0oAOkWHS1si61AN7XJ9k6o71f/68v/uF/+3/agVFarbdTOPVJGiLf0ypkAKa92hrC7KnGdZqqDkkGEc4H3Y4/L1NKAzYXKT+u04ZCD2TTQqlH0piQyGqVXovGpARpdo7Js1HjPVZVqNUcS94bw6YVIkAyr9uCr0pbcO/qk/7x4//74n/c+M9NW2d0pvvqfE6HUZH6JBso8eByJ5Scc80XeBTA6lFWgeK4NAbd3l+LEbgOHFzsWxVCs4fc5Ng5c1yu77mgJySM35MSeJvWpEre926uG8OmFSZAiklbcJvDeMOHmay/e9Ds1Rn94N/89HzlbzpaO66PyxaWP5T6JKcbXZZUKAGS5qA5pFlw34KHUPZSypS80FxnBUx8rmV202Va1kaINWInkZUUb5soyXV02VxlvyATEkbhXUb9swLFjWOvud78fFQ+bAybVrgAKRZdrWxJW3Ct/U+OZdt22zqjH/3LR1nUGU3K1ic9kfok0u4msx/QIEfzJR5Seolv9R4uZ1mDWMkbg+vv1TQDNpfXqRnYpMK4fA7SXQ/aC3Md5V3mcl+rSl4D7hLQmpjNq1lD7hvDphU2QDKv24LbB/WlLItHv/cHf/Hyz371R4c51BlNqiZtwQsx85yxYAIDmZHXWFWNApsJdx0gTbMqseT4pRB6HcQbZDXC5Xd2msGxy4H1RCkfoZAg3ddVJNdpX4W651hFCoPiPVbLqblG7hvDphU6QIpJfdKK1Cep7fpt23bbOqPf/N0/Pzv7xatzWr9HSb8+qVo/oC34WEKbrddYPQltgOBTMOd6djWoPVhG5PL7Nc2MtsvnYhGvU5qvf6PLe26ngIGuT6u2OF4hVpF82Rg2rRQBUkzqk+zD8ZbLWUnbtvu//ItHz2zbbo/qjCZVlbbgu7QFP1nB0pkmFdQ58Gy1y+XgISpCu+gBXH+/xh4gO17p2yli7VGadPb0qqOdzIy7HIgVbfUofj66bGRDip0Srb39ckiz82Jj2LRSBUix6GplU+qTpto92tYZ/fYPnjz7k1/7sbkcNd91epD5s/VJT021vkl90lDeFiJnrIiD8qy4nIQo6nXIPUByHMgWblB9DN9WkVwP1rnnTlZlPyRVGjVwdqU9k452Pm0Mm3ZG4aCCYOuTbJRcfdzclIf4WBfoV/4oOrzzH/ZPzXVeFS0wSrvZi+5tfVJUK3JR8SRCnAVuOH4YNcswG67I5bWwGzMeefcX+meSQbLLQLZMAdKuvEN84Trd60mr3fXoz/PWUoDp6KHYkqYorrOX1jKa4PBmY9i0Uq4gJUl90rLUJ524VGnrjP70+3/47Ld+58m5uc6rs5kebH7sjXfXVOsN6pPeEGJg4DpfntWjCUm7YWRvkvPuauVhr8jNGdI83DyVlYx88KxTIs8TjftsUVZ31Pi2MWxa6QOkmNQnxW3B38qbtnVGP9z4Sa/O6Fs/Oyz6qtEwi1KftE19Ug8zYgRI0+AeysckNSiuAqQyPjN8SkWmHiYfPOt0aXUg1k6z82pj2DQCpBRpC74Qb8Jl64x+4/c+ado6o+9+8qysgVHadalPWqc+qfRKMxuugHsnJxPURLhKXyljgOTT38w9lw/OuyJJc9fYyqamleng48awaaWtQTqO1Cet/vQf/ae/vBz99dpXXnS+6u/R5uq2BJOZFPP5hg52PdQfTY7Z7PyMXBPheKPLMt4vPq0yL3pwDGXEs07fpkxeu7aqtELl3cawaQRIg/TrbNb/vk5njaKIejdNVCvDfh4YjgAJReds9rukDU1YZQaU2QnbVru7rzAJsKYUIHm3MWwaKXZJNl2sWt/q1dkQHA3TlDqtJYIjABiZxn4lIaBOEchGEC2/fd0YNo0AKWbrafqz4RodNYqiLoGRXTliVhCYDoXL5VLK1VZfuvbRNRJFp7hxrOsVJC83hk0jxa5aX5GTqhHNFsV+bzk0qlFz8xqbxGJaDNiA7HC/oQy2pD7cJbvZ77KLumufN4ZNK2+AVK0vSWBEKt1wTQmMSKUDAADw26ZCgGSkZsjFJLm3G8OmlS/Frl9nZL9ATwiOjrXRm3EjOAIAAPCepLTWFY7z+rRpqr5vDJtWrgCpWl+TPPCbHhyNr2zq2CXqjAB13F9Adui4ibLQ2jh22s5zXm8Mm1aOAMm27a7W7cPxrlLf9SKwhX0fmKi2bKIaLxJAH929yqWUe8H40hyhpC3WUULyXdeok16dYJPtnhA2hk0rdg1Stb4gJ45UuuGasp+RRntIAEBfWSfnaI4AZG9dtqxxyT7DViYMSLzfGDatmAGSrTPqL+VpFKoVyT0Jjkj1AcL3AddwZOOs3jl7+bba3SWX+3QEoqgrZ3WtmesCYoyRMdk4NlLo0Lw+4fdeK71OK52wgAFStb4qXTxIpRtuT7rTkeID5Mf1rNcBaUTu2XPaanddfe5SCVMrfVpBsltWLLr6MBdtjwFFNni47/jjbcvvlXFaaitvDKv2zitODVK/zqghXwaCo8HsbMKHUmdEcATky/WDnVQmPa42XyxjHdKyB8cQc7mSwf0Gr8nGsU2FYxx3NSiY1t5J4QdIts6oWt+WXEtnM0MF0+y17Y5qtm2381aIACbiOkDyaSBaNK6uVamukRRm+/RednnPUduMEGjUl18btfmK8sawqiu44QZI/f2M1iVd4boHR+SruuxnpBppAxiPQmpAKbukZcTVi3jRl65uGVnx7Hic3nO2pszl5wEKtOrkRh1TBrl6ZIINkPp1Rg1pwkA63WB70rZ7lSYMgLdctmJlBUmPy5Rk34IGTb79ra5nnLnn4DWZiNPYOHblpJbfmhvDSvqgqrACpGp9yVTru1JnpFHwVQQ2V/6G1BlRQAr4zeWMdoUZbTUuAyStGVWvyODJt+wO0lpRRhppdpURnmVqG8Mqfe4bwgiQ+ul0Nlp8Qt7vUP06I5tmE9VoPQqEwfUkRikG31mTWVhXjRoWJS+/6Lz7Ljq+jtb1STfOBLIiWwtobBw7NABS3hiWAKmnX2d0oLRMVxQ7EhixpxEQFtfdJMuUvpU1l8FsGQJZrdnjaXHPoYw0goqqtPAeRHNj2EzGuf4GSNX6iqnWD6gzOta+1BmtmKjG/idAYGRmz2Ub1mpJVify4DJAqhW5WYPivicusGqL0pF9i1yunsaGff+D296s0uIAAA3qSURBVBg2zb8Aqd+22z7AHlJnNFRT6oyWqDMCgue69b6vM/ehc32dCpkKLak1PndNdX0dr1H7h0Bo3Jdvff9D3Rg2zZ8AqV9nZJcAn1JndKx70rabOiOgGFxPclwv2upEq91dty/dPAeiktax4/AjrxV0tW/N58lNhTokU7RJCfu9lHtumRqrQtnOaOPYYFt7J/kRIFXra1JndNODo/GVLbC7ZKLaGnVGQKFobN5cmH3PZDbytnQvfdJqd5+32t1dGcCd2GrWMeerSEUagEoAe9uDQzmJ6+tYtJTJLbmOdgP+z1vtbqPV7trv6hqrZeGSSR6NyfVa/BwLeWPYtHwDpGp9WeqM7lJnNFQkdUbL1BkBxaOwMmHkhRX86sSQdK2KvIBvSyq2HcAdtNrdbRnAqf3dsveG05qxrDoyaZNrFUpmg8ZxFiKrw95DA1YAF6VR1l2ZpDiSSYpNmaQo0+bHodN63sSrSIVYPTK5BUj9OqNtmZ2gzmgw+xK+ZaLaAnVGQOFpDK6KMPAeNV2rKnvu2AHcIxnANWQAt+p4AOf6WtWO6QQVkk0ZSHtPmqPsOz5OmzIZdEe7MevHrknWj52keCoru9txap7yoWJCihvHroa+MWxatgFSv85oXeqMfNtAzid1qTMqxMwigOMpdRiy++0Em2rnIF1rUQZw92UA56q9s8ZzeTPk1CUJ8ELbikPjOoaeMrk1RTZPRcZ1txOTFLRA95NGsFFVqKeN5TIWzi5AqtZXpc4ohPzkvNg6oysmqq1SZwSUjsZL63aIA2+ldC0nn6c0A2sHl7uBXqtVCUJDo1GwXgk11U6uo8uJ66ZM/MAzUsujsXGsVkZYQQOkfp1RQx6g1BkNZmeOP5Q6I9eb2AEIw6ZSh6HdAGe1XadrNR0PXDVW5oILkgIOjuLaP42B1/XQVm7lO+f6XJAB47dQAvnMNoZN0wuQXrftfhRKXnIO7Et7Q+qMmGkBSkxeAhrPgUpIQZIMLl2na227fMkq5vEHEySFHBwlaA0Sb4dSVybPhV2FCWy2IvGY1PRobBzrWm6TDToBkl01om33SezL1W70Wph2vACmtq60irQYQpCUaOntmsZzVuvZHQdJ3g6wJYgNPTiKA917Sh9/3/cgSTE4qme5oScm5vsqX6Ybw6a5D5BerxqRTjfYnrTtXqVtN4AkeRlovbS8DpKkvbDGoFtlsCafueH6c0VFBtheDWDsd8e2dy5YLbHWpITxOUiSjmO7Shk+TPyGwfW2Ba7l+j1yFyD1W3c3WDUayi5l3pA6I9p2AxhGqxbJyGCo4VsKl92EUlp0u9YcsMu7S5vKaSo3pV157tdLOpIdKG0CmRvFWqSYj4Gu/T41lIKjDVaPwqCY1u1C5hvDprkJkKp1zZutCDYknY6cXADHkpeW5qC+KitJmr9jJHYWW9pva7WI3tQs8JXP1l4hWJTNOXNpIW0H07Jq9LComSHzczPrGQS6uz5sqCr3/ROla9mkOUNwfF3ty/24pg+Q+u27NXJYi8Dujn+pV2dE224AI5ICWo02rDH7vL6b56BN6lg0J9YiGfiqkllOrTqWJBtEHsjmt+rXTAKjLRlMF2rVaAjtQPearN7mMjGRCHQ1Vmpja3l1HMNkZLVvx7PTl8vGsGnTBUj94Ij23W/blzqjFeqMAExoNYP88GuyiepmVqsTtiaj1e7Ge+JpvjuyrP1Yl+e+toqksdtrtu16I06pMVqVgfSTADd/nVhGgW48MXGQ1SaqskqbRaC758OgFhPxbdXPi+OZPEB6HRzhNTuYuWWi2hJ1RgCmITN7Wc0220H355LG5bzeRWavbRD2XN4bWhsKxu5lmb+eSLXLsuDZbur50J5TCZZWJ7l2rXZ32a5qSFD0uVyfMqwYDZJVoFuVa3cg183p5EQq0H2aQaDbzHhCAg7JszKL7/2ovAiQzkz0XxEcDXKv93AllQ6AI3ZG1g5gM5zJt7+n1mp3Iyne3Z4k0JAUMDtYt8e+kkFAlLSfR/76/NxMQ7ET33EqEizZH3vuTSI906Ywpt9JF+TaXKBu+E020JWuc1mVDVTl+2IbOezIPbc7SZMDeU7E99x1ncMdao3GDMHb9GRcn9vGsGmnjo6OxvsvqvUVKdZE315v5oRUOu+12t0xv+zHsukEywGeg3XHLXo/yLvTzCRC+y5II4M8B7P7MtAedq3jQbfJefXBzmQv22AlrwNQuMfwNtXnjqS/5TnOaSaC22Hf5TjIXch4AiLNDmgzXz3ife6epD7n+V2yLvkSbI+3gtTvVkeOaV8kgRGpdAC0LctAKa+XVxyc+Z56tZZncGSkI5qsoJWmfqdo5udmbMriLeWGBsepJO61rFeDxrGfYRow9G3lPLmT68awaaPXIFXrWjsuh6bZa9sd1RYIjgBkQVIOVjzf1C9vt3wpEpcZ9boHh4IJzc/NbHINjxXJai1lBcWRd+2PVy3Hx2nSsE1w1HtYLvTadgNAhmRlZJkgaaC6DGi9QZAUPq7hUPYZtEJwVCxyPfP6vue+MWzaaAFStb5e4q42RuqMrpiotkoTBgB5IUgaKJcaiFEwwA4f1/Atudf5QVVeCwDeLTycHCBV68slLji1S8gfmqi2bKIaDwMAuSNIeoO3wVFMji+LjWShhCDpSwRHBSc1QJqblA8S+biH1vEBUr/uqIxNGfp1RrZLTFTb9uB4AOBLMkBZ8mzviqxt+B4cxebnZmwh+w0/jiZzTZlsDJp818p6DY08awiOyiHr1Rwv44yTVpDWPGj5l7W6BEbsaQTAWzLTZ1eSdkp2leyA+4btFufBsYxMZkivFCFYGMOetKEuxDYYcg0/LOHq7R7BUXlILVBWz6mmLxvDpg0PkKr1hZKl1tnZkQ+kzog9jQB4zxbVzs/N2O52t0pyteJZ7CAzGxIrf2UIau0KX+G6nNkW4CVbvS3kdcSJspqA2vb1u3XcClJZUut6s5Emqi3RthtAiKSD25WCD9ruFWEWOxHUFnUlwn4Hr4S2wjcOu3o7PzezJKn4RRXJhrx07S0hmYTK4vnk7fdrcIDUb8xQhq51G9K2m81vAQTNBg6JQVuRBt7xQG2tSLPYshKxUKDi/6asNiyVJRVLgocrORS1a7OTEUu+tV1G5rRT33Z82hg2bdgKUtFnDGx6wyXqjAAUjQzalgow8I4H3AtFHajJatJqL7077EF2XQbUpVttkImJZWngEHp92Z6s/hVqMgIT0w6QvKw9ir0dIBV79SiSOqMV6owAFJWkAIU68I67iC6UZcBtA0AZZId2vWxgdMl+13yeCc6CpCSFuoK7J6u0NGLAl5Q3jt33feJr0ApSEV9IzV4Rc1RboM4IQFkkBt5XAlhRipKBURlnsFOBkq+NHJoERoPJiuC6pE6GsKK0kwiMGBthkAWls+L16pF5K0Dqd64r2urRPakz8v5iAIAGSQNa7aUW9zve+TRws4PtDyWVrpSBUZoESityvTY8uV77MuhfIDA6ngRKW/Y7Lc04fJqciOQZYAPcFQIjDNNqd7UyyrzcGDbt1NHR0ev/p2rdBhE3PTm2adklY1p2A8AArXZ3qfeM7O+ltJjhObIDNDsos00KdgmIRiPXa0V+srpeO/G1IiCaTqvdvZC4fvaeq2T46/cS9xspdBhJq921QUxN4WxthJA+nQ6Qnmd802qIehvcRrXtwP8OAMiEDN6W5WfJ8ayhHZw15GeXgfb0EtdrKfG/0767o9R1YmVBkQS8yXvO1ab8XEdMrdXu2tXPpwpnsimr0N5PjL0OkKp1O6vxMO8DmkJ/N17bmQ4AMBV5QSZ/rAsymEs6kJ+YHZjZl1+D1aHsSNAUX5vlEX7xc7lWhkG0HySlyaSu35Lcd0nJ6xVfx+esDsGVVrurlVFWl3Rv7yUDJK2ltCzUZdWIlzEAAAAwAZlsOVDKKLsUShbBmcQ/r+R4HJPak8CIWRMAAABgOnaFRyM48npj2LR+gNTf+yik2qOo1448qnnfBQMAAAAIxJrSYQbVTTpeQRolX9kHTTnBm6TTAQAAAG602t1Vhw1DkrzfGDYtpABpR9Lp6IAEAAAAuKXVQCG4vUjjAMnnzWH3JTCiyw4AAADgWNk3hk07Y6r1dMtWXzQlMKLOCAAAANCjtXoU5Dj+9IA9LXyw0dt7g+AIAAAAUCP73mls9dMMMb3OSIrdwgj/Xlb2ehEsdUYAAABAFrQ6122HumH4GU8aNEQSGFFnBAAAAGRANobVSq9bD/Uanhnh39HUlP2Mglx+AwAAAALGxrADnM6xg909qTMiOAIAAACyx8awA+SxgrQn3ekaOfxuAAAAoPTYGHa4LAOkSAKj7Qx/JwAAAIC3sTHsEFkESP0Wf1Et2EItAAAAoCjYGPZ42gFSXVaNgmzxBwAAABQQG8MeQytA2pPudLTtBgAAADzBxrAncx0gRRIYFSJ6BAAAAAqGjWFP4DJA2pBaI9LpAAAAAM+wMexoXARIO1JnFOxmUAAAAEAJsDHsCKYJkPYlMKLOCAAAAPAfG8OOYJIAqSmBEXVGAAAAQAAUN4bdC31j2LTTY/7794wxCwRHAAAAQFBo7T2iUVeQ9nonlTojAAAAIChsDDuekwKkSAIj6owAAACAMGmtHhWq9ig2LEBqyn5GhfyjAQAAgDJQ3hi2kGU3gwKkujRhYD8jAAAAIGxaneu2irIxbFoyQNqTwKjhx6EBAAAAmJTyxrCFzTQ7I3VGNjDa9uB4AAAAALihtTFsvUgbw77BGPP/AeC3SaMOJWk0AAAAAElFTkSuQmCC"
CSS = r"""
:root{--primary:#2882FA;--cyan:#1EC8F0;--magenta:#FF1EA0;--pink:#FF57B0;--ink:#202035;--bg:#F4F6F9;--card:#FFF;--muted:#7A8699;--line:#E4E8EE;--grey:#E9EBEE;}
*{box-sizing:border-box}
body{margin:0;background:var(--bg);color:var(--ink);font-family:'Inter','Segoe UI',system-ui,Arial,sans-serif;-webkit-font-smoothing:antialiased}
.top{position:relative;overflow:hidden;color:#fff;padding:28px 0;margin-bottom:22px;background:radial-gradient(105% 150% at 6% 135%, rgba(30,200,240,.42), rgba(30,200,240,0) 45%),radial-gradient(95% 150% at 102% -25%, rgba(150,80,240,.48), rgba(150,80,240,0) 52%),radial-gradient(70% 130% at 84% 130%, rgba(255,30,160,.22), rgba(255,30,160,0) 55%),linear-gradient(115deg,#1b1c3c 0%,#181934 55%,#211a3c 100%);}
.top .inner{max-width:920px;margin:0 auto;padding:0 22px;display:flex;align-items:center;justify-content:space-between;gap:20px;flex-wrap:wrap}
.brandmark{display:flex;align-items:center;gap:15px}
.logo-img{height:38px;width:auto;display:block}
.eyebrow{font-size:11px;font-weight:800;letter-spacing:1.5px;color:var(--cyan);text-transform:uppercase;margin-bottom:5px}
h1{font-size:22px;margin:0;font-weight:800}
.sub{color:var(--pink);font-size:13px;margin-top:5px;font-weight:600}
.toolbar{display:flex;gap:10px}
.btn{border:0;border-radius:9px;padding:10px 15px;font-weight:650;font-size:13px;cursor:pointer;text-decoration:none;background:#fff;color:var(--ink)}
.wrap{max-width:920px;margin:0 auto;padding:0 22px 40px}
.sc-grid{display:grid;grid-template-columns:repeat(4,1fr);gap:14px;margin-bottom:20px}
.sc{background:var(--card);border:1px solid var(--line);border-radius:13px;padding:15px 16px;box-shadow:0 1px 3px rgba(32,32,53,.06)}
.sc-lab{font-size:11px;color:var(--muted);text-transform:uppercase;letter-spacing:.5px;font-weight:700}
.sc-val{font-size:18px;font-weight:800;margin-top:6px}
.sc-val.stage-lost{color:var(--magenta)}
.card{background:var(--card);border:1px solid var(--line);border-radius:14px;padding:18px 20px;box-shadow:0 1px 3px rgba(32,32,53,.06);margin-bottom:18px}
h2{font-size:16px;margin:0 0 12px;font-weight:800;text-transform:uppercase;letter-spacing:.4px;color:var(--ink);padding-bottom:8px;border-bottom:2px solid var(--grey)}
.kv{width:100%;border-collapse:collapse;font-size:13px}
.kv td{padding:7px 8px;border-bottom:1px solid var(--line);vertical-align:top}
.kv td.k{color:var(--muted);width:38%;font-weight:600}
.kv td.v{font-weight:600}
.ct-grid{display:grid;grid-template-columns:1fr 1fr;gap:12px}
.ct{border:1px solid var(--line);border-left:5px solid var(--muted);border-radius:11px;padding:12px 14px;background:#FCFDFE}
.ct.buyer{border-left-color:var(--magenta)}.ct.broker{border-left-color:var(--primary)}.ct.internal{border-left-color:var(--cyan)}
.ct-nm{font-weight:800;font-size:14px}.ct-role{font-size:12px;color:var(--muted);margin-top:2px}.ct-note{font-size:12px;margin-top:6px}
.narr{font-size:13px;line-height:1.6}
.cl-quote{margin:12px 0;padding:12px 14px;background:#FFF0F8;border-left:4px solid var(--magenta);border-radius:8px;font-size:13px;font-style:italic;color:#7a1750}
.cl-line{font-size:13px;line-height:1.6;margin:10px 0}
.cl-flag{background:#FFF7E8;border:1px solid #F3D28A;border-radius:10px;padding:12px 14px;font-size:13px;line-height:1.6;margin:12px 0}
.cl-win{background:#EAF6FF;border:1px solid #BFE1FB;border-radius:10px;padding:12px 14px;font-size:13px;line-height:1.6}
.gong{border:1px solid var(--line);border-left:5px solid var(--cyan);border-radius:11px;padding:14px 16px;background:#FAFEFF;margin-bottom:12px}
.gong-h{font-weight:800;font-size:14px}.gong-meta{font-size:12px;color:var(--muted);margin:3px 0 8px}
.gong-brief{font-size:13px;line-height:1.55}.gong-pts{margin:8px 0;padding-left:18px;font-size:13px;line-height:1.5}
.gong-next{font-size:12.5px;margin-top:6px}.gong-link{display:inline-block;margin-top:10px;color:var(--primary);font-weight:700;font-size:13px;text-decoration:none}
.gong-attr{font-size:11px;color:var(--muted);margin-top:6px}
.tl{width:100%;border-collapse:collapse;font-size:12.5px}
.tl th{text-align:left;color:var(--muted);text-transform:uppercase;font-size:10px;letter-spacing:.4px;padding:6px 8px;border-bottom:2px solid var(--line)}
.tl td{padding:7px 8px;border-bottom:1px solid var(--line);vertical-align:top}
.tl-when{white-space:nowrap;font-weight:700;color:var(--ink);width:16%}
.tl-type{white-space:nowrap;color:var(--primary);font-weight:600;width:22%}
.tl-sum{color:#3a4658}
.two{display:grid;grid-template-columns:1fr 1fr;gap:18px}
ul.tight{margin:0;padding-left:18px;font-size:13px;line-height:1.6}
.foot{color:var(--muted);font-size:11px;margin-top:16px;text-align:center}
a.deal{color:var(--primary);font-weight:700;text-decoration:none;font-size:12px}
@media print{@page{size:A4 portrait;margin:11mm;background:#F4F6F9}html,body{background:#F4F6F9;-webkit-print-color-adjust:exact;print-color-adjust:exact}.toolbar{display:none!important}.wrap{max-width:none}.sc-grid{grid-template-columns:repeat(2,1fr)}.two{grid-template-columns:1fr}.card,.sc,.gong,.ct{break-inside:avoid;box-shadow:0 1px 4px rgba(32,32,53,.10)}}
"""
D=json.load(open(sys.argv[1],encoding="utf-8"))
def esc(x): return _h.escape(str(x))
def snap():
    o=""
    for c in D["snapshot"]:
        cls="sc-val stage-lost" if c.get("lost") else "sc-val"
        o+=f'<div class="sc"><div class="sc-lab">{esc(c["label"])}</div><div class="{cls}">{esc(c["value"])}</div></div>'
    return o
def kv():
    return "".join(f'<tr><td class="k">{esc(k)}</td><td class="v">{esc(v)}</td></tr>' for k,v in D["deal_snapshot"])
def contacts():
    o=""
    for c in D["contacts"]:
        o+=f'<div class="ct {c.get("role_class","internal")}"><div class="ct-nm">{esc(c["name"])}</div><div class="ct-role">{esc(c["role"])}</div><div class="ct-note">{esc(c["note"])}</div></div>'
    return o
def gongs():
    o=""
    for g in D.get("gong",[]):
        pts="".join(f"<li>{p}</li>" for p in g.get("points",[]))
        nx=f'<div class="gong-next"><b>Next steps (at the time):</b> {g["next"]}</div>' if g.get("next") else ""
        lk=f'<a class="gong-link" href="{g["url"]}" target="_blank" rel="noopener">Open Gong recording →</a>' if g.get("url") else ""
        o+=f'<div class="gong"><div class="gong-h">{esc(g["title"])}</div><div class="gong-meta">{esc(g.get("meta",""))}</div><div class="gong-brief">{g.get("brief","")}</div><ul class="gong-pts">{pts}</ul>{nx}{lk}<div class="gong-attr">Summary auto-generated by Gong.</div></div>'
    return o or '<div class="narr" style="color:var(--muted)">No Gong calls on record for this deal.</div>'
def timeline():
    return "".join(f'<tr><td class="tl-when">{esc(w)}</td><td class="tl-type">{esc(t)}</td><td class="tl-sum">{s}</td></tr>' for w,t,s in D.get("timeline",[]))
def bullets(key):
    return "".join(f"<li>{x}</li>" for x in D.get(key,[]))
p=[]
p.append("<!doctype html><html lang='en'><head><meta charset='utf-8'>")
p.append(f"<title>Deal Handoff — {esc(D['company'])}</title><style>"+CSS+"</style></head><body>")
p.append("<div class='top'><div class='inner'><div class='brandmark'>")
p.append(f"<img class='logo-img' src='data:image/png;base64,{LOGO}' alt='Medefy'>")
p.append(f"<div><div class='eyebrow'>{esc(D['eyebrow'])}</div><h1>{esc(D['company'])}</h1>")
p.append(f"<div class='sub'>{esc(D['subtitle'])}</div></div></div>")
p.append("<div class='toolbar'><button class='btn' onclick='window.print()'>⬇ Save as PDF</button></div></div></div>")
p.append("<div class='wrap'>")
p.append("<div class='sc-grid'>"+snap()+"</div>")
p.append("<div class='card'><h2>Deal Snapshot</h2><table class='kv'>"+kv()+"</table>")
if D.get("deal_url"): p.append(f"<div style='margin-top:10px'><a class='deal' href='{D['deal_url']}' target='_blank' rel='noopener'>Open deal in HubSpot →</a></div>")
p.append("</div>")
p.append("<div class='card'><h2>Key Contacts &amp; Roles</h2><div class='ct-grid'>"+contacts()+"</div></div>")
p.append("<div class='card'><h2>Where It Stands</h2><div class='narr'>"+D["status_narrative"]+"</div></div>")
cl=D.get("closed_lost")
if cl:
    p.append("<div class='card'><h2>Closed-Lost Story</h2>")
    p.append(f"<div class='narr'>{cl['narrative']}</div>")
    if cl.get("quote"): p.append(f"<div class='cl-quote'>{cl['quote']}</div>")
    if cl.get("crm"): p.append(f"<div class='cl-line'>{cl['crm']}</div>")
    if cl.get("flag"): p.append(f"<div class='cl-flag'>{cl['flag']}</div>")
    if cl.get("winback"): p.append(f"<div class='cl-win'>{cl['winback']}</div>")
    p.append("</div>")
p.append("<div class='card'><h2>Gong Call Summary</h2>"+gongs()+"</div>")
p.append("<div class='card'><h2>Activity Timeline</h2><table class='tl'><thead><tr><th>When</th><th>Type</th><th>Summary</th></tr></thead><tbody>"+timeline()+"</tbody></table>")
if D.get("timeline_note"): p.append(f"<div style='font-size:11px;color:#7A8699;margin-top:8px'>{D['timeline_note']}</div>")
p.append("</div>")
p.append("<div class='two'>")
p.append("<div class='card'><h2>Risks &amp; Open Items</h2><ul class='tight'>"+bullets("risks")+"</ul></div>")
p.append("<div class='card'><h2>Recommended Next Steps</h2><ul class='tight'>"+bullets("next_steps")+"</ul></div>")
p.append("</div>")
p.append(f"<div class='foot'>{D.get('footer','')}</div>")
p.append("</div></body></html>")
HTML="".join(p)
open(sys.argv[2],"w",encoding="utf-8").write(HTML)
try:
    from weasyprint import HTML as WH
    WH(sys.argv[2]).write_pdf(sys.argv[3])
    print("OK html+pdf", len(HTML))
except Exception as e:
    print("HTML written; PDF step failed:", e)

```


## Notes
- Gong summaries are auto-generated by Gong — attribute them.
- Watch email payload size; itemize subjects/dates, not bodies.
- Keep the honest data-quality flags visible in Risks & Open Items.
- The closed-lost loss reason comes from the calls/emails first; the CRM `closed_lost_category` is a cross-check, and any disagreement is flagged in the Closed-Lost Story.
