# SRPE Platform — Website, Data & API: Complete Research Handover

> **Date:** 2026-08-06
> **Author:** FDE Agent (PD-02 round)
> **Purpose:** Everything learned about the SRPE (Sales of First-hand Residential Properties Electronic Platform) — what the website is, how its data is organised, the discovered public JSON API, the mature solution design for a monitoring pipeline, and the pitfalls to avoid. This is the authoritative reference for anyone building on top of SRPE (PD-02, REA-02, or any future RE project).
> **One-line conclusion:** SRPE has a **public, token-free JSON API** that lets any system (n8n, a web page, a script) list all developments, list every document of a development, and **download the original PDFs directly** — no browser emulator, no scraping, no OCR needed for current documents.

---

## 1. What SRPE is

| Item | Detail |
|---|---|
| **Full name** | Sales of First-hand Residential Properties Electronic Platform 一手住宅物業銷售資訊網 |
| **URL** | `https://www.srpe.gov.hk/` |
| **Legal basis** | Residential Properties (First-hand Sales) Ordinance, Cap. 621 — vendors MUST publish sales documents here |
| **Operator** | Sales of First-hand Residential Properties Authority (SRPA), Hong Kong government |
| **What it hosts** | Every first-hand residential project's statutory sales documents: price lists, sales arrangements, registers of transactions, sales brochures, exam records |
| **Access** | Fully public, no login, no rate-limiting observed (30+ rapid requests in testing, all 200) |

### 1.1 Website technology (as of Aug 2026)

- **Frontend:** React SPA built with Vite — the HTML is a shell (`<div id="root">`) plus versioned bundles under `/opip/v0.0.54/js/*.js`
- **Backend:** Spring Boot-style JSON service at `/api/SrpeWebService/*` (error envelope `{"timestamp","status","error","path"}`)
- **Legacy note:** REA-02's Excel comments ("ASPX forms, session tokens, JS-heavy, needs a browser emulator / Puppeteer") describe the **OLD** website. The current site is a modern SPA + REST-like backend that is **directly callable**.
- **CORS:** explicitly allows `https://prefacecontent.github.io` (`Access-Control-Allow-Origin` echoed; `Allow-Credentials: true`) → a GitHub Pages frontend can call the API **directly from the browser**.
- **Auth:** the SPA reads a token from `localStorage.useragent.token` and sends it as `Authorization` — but the API **does not require it** (all endpoints tested anonymously).

---

## 2. The API — complete recipe

**Base URL:** `https://www.srpe.gov.hk/api/SrpeWebService`
**Method:** `POST` for everything
**Headers:** `Content-Type: application/json`
**Body shape:** `{"language":"en", ...params}` (also `zhhk` / `zhcn`)
**Response envelope:** `{"code": 0, "remarks": null, "status": null, "encrypted": false, "resultData": ...}` — `code == 0` means success; negative codes are errors (e.g. `-1009`/`-1012` → session/agreement issues; the SPA redirects to the disclaimer page).

### 2.1 Endpoints (all verified live)

| Endpoint | Purpose | Key body params | Returns |
|---|---|---|---|
| `POST /DevBldgSearch/getDevBldgSearchResult` | Search / list all developments | `{name: "LOHAS"}` (optional filter; omit = **all 521 projects**) | `resultData.list[]` — every project & phase with id, EN/CN names, phase, district, address, active flag |
| `POST /DevBldgSearch/getSelectedDevResult` | **Everything about one development** | `{devId: "11385"}` | `resultData.devInfoResp` — dev profile + `prices[]` + `salesArrangements[]` + `transactions[]` + `brochure` (+ `devpopupmsg`) |
| `POST /JsonList/getDevelopmentNameList` | Flat name list | `{}` | All development names (code/description pairs) |
| `POST /DistrictAreaSearch/getDistrictAreaSearchResult` | District-based search | — | Projects by district |
| `POST /download/downloadPrice` | **Download a price list PDF** | `{id, seq:"", devId}` | `application/octet-stream` PDF |
| `POST /download/downloadSalesArrangement` | **Download a sales arrangement PDF** | `{id, seq:"", devId}` | PDF |
| `POST /download/downloadTrx` | **Download a register of transactions PDF** | `{id, seq:"", devId}` | PDF |
| `POST /download/downloadBrochure` | **Download the sales brochure PDF** (23–40 MB) | `{id, seq, devId}` (`seq` from `partFiles[].seq`) | PDF |
| `POST /download/downloadExamRecord` | Download exam record | — | PDF |
| `POST /JsonList/getFrontPageAnnouncements` | Front page announcements | `{}` | Currently empty list (not useful) |
| `POST /ContactUs/contactUsSubmit` | Contact form | — | — |

### 2.2 Document record structure (from `getSelectedDevResult`)

```jsonc
// prices[] — price lists (價單). filename carries a date code; revisions have "R" suffix
{"id": "38579", "serialNo": "1", "dateOfPrinting": "2026-07-15T00:00:00.000+08:00",
 "file": {"id": "38579", "seq": null, "partNo": null, "fileName": "63817260715002PO.pdf",
          "fileSize": 1890948, "submissionTime": "2026-07-15T13:46:23.000+08:00"}}

// transactions[] — registers (成交紀錄冊). A NEW FILE replaces the old one each update
{"id": "104566", "updateDate": "2026-08-06", "updateDateTime": "2026-08-06T18:37:00.000+08:00",
 "file": {"id": "104566", "seq": null, "fileName": "63817260806019RT.pdf", "fileSize": 899779, ...}}

// salesArrangements[] — sales arrangements (銷售安排)
{"id": "26716", "serialNo": null, "dateOfPrinting": "2026-07-20T00:00:00.000+08:00",
 "file": {"fileName": "63817260720003SA.pdf", "fileSize": 497868, ...}}

// brochure — the sales brochure (樓書), one huge PDF split into parts
{"id": "29324", "dateOfPrint": "2026-07-30T00:00:00.000+08:00",
 "partFiles": [{"id": "29324", "seq": "1", "partNo": "0",
                "fileName": "1081626080500100.pdf", "fileSize": 22983397, "fullVersionInd": "Y", ...}]}
```

### 2.3 Change detection (the key to a monitoring pipeline)

Every document has a stable `id` plus `submissionTime`/`updateDateTime`/`dateOfPrinting`, and filenames embed date codes (`63817260806019RT.pdf` → register updated 2026-08-06). **A new version of a register = a new file id.** Therefore: *store the set of (type, id) per development; anything not seen before is NEW.*

---

## 3. The data itself

### 3.1 Document types (the analyst's weekly trio)

| Type | Chinese | What it contains | Funnel stage |
|---|---|---|---|
| Price List 價單 | 價單 | Unit catalogue: tower/floor/flat, saleable area (sq m + sq ft), list price, $/sq ft, payment plans, discounts; revisions marked "R" (e.g. No. 3R) | offered |
| Sales Arrangement 銷售安排 | 銷售安排 | Which units are offered on which sale date, sales method (ballot 抽籤 / FCFS 先到先得), time, venue + unit list | selling |
| Register of Transactions 成交紀錄冊 | 成交紀錄冊 | Chronological PASP/ASP table: date, unit, price, purchaser type (individual / local company), remarks incl. terminations 撻訂 | sold / terminated |

### 3.2 Real-document quality (verified Aug 2026)

- **All three document types + brochure are text-extractable** (pdftotext): e.g. a 900 KB register yields ~191 KB of clean bilingual text; a 1.9 MB price list yields ~84 KB. **No OCR stage needed** for currently published documents (the design spec's "blocking feasibility check" is now answered: PASS).
- **Bilingual** (English + Traditional Chinese), structured with part headers.
- **Layouts are dense, landscape tables** — the raw text interleaves columns, so extraction prompts must tolerate scrambled table text (Gemini handles this well; see §5 pitfalls).
- **Brochure (樓書):** 200+ pages A3, 23–40 MB, contains the floor plans 圖則. Floor-plan pages have a **text overlay** (unit labels like `T3 B` and dimension annotations are extractable per page — verified p26–p49 of THE STERLING brochure) → a deterministic "unit → page number" index is feasible. Where labels are image-only, humans view the PDF directly (user decision: **image verification stays with humans**).
- **Scale reality:** real phases are large — e.g. LOHAS PARK XIIIA has **1,284 units**; HIGHWOOD Phase 2 has **415 units**. A single LLM extraction call cannot reliably emit 1,284 unit rows → **chunked extraction is mandatory** for price lists (see §5).

### 3.3 Verified sample developments (Aug 2026)

| Project | devId | Documents | Notes |
|---|---|---|---|
| LOHAS PARK — LA MIRABELLE II (日出康城 XIIIA 期 海瑅灣II) | `11385` | 1 price list, 3 sales arrangements, 1 register | Register updated **2026-08-06** (same day as research) |
| HIGHWOOD Phase 2 (壹沐 第2期) | `11520` | 8 price lists (incl. "R" revisions), 7 arrangements, 1 register | 415 units, register updated 2026-08-06 |
| THE STERLING (叡璟) | `11524` | Brochure only (2026-08-05) | Brand-new launch, price list not yet out |
| LOHAS PARK full search ("LOHAS") | — | 21 phases returned | Name search works precisely |

---

## 4. The mature solution (as designed with the client)

### 4.1 Architecture

```
SRPE public API (token-free, CORS-open to GitHub Pages)
   │
   ├─ n8n workflow (hourly Schedule Trigger + manual POST /pd02/check)
   │    1. Refresh the 521-project developments list  → developments store
   │    2. For each watched development (user-managed watchlist):
   │         getSelectedDevResult → diff vs stored inventory → NEW documents
   │    3. NEW price list / arrangement / register → download PDF → AI pipeline
   │       (classify → chunked extract → reconcile funnel rule) → unit_records
   │       table + weekly report (all numbers computed in code, zero LLM arithmetic)
   │    4. Brochures are tracked but NOT downloaded/stored (23–40 MB)
   │       → frontend fetches them live from SRPE when the user clicks
   │
   ├─ Web API (n8n webhooks): watchlist CRUD, developments list, snapshot,
   │     batch-status, units, report
   │
   └─ GitHub Pages frontend (single console)
        • Project library: search box + browseable list of all 521 projects
          (from the developments store — "the list comes from our database")
        • Watchlist management (add/remove → hourly monitoring)
        • Per-project document timeline with 🆕 NEW badges
        • PDF viewer: fetches ANY document (incl. brochure with #page=N floor-plan
          jump) DIRECTLY from the SRPE API (CORS verified) → blob → iframe
        • Human verification: floor-plan PDF side-by-side with AI-extracted data
```

### 4.2 Key design decisions (client-approved)

| Decision | Rationale |
|---|---|
| **No mock data** — pipeline consumes only real SRPE documents | Demo credibility; SRPE is public and stable |
| **Watchlist = search box + full list**, both served from a stored developments database | Consistent UX; one data source; fast & offline-capable |
| **Hourly monitoring**; zero LLM cost when nothing changed | Change detection is a cheap diff on ids/timestamps |
| **PDFs are never stored** in n8n (static data must stay < ~1 MB; a 40 MB brochure would crash workers) — only metadata (id/fileName/size/dates) is kept | The API is always the source of truth; frontend pulls live |
| **Image/floor-plan verification by humans** — AI extracts structured data; the human eyeballs the floor plan PDF | Honest division of labour; AI does not guess at images |
| **One Gemini model node per AI node** (4 separate model nodes instead of 1 shared) | n8n's "Import from URL" keeps only the FIRST ai_languageModel edge per source node — 4 sources × 1 edge each survives import |
| **Chunked price-list extraction** (~15k chars/chunk with overlap, dedupe on unit_key) | Real price lists hold 400–1,300+ units; one LLM call cannot emit them all reliably |

### 4.3 Storage (demo database)

| Store | Content | Update cadence |
|---|---|---|
| `sd.pd02_v2.watchlist` | User-selected developments (devId + names/aliases) | On user action |
| `sd.pd02_v2.developments` | All 521 projects, trimmed fields (~60–80 KB) | Hourly (with the check) |
| `sd.pd02_v2.inventory` | Per dev: every document's metadata (id, fileName, size, dates) + `is_new` flags | Hourly diff |
| `sd.pd02_v2.check_log` | Last 50 check results (found/downloaded/processed, batch_id) | Hourly |
| `sd.pd02_v1.batches` | Batch records incl. units/review queue/stats/report HTML (24 h / 50 batch GC) | Per batch |
| `unit_records` Data Table | One row per unit, current status (shared with REA-02) | After each batch |

---

## 5. Pitfalls & hard-won lessons

### 5.1 SRPE API / data

1. **Don't trust REA-02's "ASPX/browser-emulator" comments** — they describe the old site. Test the API directly with curl before designing scraping.
2. **CORS is explicit per-origin** — `prefacecontent.github.io` is allowed; other origins must be verified (the API echoes `Access-Control-Allow-Origin`).
3. **Response envelope**: always check `code` (0 = success); negative codes exist for session/agreement edge cases.
4. **Brochure parts**: download needs `seq` from `partFiles[]` (e.g. `"1"`), not empty.
5. **Register updates REPLACE the file** — same `updateDate` can mean new `id` + new filename; always dedupe on (type, id), never on date alone.
6. **Price lists can be enormous** (1,284 units at LOHAS PARK) — chunk the text (~15k chars, 1k overlap) and extract per chunk; merge with unit_key dedupe (overlap may duplicate a row).
7. **The classified text of real price lists is scrambled** (landscape multi-column tables) — prompts must say "extract every unit row present, tolerate layout noise".

### 5.2 n8n deployment (this round's specific traps)

8. **"Import from URL" while a workflow editor is open MERGES into the open workflow** (name collisions get `1` suffixes) → always import into a **fresh empty workflow**.
9. **Import drops ai_languageModel edges** — all but the first per source node. Workaround: one model node per AI node (see §4.2). Verify edge count after import via the browser store (`pinia._s.get('workflowsList').$state.workflowsById[ID].connections`).
10. **Canvas drag-connect via CDP is unreliable** (connection line appears, drop doesn't commit; also at low zoom handles are ~8 px). Don't plan on it.
11. **`window.__content` must be built and tree-save'd on the SAME github.com page load** — any navigation/reload in between silently writes a 10-byte file. Verify with the contents API (size + JSON.parse) afterwards.
12. **tree-save 422 = stale HEAD sha** — refresh `commits/main` sha right before each tree-save.
13. **n8n `/rest/*` returns 401 from raw fetch** even in a logged-in tab (playbook rule) — the UI-only constraint still holds.
14. **Zoom shortcuts unreliable via CDP**; canvas wheel zoom in this environment is erratic — avoid depending on precise canvas geometry.

### 5.3 Reverse-engineering method (for future API changes)

1. Fetch the page shell → find the Vite bundle URL (`/opip/<ver>/js/index.*.js`).
2. Download the main bundle; grep for `baseURL:"/api/SrpeWebService"` → axios instance + interceptors (token from `localStorage.useragent`, but optional).
3. Download all lazy chunks (they're in the main bundle's import list); grep for `"/[A-Za-z]+/[A-Za-z0-9_]+"` → endpoint list.
4. Probe each endpoint with curl — behaviour beats source reading.

---

## 6. Assets & artifacts (where everything lives)

| Artifact | Location |
|---|---|
| Real SRPE sample PDFs (LOHAS PARK trio + THE STERLING brochure, 23 MB) | `PD02 Property Development/04_Mock_Data/Real_SRPE/` |
| This research report (Chinese original) | `PD02 Property Development/01_Docs/PD-02 SRPE 真實數據管道調研報告 — 2026-08-06.md` |
| Workflow v2.0 builder (source of truth) | `PD02 Property Development/scripts/build_workflow_v2.py` |
| Workflow v2.0 JSON (60 nodes, 4 model nodes) | `PD02 Property Development/03_n8n_Workflows/PD-02-n8n-workflow-v2.0.json` + GitHub `prefacecontent/pd02-srpe-reporting` (same path; re-upload if the 10-byte corruption persists) |
| Frontend (to be extended with the monitor view) | `PD02 Property Development/02_Frontend/index.html` |

---

## 7. Handover checklist for the next agent

- [ ] Re-upload `PD-02-n8n-workflow-v2.0.json` to GitHub (contents API must show ~94 KB; currently a 10-byte file from a failed tree-save).
- [ ] Create a fresh empty n8n workflow; import via URL; verify **4 ai_languageModel edges** survive (check via the browser pinia store, not the canvas).
- [ ] Verify the Gemini credential on the 4 model nodes (expect "JJ [Paid]").
- [ ] Deactivate the old v1.9 workflow `Fk9gYFZHEw41bpwP` and the ~90-node merged workflow; publish v2.0.
- [ ] E2E: add LOHAS PARK `11385` + HIGHWOOD `11520` to the watchlist → `POST /pd02/check` → expect 21 new documents → ~5–10 min AI run → snapshot/units/report all green.
- [ ] Extend the frontend: project library (search + list), watchlist, monitor view with PDF viewer (direct SRPE fetch, CORS OK), floor-plan page jump.
- [ ] Update `00_README.md` and the design spec to reflect the live-API architecture.
