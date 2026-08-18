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
8. **Build the dashboard** (`medefy-brand` dark banner + reversed logo): eyebrow + title + pink scope/date subtitle; a **KPI row of 4 cards** using the labels & descriptions in CONFIG (Broker Contacts description = "Active brokers"), where **each card carries a small pink-gradient "HubSpot | {category}" link** to its saved view; a **single State-filter disclaimer line beneath the KPI row** (lists the territory's states; notes Broker Contacts = Active Employee Yes) — no separate "Open in HubSpot" section; **then the METRO-DENSITY MAP card (step 7 fragment) directly beneath that disclaimer**; top-referring & top-closing headline cards + leaderboards (top 10); **a "By State" table (states × Clients, Prospects, Broker Firms, Active Brokers, Referring/Closing, with a territory-total row) — include ONLY on full-territory runs**; Broker Program Breakdown bars; Broker Program Level chart; Definitions & Caveats. Include a "Save as PDF" button (`window.print()`, hidden via `@media print`). **Print CSS = A4 PORTRAIT**, `print-color-adjust:exact`; **tint the whole page the dashboard background `#F4F6F9`** (set it on BOTH `@page` and `html,body`) so the white cards stand out, and **keep card `box-shadow`s** for lift; inside `@media print` reflow the KPI grid to 2-up (`repeat(2,1fr)`), keep the two maps side-by-side, and stack the other two-column sections (`.cols,.head2 → 1fr`).
9. **Deliver:** save the dashboard HTML, then generate a **print-perfect PDF** from it with **weasyprint** (`pip install weasyprint --break-system-packages` if needed, then `weasyprint.HTML(html_path).write_pdf(pdf_path)` — renders the tinted page, dark banner, inline map SVGs, grids, gradients, pink accents and embedded logo faithfully; use `pdftoppm -png` on a page to verify it's portrait, tinted, and uncramped). **Present BOTH the HTML and the PDF.** The in-page "Save as PDF" button works in a full browser but is usually blocked in the in-app preview, so always ship the generated PDF. (An Excel workbook via the `xlsx` skill is optional — build only if asked.)

## Notes
- Counts use the search `total` — watch pagination.
- Co-brokered deals are credited to every associated broker (document this).
- Prospect count is intentionally broad; state it plainly.
- Map geocoding is best-effort: report how many records had no city and how many cities couldn't be geocoded, so the map is understood as directional, not exact.
