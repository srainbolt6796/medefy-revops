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

## EXACT VISUAL TEMPLATE — reproduce verbatim, do NOT restyle

The handoff's look is **locked**. Build the HTML with the CSS and skeleton below **exactly** — same class names, same banner, same card styles. Only substitute data into the `{{PLACEHOLDERS}}`. Do not redesign, rename classes, reorder sections, recolor, or "improve" the styling. A run that changes the design is wrong even if the data is right.

**Non-negotiables (so every run matches):**
- **Banner** = a full-width dark bar at the very top (class `top`), NOT a rounded card. Logo left; cyan eyebrow (`Deal Handoff · Closed-Lost Win-Back` for lost deals, else `Deal Handoff`); white `<h1>` = the **company name**; pink `sub` line = `{Stage} · {close date} • Owner: {name} • Broker: {firm}`. White **Save as PDF** button top-right (hidden in print).
- **Section titles** use the `h2` style with the **2px underline divider** (16px, ink `#202035`, bold, uppercase). Not grey, not giant.
- **Snapshot** = exactly 4 white `sc` cards: Stage · Annual Value · Close/Target date · Owner. For a lost deal the Stage value uses `sc-val stage-lost` (magenta).
- **Key Contacts** = `ct` cards with a colored **left border by role**: buyer/economic-decider `ct buyer` (magenta), broker `ct broker` (blue), internal `ct internal` (cyan).
- **Closed-Lost Story** (lost deals only) = its own card right after "Where It Stands": narrative, then the buyer quote in a `cl-quote` block, a `cl-line` "CRM says…" line, the amber `cl-flag` ⚠ mismatch box, then the blue `cl-win` win-back box. Omit this whole card for non-lost deals.
- **Gong Call Summary** = `gong` card(s) with a cyan left border, an "Open Gong recording →" `gong-link`, and a `gong-attr` "Summary auto-generated by Gong." note.
- **Activity Timeline** = `tl` table (When · Type · Summary).
- **Risks & Open Items** and **Recommended Next Steps** = two side-by-side `card`s inside a `two` grid, each with a `ul.tight` bullet list.
- Page tint `#F4F6F9`; white cards with soft shadow; dollar values in primary blue.

### CSS (paste exactly into a `<style>` block)
```css
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
.gong{border:1px solid var(--line);border-left:5px solid var(--cyan);border-radius:11px;padding:14px 16px;background:#FAFEFF}
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
```

### HTML skeleton (fill the `{{PLACEHOLDERS}}`; keep everything else)
```html
<!doctype html><html lang='en'><head><meta charset='utf-8'>
<title>Deal Handoff — {{COMPANY}}</title><style>/* the CSS above */</style></head><body>
<div class='top'><div class='inner'><div class='brandmark'>
<img class='logo-img' src='{{LOGO_DATA_URI}}' alt='Medefy'>
<div><div class='eyebrow'>{{EYEBROW}}</div><h1>{{COMPANY}}</h1>
<div class='sub'>{{SUBTITLE}}</div></div></div>
<div class='toolbar'><button class='btn' onclick='window.print()'>⬇ Save as PDF</button></div></div></div>
<div class='wrap'>
<div class='sc-grid'>{{SNAPSHOT_CARDS}}</div>
<div class='card'><h2>Deal Snapshot</h2><table class='kv'>{{DEAL_SNAPSHOT_ROWS}}</table>
<div style='margin-top:10px'><a class='deal' href='{{DEAL_URL}}' target='_blank' rel='noopener'>Open deal in HubSpot →</a></div></div>
<div class='card'><h2>Key Contacts &amp; Roles</h2><div class='ct-grid'>{{CONTACT_CARDS}}</div></div>
<div class='card'><h2>Where It Stands</h2><div class='narr'>{{STATUS_NARRATIVE}}</div></div>
{{CLOSED_LOST_STORY_CARD}}
<div class='card'><h2>Gong Call Summary</h2>{{GONG_CARDS}}</div>
<div class='card'><h2>Activity Timeline</h2><table class='tl'><thead><tr><th>When</th><th>Type</th><th>Summary</th></tr></thead><tbody>{{TIMELINE_ROWS}}</tbody></table>
<div style='font-size:11px;color:#7A8699;margin-top:8px'>{{TIMELINE_NOTE}}</div></div>
<div class='two'>
<div class='card'><h2>Risks &amp; Open Items</h2><ul class='tight'>{{RISKS}}</ul></div>
<div class='card'><h2>Recommended Next Steps</h2><ul class='tight'>{{NEXT_STEPS}}</ul></div>
</div>
<div class='foot'>{{FOOTER}}</div>
</div></body></html>
```

### Per-part snippets (use these exact class structures)
- **Snapshot card** (×4): `<div class="sc"><div class="sc-lab">{label}</div><div class="sc-val {stage-lost if lost Stage card}">{value}</div></div>`
- **Deal Snapshot row**: `<tr><td class="k">{key}</td><td class="v">{value}</td></tr>`
- **Contact card**: `<div class="ct {buyer|broker|internal}"><div class="ct-nm">{name}</div><div class="ct-role">{role}</div><div class="ct-note">{note}</div></div>`
- **Closed-Lost Story card** (lost only): `<div class="card"><h2>Closed-Lost Story</h2><div class="narr">{narrative}</div><div class="cl-quote">{buyer quote + attribution}</div><div class="cl-line"><b>CRM says:</b> {category/reason/notes}</div><div class="cl-flag"><b>⚠ …</b> {evidence-vs-field reconciliation}</div><div class="cl-win"><b>Win-back angle.</b> {plays}</div></div>`
- **Gong card**: `<div class="gong"><div class="gong-h">{title}</div><div class="gong-meta">{date · duration · attendees}</div><div class="gong-brief">{brief}</div><ul class="gong-pts">{key points}</ul><div class="gong-next"><b>Next steps (at the time):</b> {…}</div><a class="gong-link" href="{gong url}" target="_blank" rel="noopener">Open Gong recording →</a><div class="gong-attr">Summary auto-generated by Gong.</div></div>`
- **Timeline row**: `<tr><td class="tl-when">{when}</td><td class="tl-type">{type}</td><td class="tl-sum">{summary}</td></tr>`
- **Risks / Next Steps bullet**: `<li>{text}</li>`


## Notes
- Gong summaries are auto-generated by Gong — attribute them.
- Watch email payload size; itemize subjects/dates, not bodies.
- Keep the honest data-quality flags visible in Risks & Open Items.
- The closed-lost loss reason comes from the calls/emails first; the CRM `closed_lost_category` is a cross-check, and any disagreement is flagged in the Closed-Lost Story.
