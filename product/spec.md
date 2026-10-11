# Zimbabwe / Africa Data Platform: Product Spec (v0, mind-dump)

Status: draft for discussion. Nothing here is built. Everything under "How it could be done" is speculation, not tested against the live sources.

## 1. Objective

Build one queryable, continuously refreshed dataset that connects four things that are normally scattered: where minerals and mines are, what the companies behind them are worth, who the electricity and procurement counterparties are, and what public money is being spent on. Then lay analysis products on top (investor support, sector analysis, tender intelligence) that update without manual work.

The core claim of the product is the **join**, not any single source. Mindat alone, a bourse filing alone, or a tender list alone is already public. A mine linked to its listed owner, its financials, its power connection and the tenders landing in its district is not.

## 2. Source inventory (from the whiteboard)

| Domain | Source | What we want | Notes |
|---|---|---|---|
| Mines and minerals | Mindat | Localities, minerals, commodities, coordinates, geology, mine status | Has an official API (key needed). Site terms restrict bulk scraping. |
| Company financials | African Financials | Statements, ratios, price data for listed African companies | Likely partly subscription. Account status unknown. |
| Regulatory filings | The bourses (ZSE first, then others) | Annual and interim reports, trading statements, circulars | Mostly PDFs, inconsistent formats, per-exchange handling. |
| Power / meters | `zesa.sql`, in BigQuery (project "Emancipation of TJ") | Names and meter records to be mapped to entities | Schema not yet seen. May contain personal data. |
| Tenders | Procurement portals (PRAZ and others), plus press notices | Daily new tenders, awards, awardees, values | Must run unattended. |
| Data science layer | Internal | Models and analysis over the joined data | Handwriting on the note is unclear; see section 6. |

Out of scope for now: the "Breaking into Wall Street" material (investment-banking content). Parked deliberately.

## 3. Users and jobs to be done

- **The owner (TJ):** wants a living map of the Zimbabwean and wider African mining, agri and finance landscape, with minimal upkeep.
- **Investors, local and foreign:** want to screen mining and agri companies against ground truth (deposit, output, power access, contracts) rather than only filings.
- **Analysts and journalists:** want to trace who wins public tenders and whether awardees tie back to known entities.

## 4. Functional requirements

1. **Ingest** each source on its own schedule, idempotently, storing the raw payload untouched alongside the parsed record.
2. **Resolve entities.** One canonical entity table (company, mine, site, person-or-org counterparty) with aliases and confidence-scored links across sources.
3. **Map ZESA records** onto that entity table, with every match carrying a score, the evidence used, and a review state (auto-accepted, needs review, rejected).
4. **Tender pipeline, daily and hands-off:** discover new tenders, extract structured fields, classify by sector, link to entities and districts, score relevance, and publish a digest.
5. **Financials normalisation:** a common schema across currencies and reporting periods, with the source document and page retained for every figure.
6. **Query and presentation:** a documented schema plus a small set of dashboards or reports. The tender digest should read well, not only be correct.
7. **Provenance everywhere:** any number on any output can be traced to a source URL, fetch time and parser version.

## 5. Non-functional requirements

- Unattended operation with alerting on failure, not on success.
- Polite scraping: robots.txt respected, rate limits, a stable user agent, caching.
- Replaceable adapters, so one broken site does not stop the others.
- Reproducible: a rebuild from raw data yields the same core tables.
- Cost-bounded: prefer scheduled batch over always-on infrastructure.

## 6. Reading of the "Transform" list (verify against the note)

My transcription of the handwritten list, which is uncertain in places: Zim investor support; third-party mining analysis; third-party agri analysis; finance; ZESA meters; tenders (the last line may reference FOIA or freedom-of-information style requests). If that last item means requesting non-public procurement records, it changes the tender design, because it adds a manual or semi-manual document-request workflow. Please confirm.

## 7. How it could be done (speculation)

**Storage and shape.** Since `zesa.sql` already lives in BigQuery, use BigQuery as the warehouse and avoid a second database. Four layers: `raw` (verbatim fetches), `staging` (parsed per source), `core` (canonical entities and links), `marts` (analysis-ready views). Raw files go to a Cloud Storage bucket, with the path recorded in the table.

**Ingestion.** One small Python adapter per source behind a common interface (`discover()`, `fetch()`, `parse()`). Mindat through its API where possible. Bourse filings through each exchange's announcements page, with PDFs parsed by a text extractor first and an LLM extraction pass for tables and narrative, validated against arithmetic checks (totals must foot) before acceptance. African Financials through an authorised account only; if that is unavailable, fall back to the underlying filings.

**Entity resolution.** The hardest part and the one that decides whether the product is worth anything. Suggested approach: normalise names (case, punctuation, legal suffixes such as Pvt, Ltd, Limited, Holdings), generate candidates with blocking (token and trigram match, plus geography where present), score with a mix of string similarity and shared attributes, and use an LLM only to adjudicate the ambiguous middle band. Keep a human-review queue and let confirmed decisions feed back as training and alias data. Never auto-merge below a set confidence.

**Tender automation.** A scheduled job (Cloud Scheduler to Cloud Run, or a GitHub Actions cron to start) runs daily: discover new notices, download, extract fields (buyer, category, value, closing date, location), classify, link entities, then write a ranked digest. A cheap model can do classification and summarisation; keep the full text and the model output separately so results can be regenerated. The digest is the product surface for tenders, so give it real design attention.

**Analysis layer.** Start with descriptive marts (mine-to-company, company-to-financials, tender-to-sector-to-district), then add modelling once the joins are trustworthy. A power-demand proxy from ZESA meter activity near mines would be an unusual and defensible signal, but only if the data supports it.

**Phasing.** (1) Warehouse skeleton plus ZESA schema profiling. (2) Mindat mines and minerals. (3) Tender pipeline, because it is self-contained and shows value fastest. (4) ZSE filings and financials. (5) Entity resolution across all of it. (6) Analysis products.

## 8. Risks and constraints

- **Access:** the build sandbox currently blocks all target sites and BigQuery; allowed hosts and a read-only service account are prerequisites.
- **Terms of use:** Mindat restricts bulk scraping, and African Financials content is probably licensed. Redistribution of either in a product needs a licence check before launch.
- **Personal data:** ZESA names may identify individuals. Zimbabwe's Cyber and Data Protection Act applies. Prefer mapping organisations only, and pseudonymise or exclude individuals.
- **Fragility:** scrapers break when sites change; budget ongoing maintenance and monitoring.
- **Match quality:** wrong entity links erode trust faster than missing ones, so precision over recall.

## 9. Open questions

1. What are the ZESA dataset and table names, and what do the rows represent (customers, meters, suppliers)?
2. Do you hold a Mindat API key and an African Financials subscription?
3. Which bourses beyond the ZSE, and in what order?
4. Which tender portals are in scope, and does "FOIA" mean requesting non-public records?
5. Is this for personal use, an internal tool, or a product sold to others? The licensing answer depends on it.
6. Where should the daily digest go: email, a dashboard, a chat channel?
