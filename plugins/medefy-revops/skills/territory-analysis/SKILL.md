---
name: "territory-analysis"
description: "Generate a Medefy territory or state analysis: a branded HTML dashboard plus a print-perfect portrait PDF, with a metro-density map, a per-state breakdown, linking to HubSpot saved views. Covers clients, prospects, broker firms, the broker-program breakdown (signed, closing, referring, passive, dormant), program-level mix (Bronze/Silver/Gold/Platinum), and the top referring and top closing brokers over the last 12 months. Use for a territory analysis, state analysis, or broker landscape for a region or state."
---

# Territory Analysis

Generate a territory or state analysis of Medefy's book of business and broker program. Output a **Medefy-branded HTML dashboard** plus a **print-perfect portrait PDF**, linking to the **HubSpot saved views** for record-level detail. Anyone can run it: "run a territory analysis on the Southwest" or "territory analysis for TX, OK, NM."

## When to use
"Run a territory analysis on [territory]," "[state] territory/broker analysis," "broker landscape for [region/state]," or a breakdown of clients / prospects / broker firms and the broker program for a region.

## Inputs
- A **territory name** (resolved via TERRITORY_MAP), OR one or more **U.S. state codes** (2-letter).
If a territory isn't in TERRITORY_MAP, ask for its states. Always echo the resolved state list before running.

## CONFIG — all business logic centralized here

**Geo fields:**
- Company state: `u_s_state` (label "U.S. State", 2-letter). Do NOT use `state` or `medefy_territory`.
- Contact state: `shipping_state__united_states_` (label "U.S State"). Do NOT use `state`/`hs_state_code`.
- Company city: `city`. Contact city: `city`. (Used for the metro-density map.)

**TERRITORY_MAP (4-territory scheme; rep for reference):**
- Northeast (Mike LaMura): CT, MA, ME, NH, NJ, NY, RI, VT
- Southeast (Haley Sullivan): AL, DC, DE, FL, GA, KY, MD, MS, NC, PA, SC, TN, VA, WV
- Southwest (Walid Natafgy): AR, AZ, CA, HI, LA, NM, NV, OK, TX, UT
- Midwest (Kelsey LeGrande): AK, CO, IA, ID, IL, IN, KS, MI, MN, MO, MT, ND, NE, OH, OR, SD, WA, WI, WY

**Company classification:**
- Client = `contract_status` IN (Active, Implementation)
- Former client (report separately) = `contract_status` IN (Pending Termination, Churned)
- Prospect = employer-type company with no contract. Employer-type = `wip__company_type` IN (Employer, "Employers, Large", "Employers, Jumbo", "Employers - SLG", "Employer, Prospect"); fallback `employer__broker__other` = 'Employer'. (Intentionally broad — includes the imported employer universe.)
- Broker firm = `employer__broker__other` = 'Broker' (cross-checks to `wip__company_type` = 'Broker Firm').
- If the company-type field changes, update only this block.

**Broker contacts — always filter `active_employee` = true:**
- broker = `broker_program_status` present; signed = `date_of_broker_contract_signed` present.
- `broker_program_status`: Closing, Referring, Passive, Dormant, Disqualified.
- `broker_program_level`: Bronze, Silver, Gold, Platinum (only a subset carry a level).

**Deal fields:** `amount`, `hs_arr`, `closedate`, `createdate`, `hs_is_closed_won`. **Window:** trailing 12 months.

**Branding:** theme with the **`medefy-brand`** skill. Banner = the **Dark header banner** with the **reversed Medefy logo** (`medefy-logo-reversed.png` — read the file and base64-embed at build time, ~40px; if not in the session, ask the user to attach it), a cyan uppercase eyebrow, white title, and the **scope/metadata subtitle in pink `#FF57B0`**. **HubSpot link chips use the pink gradient** `linear-gradient(135deg,#FF1EA0,#FF6FB5)`. Status colors: Closing #1EC8F0 / Referring #FF1EA0 / Passive #2882FA / Dormant grey. KPI numbers and bar fills primary #2882FA. Page background is the light tint `#F4F6F9`; cards are white with a soft shadow so they stand out.

**METRO-DENSITY MAP (Version B — color darkness = density):** a card placed **directly under the KPI row** titled "Where They Are — Metro Density", holding **two side-by-side maps** of the territory's states only: **Clients** (blue) and **Referring & Closing Brokers** (pink). Each metro is **one fixed-size dot (r≈7)** whose **color is interpolated light→dark by count** (`fill = lerp(light, dark, count/max)`, white stroke): Clients `#CFE0FF`→`#0B3D91`, Brokers `#FFD3EC`→`#A10E63`. Label the **top 4 metros** ("City N") above their dots, add a small **"fewer → more" gradient legend**, and beneath each map a **"Largest metros" note** listing the top 4 with counts (e.g., "Largest client metros: Oklahoma City (12) · Tulsa (10) · Houston (7) · Dallas (6)"). If some records have no city, add "N clients have no city on record."

**HUBSPOT_VIEWS (base saved views; each has the type filter baked in + a State (U.S. State) quick-filter):**
- Clients: https://app.hubspot.com/contacts/8313828/objects/0-2/views/69622042/list
- Prospects: https://app.hubspot.com/contacts/8313828/objects/0-2/views/69623050/list
- Broker Firms: https://app.hubspot.com/contacts/8313828/objects/0-2/views/69623593/list
- Broker Contacts: https://app.hubspot.com/contacts/8313828/objects/0-1/views/65872132/list  (filtered to Active Employee = Yes)
HubSpot can't pre-apply state filters via URL, so the dashboard links to these base views and instructs the user to select the territory's states in the State quick-filter.

**KPI cards (label — description):** Clients — "Active or Implementation contracts"; Prospects — "Employer-type, no contract"; Broker Firms — "Companies typed Broker"; Broker Contacts — "Active brokers".

## Procedure
1. **Resolve scope → state list** (TERRITORY_MAP or given states). Echo it.
2. **Company counts** (`u_s_state` IN states; read `total` from `search_crm_objects`): Clients; Prospects; Broker firms; (optional) Former clients.
3. **Broker program breakdown** (`shipping_state__united_states_` IN states AND `active_employee`=true): total brokers; signed; counts by status. Program-level cross-tab: `SELECT broker_program_level, broker_program_status, COUNT(*) FROM CONTACT WHERE shipping_state__united_states_ IN (...) AND active_employee='true' AND broker_program_level IS NOT NULL GROUP BY broker_program_level, broker_program_status` (only a subset carry a level).
4. **Top referring broker (12mo)** — candidates = brokers in scope (shipping_state + active) with status IN (Referring, Closing); for each, `search_crm_objects` DEAL with `associatedWith` contacts = broker id and `createdate` >= today−12mo; sum `amount` = referred pipeline $. Rank (association-loop; cross-object SQL aggregate is unsupported). **Do not flag/discount a broker for a stale deal — a referral counts regardless of outcome.**
5. **Top closing broker (12mo)** — for each broker in scope, associated DEALs with `hs_is_closed_won`=true and `closedate` >= today−12mo; sum `hs_arr` (fallback `amount`) = Closed-Won ARR. Rank.
6. **Per-state breakdown (FULL-TERRITORY runs only — skip when the input is a single state).** One GROUP BY per metric so each state's numbers tie to the definitions above:
   - Clients: `SELECT u_s_state, COUNT(*) FROM COMPANY WHERE u_s_state IN (...) AND contract_status IN ('Active','Implementation') GROUP BY u_s_state`
   - Prospects: `... AND wip__company_type IN (<employer types>) AND contract_status IS NULL GROUP BY u_s_state`
   - Broker firms: `... AND employer__broker__other='Broker' GROUP BY u_s_state`
   - Active brokers: `SELECT shipping_state__united_states_, COUNT(*) FROM CONTACT WHERE shipping_state__united_states_ IN (...) AND broker_program_status IS NOT NULL AND active_employee='true' GROUP BY shipping_state__united_states_`
   - Referring/Closing brokers: `... AND broker_program_status IN ('Referring','Closing') AND active_employee='true' GROUP BY shipping_state__united_states_`
   Fill zeros for states absent from a result. Note per-state sums can differ from the headline totals by a few records (queries run moments apart / null-vs-empty semantics).
7. **Metro-density map data + render** (works for ANY territory):
   a. **Pull city+state** for the two layers: **Clients** = companies with `contract_status` IN (Active, Implementation) and `u_s_state` IN states — return `city`, `u_s_state`; **Brokers** = contacts with `broker_program_status` IN (Referring, Closing), `active_employee`='true', `shipping_state__united_states_` IN states — return `city`, state. Tally counts per (city, state). Drop blank cities and keep a count of how many were dropped ("no city on record").
   b. **Geocode offline** with Python `geonamescache` (`pip install geonamescache --break-system-packages`): for each (city, state), find the US city whose name matches and whose state = that state; if several, take the most populous → `[lon, lat]`. Cities that can't be matched are skipped and counted. Emit two JSON lists `{label, lon, lat, count}` (clients, brokers).
   c. **Render** with Node using `us-atlas` (states-10m TopoJSON), `topojson-client`, and `d3-geo` (install into a temp dir and run from there — ESM ignores NODE_PATH). Map the territory's 2-letter codes to state **names**, `feature()` the TopoJSON, filter to those state features, `geoAlbersUsa().fitExtent([[8,8],[W-8,H-8]], {type:'FeatureCollection',features})`, draw state paths (`#EDF2F8` fill, `#C4D0DE` stroke). Plot each metro with the **Version B color-density** style from CONFIG (fixed r≈7, `fill=lerp(light,dark,count/max)`, white stroke), label the top 4, add the gradient legend. Write an HTML fragment (two maps + notes) to a temp file.
   d. Keep the fragment for injection under the KPI row in step 8.
8. **Assemble the dashboard DATA as a JSON object** (schema in DATA SCHEMA below) — you produce ONLY data, never HTML/CSS. Section order for reference: eyebrow + title + pink scope/date subtitle; a **KPI row of 4 cards** using the labels & descriptions in CONFIG (Broker Contacts description = "Active brokers"), where **each card carries a small pink-gradient "HubSpot | {category}" link** to its saved view; a **single State-filter disclaimer line beneath the KPI row** (lists the territory's states; notes Broker Contacts = Active Employee Yes) — no separate "Open in HubSpot" section; **then the METRO-DENSITY MAP card (step 7 fragment) directly beneath that disclaimer**; top-referring & top-closing headline cards + leaderboards (top 10); **a "By State" table (states × Clients, Prospects, Broker Firms, Active Brokers, Referring/Closing, with a territory-total row) — include ONLY on full-territory runs**; Broker Program Breakdown bars; Broker Program Level chart; Definitions & Caveats. Include a "Save as PDF" button (`window.print()`, hidden via `@media print`). **Print CSS = A4 PORTRAIT**, `print-color-adjust:exact`; **tint the whole page the dashboard background `#F4F6F9`** (set it on BOTH `@page` and `html,body`) so the white cards stand out, and **keep card `box-shadow`s** for lift; inside `@media print` reflow the KPI grid to 2-up (`repeat(2,1fr)`), keep the two maps side-by-side, and stack the other two-column sections (`.cols,.head2 → 1fr`).
9. **Deliver:** save the dashboard HTML, then generate a **print-perfect PDF** from it with **weasyprint** (`pip install weasyprint --break-system-packages` if needed, then `weasyprint.HTML(html_path).write_pdf(pdf_path)` — renders the tinted page, dark banner, inline map SVGs, grids, gradients, pink accents and embedded logo faithfully; use `pdftoppm -png` on a page to verify it's portrait, tinted, and uncramped). **Present BOTH the HTML and the PDF.** The in-page "Save as PDF" button works in a full browser but is usually blocked in the in-app preview, so always ship the generated PDF. (An Excel workbook via the `xlsx` skill is optional — build only if asked.)


## Render with the bundled builder (do NOT hand-write HTML/CSS)
The dashboard's look is produced by a fixed Python builder — the model supplies only data, so the design can never drift.
1. Write the **Dashboard builder** code block below **verbatim** to `build_dashboard.py` (do not modify it — the logo, CSS, and layout are baked in).
2. Write your assembled data (DATA SCHEMA below) to `data.json`.
3. Run: `pip install weasyprint --break-system-packages` (if needed), then `python3 build_dashboard.py data.json "Territory Analysis Dashboard - {Territory} - {date}.html" "Territory Analysis Dashboard - {Territory} - {date}.pdf"`.
4. Verify with `pdftoppm -png` and **present BOTH** the HTML and the PDF.

## DATA SCHEMA (the JSON you pass to the builder)
```
{
  "territory": "Southwest",                      // or a single state name like "Utah"
  "subtitle": "Rep: {rep} &nbsp;•&nbsp; {STATES} &nbsp;•&nbsp; Run {date} &nbsp;•&nbsp; trailing-12-month rankings",
  "kpis": [ ["97","Clients","Active or Implementation contracts","{clients_view_url}"], ["6,027","Prospects","Employer-type, no contract","{prospects_url}"], ["768","Broker Firms","Companies typed “Broker”","{firms_url}"], ["5,602","Broker Contacts","Active brokers","{brokers_url}"] ],
  "disclaimer": "Pink links open your saved HubSpot view. Apply the <b>State</b> quick-filter … The Broker Contacts view is filtered to Active Employee = Yes.",
  "map_section_html": "<div class='card mapsec'>…the step-7 fragment…</div>",
  "headline_ref": {"label":"Top referring broker · 12mo","name":"…","st":"TX","amount_line":"$800,000 referred pipeline","meta":"…"},
  "headline_clo": {"label":"Top closing broker · 12mo","name":"…","st":"TX","amount_line":"$110,250 Closed-Won ARR","meta":"…"},
  "bystate": {"include": true, "total_label":"Southwest total", "note":"…", "order":["TX","OK",…], "rows":{"TX":[clients,prospects,firms,active_brokers,ref_closing], …}},
  "program": [["Dormant",5267],["Passive",297],["Referring",24],["Closing",14]], "pmax": 5267,
  "pills": {"signed":211,"engaged":335,"dormant":5267},
  "levels": [["Bronze",[["Passive",7]]],["Silver",[["Referring",1]]],["Gold",[["Closing",1]]],["Platinum",[["Dormant",150],["Passive",27],["Referring",3]]]], "lmax": 180,
  "level_note": "…",
  "ref": [["name","ST",800000], … up to 10],
  "clo": [["name","ST",110250], … up to 10],
  "definitions_html": "…prose with <b>bold</b>…",
  "footer": "Generated {date} · Medefy RevOps · Territory Analysis skill"
}
```
Notes on the schema:
- **Single-state run:** set `"bystate": {"include": false}` (the By State card is omitted). Subtitle can read `State: {ST} …`.
- `kpis` are 4 rows of `[number, label, description, hubspot_view_url]`; the builder renders the pink "HubSpot | {label}" chip.
- `program` order Dormant→Passive→Referring→Closing; `levels` order Bronze→Silver→Gold→Platinum; the builder colors and sizes the bars.
- `map_section_html` is the fragment produced in step 7 (must use the `card mapsec / mapgrid / mapcol / maptitle / dlgd / dbar / dend / mnote` classes so the builder's CSS styles it).
- If there are no referring or closing brokers, pass an empty `ref`/`clo` list and phrase the headline `amount_line` as "None / No … in the last 12 months".

## Dashboard builder — write this to `build_dashboard.py` VERBATIM
```python
#!/usr/bin/env python3
# Territory Analysis deterministic builder. Usage: python build_dashboard.py data.json out.html out.pdf
import json, sys, html as _h
LOGO = "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"
SC = {"Closing":"#1EC8F0","Referring":"#FF1EA0","Passive":"#2882FA","Dormant":"#9AA6B2"}
def money(v):
    try: return "${:,.0f}".format(float(v))
    except Exception: return str(v)
CSS = r"""
:root{--primary:#2882FA;--blue2:#1EAAFA;--cyan:#1EC8F0;--magenta:#FF1EA0;--pink:#FF57B0;--ink:#202035;--grey:#E9EBEE;--bg:#F4F6F9;--card:#FFFFFF;--muted:#7A8699;--line:#E4E8EE;}
*{box-sizing:border-box}
body{margin:0;background:var(--bg);color:var(--ink);font-family:'Inter','Segoe UI',system-ui,Arial,sans-serif;-webkit-font-smoothing:antialiased}
.wrap{max-width:1180px;margin:0 auto;padding:0 22px 40px}
.top{position:relative;overflow:hidden;color:#fff;padding:30px 0;margin-bottom:24px;background:radial-gradient(105% 150% at 6% 135%, rgba(30,200,240,.42), rgba(30,200,240,0) 45%),radial-gradient(95% 150% at 102% -25%, rgba(150,80,240,.48), rgba(150,80,240,0) 52%),radial-gradient(70% 130% at 84% 130%, rgba(255,30,160,.22), rgba(255,30,160,0) 55%),linear-gradient(115deg,#1b1c3c 0%,#181934 55%,#211a3c 100%);}
.top .inner{max-width:1180px;margin:0 auto;padding:0 22px;display:flex;align-items:center;justify-content:space-between;gap:20px;flex-wrap:wrap;position:relative}
.brandmark{display:flex;align-items:center;gap:16px}
.logo-img{height:40px;width:auto;display:block}
.eyebrow{font-size:11px;font-weight:800;letter-spacing:1.5px;color:var(--cyan);text-transform:uppercase;margin-bottom:6px}
h1{font-size:23px;margin:0;font-weight:800}
.sub{color:var(--pink);font-size:13px;margin-top:5px;font-weight:600}
.toolbar{display:flex;gap:10px}
.btn{border:0;border-radius:9px;padding:10px 16px;font-weight:650;font-size:13px;cursor:pointer;text-decoration:none;display:inline-flex;align-items:center;gap:7px}
.btn.primary{background:#fff;color:var(--ink)}
.grid-kpi{display:grid;grid-template-columns:repeat(4,1fr);gap:16px;margin-bottom:8px}
.kpi{background:var(--card);border:1px solid var(--line);border-radius:14px;padding:18px 20px;box-shadow:0 1px 3px rgba(32,32,53,.06);display:flex;flex-direction:column}
.kpi-num{font-size:34px;font-weight:800;line-height:1;color:var(--primary)}
.kpi-lab{font-weight:700;margin-top:8px}.kpi-sub{color:var(--muted);font-size:12px;margin-top:3px}
.hslink{align-self:flex-start;margin-top:12px;background:linear-gradient(135deg,#FF1EA0,#FF6FB5);color:#fff;font-size:11px;font-weight:700;text-decoration:none;padding:5px 11px;border-radius:7px;box-shadow:0 2px 6px rgba(255,30,160,.28)}
.kpihint{font-size:11px;color:var(--muted);margin:2px 0 20px}
.cols{display:grid;grid-template-columns:1fr 1fr;gap:18px}
.card{background:var(--card);border:1px solid var(--line);border-radius:14px;padding:18px 20px;box-shadow:0 1px 3px rgba(32,32,53,.06);margin-bottom:18px}
.card h2{font-size:14px;margin:0 0 14px;text-transform:uppercase;letter-spacing:.6px;color:var(--muted)}
.subh{font-size:13px;font-weight:750;margin:18px 0 6px}.hint{font-size:11px;color:var(--muted);margin:6px 0 0}
.head2{display:grid;grid-template-columns:1fr 1fr;gap:18px;margin-bottom:18px}
.hl{background:var(--card);border:1px solid var(--line);border-left:6px solid var(--magenta);border-radius:12px;padding:16px 18px;box-shadow:0 1px 3px rgba(32,32,53,.06)}
.hl.win{border-left-color:var(--cyan)}
.hl .lab{color:var(--muted);font-size:12px;text-transform:uppercase;letter-spacing:.5px}
.hl .val{font-size:18px;font-weight:800;margin-top:4px}.hl .amt2{color:var(--primary);font-weight:800}
.hl .meta{font-size:12px;color:var(--muted);margin-top:6px}
.brow{display:grid;grid-template-columns:84px 1fr 66px;align-items:center;gap:12px;margin-bottom:11px}
.blab{font-size:13px;font-weight:600}.btrack{background:var(--grey);border-radius:8px;height:16px;overflow:hidden}
.bfill{height:100%;border-radius:8px}.bval{text-align:right;font-variant-numeric:tabular-nums;font-size:13px;color:var(--muted)}
.pill{display:inline-block;background:#EAF2FF;color:var(--primary);border-radius:20px;padding:6px 12px;font-size:12px;font-weight:650;margin:6px 6px 0 0}
.pill.grey{background:#F0F2F5;color:var(--muted)}
.legend{display:flex;gap:14px;flex-wrap:wrap;margin:2px 0 12px}
.lg{font-size:11px;color:var(--muted);display:flex;align-items:center;gap:5px}
.dot{width:10px;height:10px;border-radius:3px;display:inline-block}
.lvrow{display:grid;grid-template-columns:70px 1fr 42px;align-items:center;gap:12px;margin-bottom:4px}
.lvname{font-size:13px;font-weight:700}.lvtrack{display:flex;height:16px;background:var(--grey);border-radius:8px;overflow:hidden}
.seg{height:100%}.lvtot{text-align:right;font-weight:700;font-variant-numeric:tabular-nums;font-size:13px}
.lvdetail{grid-column:2 / span 2;font-size:11px;color:var(--muted);margin:0 0 8px}
.tbl{width:100%;border-collapse:collapse;font-size:12px}
.tbl th{text-align:right;color:var(--muted);font-weight:700;text-transform:uppercase;letter-spacing:.4px;font-size:10px;padding:7px 8px;border-bottom:2px solid var(--line)}
.tbl th.l,.tbl td.l{text-align:left}
.tbl td{padding:7px 8px;border-bottom:1px solid var(--line);text-align:right;font-variant-numeric:tabular-nums}
.tbl td.stname{font-weight:700;color:var(--ink)}
.tbl tr.tot td{font-weight:800;border-top:2px solid var(--line);border-bottom:none;color:var(--ink)}
.lrow{display:grid;grid-template-columns:24px 1.5fr 1fr auto;align-items:center;gap:10px;padding:7px 0;border-bottom:1px solid var(--line)}
.lrow:last-child{border-bottom:0}.rank{color:var(--muted);font-weight:700;font-size:13px;text-align:center}
.who{display:flex;align-items:center;gap:8px;min-width:0}
.nm{font-weight:650;font-size:13px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
.st{color:var(--muted);font-size:11px;border:1px solid var(--line);border-radius:6px;padding:1px 6px}
.ltrack{background:var(--grey);border-radius:6px;height:9px;overflow:hidden}
.lfill{height:100%;background:linear-gradient(90deg,var(--primary),var(--cyan));border-radius:6px}
.amt{font-variant-numeric:tabular-nums;font-weight:700;font-size:13px;text-align:right;white-space:nowrap}
.mapsec .mapgrid{display:grid;grid-template-columns:1fr 1fr;gap:22px}
.maptitle{font-size:13px;font-weight:800;text-transform:uppercase;letter-spacing:.5px;margin-bottom:4px}
.mapcol svg{display:block;background:#F8FAFD;border:1px solid var(--line);border-radius:10px;padding:4px}
.dlgd{display:flex;align-items:center;gap:7px;margin:8px 0 2px;font-size:10px;font-weight:700;color:var(--muted);text-transform:uppercase;letter-spacing:.4px}
.dbar{width:120px;height:9px;border-radius:5px;display:inline-block}
.dend{color:var(--muted)}
.mnote{font-size:12px;color:var(--ink);font-weight:600;margin:6px 0 0;line-height:1.5}
.notes{font-size:12px;color:var(--muted);line-height:1.55}.notes b{color:var(--ink)}
.foot{color:var(--muted);font-size:11px;margin-top:18px;text-align:center}
@media (max-width:900px){.grid-kpi{grid-template-columns:repeat(2,1fr)}.cols,.head2{grid-template-columns:1fr}}
@media print{@page{size:A4 portrait;margin:11mm;background:#F4F6F9}html,body{background:#F4F6F9;-webkit-print-color-adjust:exact;print-color-adjust:exact}.toolbar{display:none!important}.top{margin-bottom:16px}.wrap{max-width:none}.grid-kpi{grid-template-columns:repeat(2,1fr)}.cols,.head2{grid-template-columns:1fr}.card,.kpi,.hl{break-inside:avoid;box-shadow:0 1px 4px rgba(32,32,53,.10)}.wrap{padding-bottom:0}.mapsec .mapgrid{grid-template-columns:1fr 1fr}}
"""
def kpi_html(kpis):
    o=""
    for k in kpis:
        n,l,s,url=k
        o+=f'<div class="kpi"><div class="kpi-num">{n}</div><div class="kpi-lab">{l}</div><div class="kpi-sub">{s}</div><a class="hslink" href="{url}" target="_blank" rel="noopener">HubSpot | {l}</a></div>'
    return o
def prog_html(prog,pmax):
    o=""
    for lab,val in prog:
        w=max(2,round(val/pmax*100)); o+=f'<div class="brow"><div class="blab">{lab}</div><div class="btrack"><div class="bfill" style="width:{w}%;background:{SC.get(lab,"#9AA6B2")}"></div></div><div class="bval">{val:,}</div></div>'
    return o
def legend_html():
    return '<div class="legend">'+"".join(f'<span class="lg"><span class="dot" style="background:{SC[k]}"></span>{k}</span>' for k in ["Closing","Referring","Passive","Dormant"])+'</div>'
def level_html(levels,lmax):
    o=""
    for name,segs in levels:
        tot=sum(v for _,v in segs); seg="".join(f'<div class="seg" style="width:{v/lmax*100}%;background:{SC.get(st,"#9AA6B2")}"></div>' for st,v in segs); det=" · ".join(f'{v} {st}' for st,v in segs)
        o+=f'<div class="lvrow"><div class="lvname">{name}</div><div class="lvtrack">{seg}</div><div class="lvtot">{tot}</div><div class="lvdetail">{det}</div></div>'
    return o
def bystate_html(bs):
    heads=["State","Clients","Prospects","Broker Firms","Active Brokers","Referring / Closing"]
    th="".join(f'<th class="{ "l" if i==0 else "" }">{h}</th>' for i,h in enumerate(heads))
    body=""; tot=[0,0,0,0,0]
    for st in bs["order"]:
        c,p,f,b,rc=bs["rows"][st]
        tot=[tot[0]+c,tot[1]+p,tot[2]+f,tot[3]+b,tot[4]+rc]
        body+=f'<tr><td class="l stname">{st}</td><td>{c}</td><td>{p:,}</td><td>{f}</td><td>{b:,}</td><td>{rc}</td></tr>'
    label=bs.get("total_label","Total")
    body+=f'<tr class="tot"><td class="l">{label}</td><td>{tot[0]}</td><td>{tot[1]:,}</td><td>{tot[2]}</td><td>{tot[3]:,}</td><td>{tot[4]}</td></tr>'
    return f'<table class="tbl"><thead><tr>{th}</tr></thead><tbody>{body}</tbody></table>'
def board_html(data,um):
    o=""
    for i,(nm,st,v) in enumerate(data,1):
        w=max(4,round(v/um*100)) if um else 4
        o+=f'<div class="lrow"><div class="rank">{i}</div><div class="who"><span class="nm">{_h.escape(nm)}</span><span class="st">{st}</span></div><div class="ltrack"><div class="lfill" style="width:{w}%"></div></div><div class="amt">{money(v)}</div></div>'
    return o
def hl_html(h,win):
    cls="hl win" if win else "hl"
    lab=h["label"]; st=f' <span style="color:var(--muted);font-weight:600">({h["st"]})</span>' if h.get("st") else ""
    return (f'<div class="{cls}"><div class="lab">{lab}</div><div class="val">{_h.escape(h["name"])}{st}</div>'
            f'<div class="val amt2">{h["amount_line"]}</div><div class="meta">{h.get("meta","")}</div></div>')

D=json.load(open(sys.argv[1],encoding="utf-8"))
parts=[]
parts.append("<!doctype html><html lang='en'><head><meta charset='utf-8'><meta name='viewport' content='width=device-width,initial-scale=1'>")
parts.append(f"<title>Territory Analysis — {D['territory']}</title><style>"+CSS+"</style></head><body>")
parts.append("<div class='top'><div class='inner'><div class='brandmark'>")
parts.append(f"<img class='logo-img' src='data:image/png;base64,{LOGO}' alt='Medefy'>")
parts.append(f"<div><div class='eyebrow'>Territory Analysis · Back-of-Book</div><h1>{D['territory']} Territory</h1>")
parts.append(f"<div class='sub'>{D['subtitle']}</div></div></div>")
parts.append("<div class='toolbar'><button class='btn primary' onclick='window.print()'>⬇ Save as PDF</button></div></div></div>")
parts.append("<div class='wrap'>")
parts.append("<div class='grid-kpi'>"+kpi_html(D["kpis"])+"</div>")
parts.append(f"<p class='kpihint'>{D['disclaimer']}</p>")
if D.get("map_section_html"): parts.append(D["map_section_html"])
parts.append("<div class='head2'>"+hl_html(D["headline_ref"],False)+hl_html(D["headline_clo"],True)+"</div>")
bs=D.get("bystate")
if bs and bs.get("include",True):
    parts.append("<div class='card'><h2>By State</h2>"+bystate_html(bs)+f"<p class='hint'>{bs.get('note','')}</p></div>")
prog=D["program"]; pmax=D.get("pmax",max(v for _,v in prog))
parts.append("<div class='card'><h2>Broker Program Breakdown</h2>"+legend_html()+prog_html(prog,pmax))
pl=D.get("pills")
if pl: parts.append(f"<div><span class='pill'>{pl['signed']} signed a broker contract</span><span class='pill grey'>{pl['engaged']} engaged vs {pl['dormant']:,} dormant</span></div>")
parts.append("<div class='subh'>Broker Program Level</div>")
parts.append(f"<p class='hint'>{D.get('level_note','')}</p>")
lv=D.get("levels",[]); lmax=D.get("lmax", max((sum(v for _,v in segs) for _,segs in lv), default=1))
parts.append(level_html(lv,lmax)+"</div>")
parts.append("<div class='cols'>")
parts.append("<div class='card'><h2>Top Referring Brokers — referred pipeline $ (12mo)</h2>"+board_html([tuple(x) for x in D['ref']], D['ref'][0][2] if D['ref'] else 0)+"</div>")
parts.append("<div class='card'><h2>Top Closing Brokers — Closed-Won ARR (12mo)</h2>"+board_html([tuple(x) for x in D['clo']], D['clo'][0][2] if D['clo'] else 0)+"</div>")
parts.append("</div>")
parts.append("<div class='card'><h2>Definitions &amp; Caveats</h2><div class='notes'>"+D["definitions_html"]+"</div></div>")
parts.append(f"<div class='foot'>{D.get('footer','')}</div>")
parts.append("</div></body></html>")
HTML="".join(parts)
open(sys.argv[2],"w",encoding="utf-8").write(HTML)
try:
    from weasyprint import HTML as WH
    WH(sys.argv[2]).write_pdf(sys.argv[3])
    print("OK html+pdf", len(HTML))
except Exception as e:
    print("HTML written; PDF step failed:", e)

```


## Notes
- Counts use the search `total` — watch pagination.
- Co-brokered deals are credited to every associated broker (document this).
- Prospect count is intentionally broad; state it plainly.
- Map geocoding is best-effort: report how many records had no city and how many cities couldn't be geocoded, so the map is understood as directional, not exact.
