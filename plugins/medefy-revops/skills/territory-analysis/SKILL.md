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
8. **Build the dashboard using the EXACT VISUAL TEMPLATE below (reproduce verbatim).** For reference the section order is: eyebrow + title + pink scope/date subtitle; a **KPI row of 4 cards** using the labels & descriptions in CONFIG (Broker Contacts description = "Active brokers"), where **each card carries a small pink-gradient "HubSpot | {category}" link** to its saved view; a **single State-filter disclaimer line beneath the KPI row** (lists the territory's states; notes Broker Contacts = Active Employee Yes) — no separate "Open in HubSpot" section; **then the METRO-DENSITY MAP card (step 7 fragment) directly beneath that disclaimer**; top-referring & top-closing headline cards + leaderboards (top 10); **a "By State" table (states × Clients, Prospects, Broker Firms, Active Brokers, Referring/Closing, with a territory-total row) — include ONLY on full-territory runs**; Broker Program Breakdown bars; Broker Program Level chart; Definitions & Caveats. Include a "Save as PDF" button (`window.print()`, hidden via `@media print`). **Print CSS = A4 PORTRAIT**, `print-color-adjust:exact`; **tint the whole page the dashboard background `#F4F6F9`** (set it on BOTH `@page` and `html,body`) so the white cards stand out, and **keep card `box-shadow`s** for lift; inside `@media print` reflow the KPI grid to 2-up (`repeat(2,1fr)`), keep the two maps side-by-side, and stack the other two-column sections (`.cols,.head2 → 1fr`).
9. **Deliver:** save the dashboard HTML, then generate a **print-perfect PDF** from it with **weasyprint** (`pip install weasyprint --break-system-packages` if needed, then `weasyprint.HTML(html_path).write_pdf(pdf_path)` — renders the tinted page, dark banner, inline map SVGs, grids, gradients, pink accents and embedded logo faithfully; use `pdftoppm -png` on a page to verify it's portrait, tinted, and uncramped). **Present BOTH the HTML and the PDF.** The in-page "Save as PDF" button works in a full browser but is usually blocked in the in-app preview, so always ship the generated PDF. (An Excel workbook via the `xlsx` skill is optional — build only if asked.)

## EXACT VISUAL TEMPLATE — reproduce verbatim, do NOT restyle

The dashboard's look is **locked**. Build the HTML with the CSS and skeleton below **exactly** — same class names, same banner, same card styles. Only substitute data into the `{{PLACEHOLDERS}}`. Do not redesign, rename classes, reorder sections, recolor, or "improve" the styling. A run that changes the design is wrong even if the data is right.

**Non-negotiables (the ways past runs drifted):**
- **Banner** = a full-width dark bar at the very top (class `top`), NOT a rounded card. Logo left; cyan eyebrow `Territory Analysis · Back-of-Book`; white `<h1>{Territory} Territory</h1>`; pink `sub` line = `Rep: {name} • {STATES} • Run {date} • trailing-12-month rankings`; a white **Save as PDF** button top-right (`btn primary`, hidden in print). No separate bottom PDF button.
- **Section titles** use the `.card h2` style (small, UPPERCASE, grey) — including the map card ("By State", "Broker Program Breakdown", etc.). Do not make them large bold dark titles.
- **Top Referring / Top Closing headline cards** = subtle white cards with a colored **left border** (`hl` pink border; `hl win` cyan border), grey uppercase label, dark name, **blue** dollar value, grey meta. NOT big solid gradient-filled cards.
- **Leaderboards** = rows with a rank number, name + state chip, a **blue-gradient progress bar**, right-aligned dollar amount (`lrow`/`lfill`). NOT a plain table with firm sublines.
- **Broker Program Breakdown and Broker Program Level** live in **one stacked full-width card** (breakdown bars + pills, then a `Broker Program Level` subhead and the level rows **Bronze → Silver → Gold → Platinum**). NOT two side-by-side cards, NOT Platinum-first.
- **Definitions & Caveats** = one prose paragraph (`notes`), NOT bullets.
- Page tint `#F4F6F9`; white cards with soft shadow; KPI numbers and bars primary blue `#2882FA`.

### CSS (paste exactly into a `<style>` block)
```css
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
```

### HTML skeleton (fill the `{{PLACEHOLDERS}}`; keep everything else)
```html
<!doctype html><html lang='en'><head><meta charset='utf-8'>
<meta name='viewport' content='width=device-width,initial-scale=1'>
<title>Territory Analysis — {{TERRITORY}}</title><style>/* the CSS above */</style></head><body>
<div class='top'><div class='inner'><div class='brandmark'>
<img class='logo-img' src='{{LOGO_DATA_URI}}' alt='Medefy'>
<div><div class='eyebrow'>Territory Analysis · Back-of-Book</div><h1>{{TERRITORY}} Territory</h1>
<div class='sub'>Rep: {{REP}} &nbsp;•&nbsp; {{STATES}} &nbsp;•&nbsp; Run {{DATE}} &nbsp;•&nbsp; trailing-12-month rankings</div></div></div>
<div class='toolbar'><button class='btn primary' onclick='window.print()'>⬇ Save as PDF</button></div></div></div>
<div class='wrap'>
<div class='grid-kpi'>{{KPI_CARDS}}</div>
<p class='kpihint'>{{STATE_FILTER_DISCLAIMER}}</p>
{{MAP_SECTION}}
<div class='head2'>{{HEADLINE_REFERRING}}{{HEADLINE_CLOSING}}</div>
<div class='card'><h2>By State</h2>{{BYSTATE_TABLE}}<p class='hint'>{{BYSTATE_NOTE}}</p></div>
<div class='card'><h2>Broker Program Breakdown</h2>{{LEGEND}}{{PROGRAM_BARS}}
<div>{{PILLS}}</div>
<div class='subh'>Broker Program Level</div><p class='hint'>{{LEVEL_NOTE}}</p>{{LEVEL_ROWS}}</div>
<div class='cols'>
<div class='card'><h2>Top Referring Brokers — referred pipeline $ (12mo)</h2>{{LEADERBOARD_REFERRING}}</div>
<div class='card'><h2>Top Closing Brokers — Closed-Won ARR (12mo)</h2>{{LEADERBOARD_CLOSING}}</div>
</div>
<div class='card'><h2>Definitions &amp; Caveats</h2><div class='notes'>{{DEFINITIONS}}</div></div>
<div class='foot'>{{FOOTER}}</div>
</div></body></html>
```

### Per-part snippets (use these exact class structures)
- **KPI card** (×4): `<div class="kpi"><div class="kpi-num">{n}</div><div class="kpi-lab">{label}</div><div class="kpi-sub">{desc}</div><a class="hslink" href="{view_url}" target="_blank" rel="noopener">HubSpot | {label}</a></div>`
- **Headline card — referring**: `<div class="hl"><div class="lab">Top referring broker · 12mo</div><div class="val">{name} <span style="color:var(--muted);font-weight:600">({ST})</span></div><div class="val amt2">${amt} referred pipeline</div><div class="meta">{context}</div></div>` — closing card is identical but `class="hl win"` and label "Top closing broker · 12mo" / "${amt} Closed-Won ARR".
- **Leaderboard row** (×10): `<div class="lrow"><div class="rank">{i}</div><div class="who"><span class="nm">{name}</span><span class="st">{ST}</span></div><div class="ltrack"><div class="lfill" style="width:{pct}%"></div></div><div class="amt">${amt}</div></div>` (pct = value / max-in-list × 100, min 4).
- **Program bar row**: `<div class="brow"><div class="blab">{status}</div><div class="btrack"><div class="bfill" style="width:{pct}%;background:{statusColor}"></div></div><div class="bval">{count}</div></div>` — order Dormant, Passive, Referring, Closing; colors Closing `#1EC8F0` / Referring `#FF1EA0` / Passive `#2882FA` / Dormant `#9AA6B2`.
- **Level row**: `<div class="lvrow"><div class="lvname">{level}</div><div class="lvtrack">{colored segments}</div><div class="lvtot">{total}</div><div class="lvdetail">{n status · n status}</div></div>` — order Bronze, Silver, Gold, Platinum.
- **By State table**: `<table class="tbl">` with header cells `State, Clients, Prospects, Broker Firms, Active Brokers, Referring / Closing` (first `th`/`td` gets class `l`), a `tr` per state, and a final `<tr class="tot">` territory-total row.
- **MAP_SECTION** = the fragment from step 7 (the `card mapsec` with two `mapgrid` maps + notes).
- **LEGEND** = `<div class="legend">` with a `lg`/`dot` span per status (Closing, Referring, Passive, Dormant).
- **PILLS** = `<span class="pill">{signed} signed a broker contract</span><span class="pill grey">{engaged} engaged vs {dormant} dormant</span>`.


## Notes
- Counts use the search `total` — watch pagination.
- Co-brokered deals are credited to every associated broker (document this).
- Prospect count is intentionally broad; state it plainly.
- Map geocoding is best-effort: report how many records had no city and how many cities couldn't be geocoded, so the map is understood as directional, not exact.
