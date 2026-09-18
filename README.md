# VAT_Identifier_Discovery

# UK VAT DATASET FEASIBILITY STUDY

## Executive conclusion
A UK VAT dataset is buildable as a verified-positive enrichment product, but not as a complete truth table of companies -> VAT status. The discovery problem is solvable with multiple open-web source classes; the verification problem is straightforward once a candidate exists. The hard parts are recall, entity resolution, freshness, and the discipline to never turn “no candidate found” into “not VAT registered.” The largest immediate product risk is false positives, so every publishable mapping should require authoritative confirmation.

Important scope note: I did not manufacture a false-positive/recall result. The current HMRC API requires authentication, while the public checker explicitly does not support reverse lookup by company name. I therefore separate the discovery stress-test from the HMRC verification experiment and show exactly what remains to run once API credentials are available.

## 1. Executive summary

| Question | Finding | Implication |
| :--- | :--- | :--- |
| Can we discover UK VAT numbers on the open web? | Yes, for a meaningful subset. Strongest sources are VAT-bearing invoices / legal pages, public-sector spend files, public records and broad web corpora. Coverage is uneven. | Build a candidate-generation layer with multiple source types; do not rely on one directory or one search engine. |
| Can HMRC tell us the VAT number from the company name? | No. The public checker requires the VAT number as input. Current API v2 is authenticated. | Discovery and verification must be separate systems. |
| Can “not found” mean “not VAT registered”? | No. Open-web non-detection is not equivalent to non-registration. | The product should publish verified positives plus an unresolved state, not a negative VAT flag. |
| What is the most dangerous failure? | A plausible VAT number attached to the wrong company. | Precision must be treated as a hard gate; candidate scores cannot substitute for HMRC confirmation. |
| Could this work for a 40k-supplier customer? | Yes, as a targeted enrichment service. 40k is small enough to crawl high-value pages directly; UK-wide coverage is a different economics problem. | Start customer-specific, learn source yields, then generalise to a national pipeline. |

### What I would build first
* Use the current Companies House live-company snapshot as the sampling frame and canonical entity key.
* For each company, discover candidate VAT numbers from company websites, legal/terms pages, PDFs, public-spend/procurement datasets, and a bulk web corpus such as Common Crawl.
* Keep every candidate with provenance: URL/document, capture date, source type, extracted text span, company identifier, and confidence.
* Run strict authoritative verification for candidates. In production, this is the HMRC API; in the current public flow, verification is still possible only when a number is supplied to the checker.
* Publish only confirmed mappings; maintain a separate “no evidence / unresolved” state and a confidence reason code.
* Re-crawl and re-verify on change signals, not on a naive fixed full recrawl alone.

### The key economic insight
40,000 suppliers is not the hard scale. A customer-specific crawl of a few high-value pages per supplier is operationally modest. The harder problem is generalising to millions of companies with unknown websites, bot protection, stale pages, PDFs, ambiguous entity names and inconsistent disclosure. I would prove the customer-specific economics first, then decide whether national coverage justifies a Common Crawl + focused recrawl architecture.

## 2. Part 1 - research and source trail
I treated the search as a source-acquisition problem, not a scraping problem. For each source class I asked: (1) can it produce a VAT candidate without knowing the VAT number in advance, (2) can it link that candidate to the right company, (3) how broad is coverage, and (4) is the provenance acceptable for a commercial product?

### 2.1 Source landscape

| Source | What I expected | What I observed | Verdict |
| :--- | :--- | :--- | :--- |
| Companies House free bulk data | A complete population of live companies and stable company numbers to use as the frame. | The current monthly snapshot contains core company identity fields but not a VAT field or a general website field. | Essential as the population frame and entity key; not a VAT discovery source. |
| HMRC public VAT checker | Authoritative verification and perhaps a reverse lookup by company name. | The service says it cannot be used to check whether a business is VAT registered by searching its name. It expects a VAT number first. | Excellent verifier; unusable as a discovery engine. |
| HMRC VAT API v2 | Bulk authoritative verification. | Current developer documentation places v2 behind authentication; production access requires registration/credentials. | Right production verifier; not a zero-setup notebook tool. |
| Company website / legal / terms pages | A high-precision source because companies often print VAT numbers where business/legal information is disclosed. | Real UK sites expose VAT numbers on terms / legal pages. Full VAT invoices must also contain the supplier VAT registration number, but invoices are rarely public. | High-value discovery source; coverage and page structure are uneven. |
| Public-sector spend / procurement data | Structured supplier-to-VAT mappings in published CSVs. | MHRA monthly spend files expose a “Supplier VAT registration number” field with supplier names and numbers. | Very useful high-precision discovery source for the public-supplier subset; strongly biased coverage. |
| Public records / Gazette / insolvency | Structured or semi-structured names plus VAT in notices or attached documents. | Useful for individual cases and change events, but publication frequency is heterogeneous and much of the content is PDF-heavy. | Good supplemental source; not sufficient alone. |
| Common Crawl / bulk web corpus | A way to search billions of pages without crawling every site ourselves. | Current Common Crawl indexes are large enough to support domain/page retrieval at national scale; content is stale/noisy and still needs entity resolution. | Best broad-recall source once a local pipeline exists. |
| Commercial company directories | A shortcut to a large VAT field. | I found commercial company-enrichment products, but did not find a research artefact strong enough to treat one as a complete, current UK VAT ground truth. | Potential accelerator to benchmark, not a reference dataset. |

### 2.2 What was not obvious at the start
* The verifier/discovery asymmetry is now worse than a simple “API lookup” problem: the current HMRC API is authenticated, so a no-budget proof-of-concept cannot assume an unlimited public query endpoint.
* VAT invoice rules create a very strong web-discovery prior, but they do not imply web completeness. The legal requirement applies to invoices; most invoices are private.
* Companies House is excellent for population control and entity identity, but it is not a hidden VAT register. The company registration number is not a derivation path to the VAT number.
* Public-sector files are unusually valuable because they frequently contain the exact target field in structured form. Their weakness is selection bias, not data quality.
* The long tail is not “more crawling.” It is entity resolution: a number on a PDF or subsidiary page can refer to a trading entity, parent, branch, tax representative, or a different legal company.
* The dataset needs at least three states for the customer: VERIFIED VAT, NO CANDIDATE / UNRESOLVED, and CONFLICT. A binary “VAT / no VAT” field would manufacture false negatives.

### 2.3 Dead ends and why they fail

| Path I tested / considered | Expected outcome | Exact failure mode | What I kept |
| :--- | :--- | :--- | :--- |
| Reverse-searching HMRC by company name | Direct answer for every supplier. | The public service explicitly says name search is not supported; the current API is candidate-number lookup plus authentication. | HMRC as final verifier only. |
| Companies House as the VAT source | VAT number embedded in company master data. | The free company snapshot has no VAT field, and company-number -> VAT is not a deterministic relationship. | Companies House as frame + entity backbone. |
| Checksum probing against HMRC | Generate plausible VATs until one is accepted, then attach it. | A checksum only narrows the search space; it does not identify an entity. Repeated probing creates a query-amplification / abuse risk and cannot distinguish the correct company among valid numbers. | Checksum as a cheap local filter before verification, never as evidence of ownership. |
| Search-engine results as the product layer | Search the company name + “VAT” and scrape the snippet. | Ranking is unstable, snippets are incomplete, duplicate the same upstream source and are not auditable enough for identity-critical data. | Use search as a discovery accelerator; store the underlying page/PDF as evidence. |
| A single commercial directory as ground truth | Skip open-web acquisition entirely. | No reference dataset was available to establish completeness or accuracy. A vendor can simply be another layer of the same errors/staleness. | Benchmark as a candidate source; never make it the sole authority. |
| Public procurement/spend as the complete answer | Structured VAT coverage for all UK companies. | Only public suppliers appear; publication policies differ by department and time. Some public spend publications explicitly withhold VAT identifiers. | High-value source with known selection bias. |

## 3. Part 1 evidence - what I actually observed
The strongest evidence in this exercise is not a large scrape. It is a set of small observations that constrain what a scalable system can legitimately claim.

### 3.1 HMRC: discovery is impossible from the public checker alone
**Observed behavior**
The GOV.UK checker asks for a VAT number and then validates it. Its guidance states that the service cannot be used to check whether a business is VAT registered by searching its name. The current HMRC API documentation likewise describes a VAT-number lookup and places API v2 behind authentication.

* Therefore an implementation that “looks up 40,000 company names in HMRC” is not a valid design.
* The right interface is `candidate_vat` -> HMRC -> returned legal name/address/reference metadata.
* Every discovery source feeds candidates into the verifier; no discovery source gets to declare truth by itself.

### 3.2 Legal disclosure: strong discovery signal, incomplete coverage
HMRC VAT Notice 700 requires a VAT invoice to include the supplier name, address and VAT registration number. The GOV.UK checker itself notes that VAT numbers can often be found on invoices, receipts or websites. In practice, company websites expose them on legal/terms pages as well. This makes legal pages a high-yield crawl target, but not a population-complete register.

| Page family | Why it matters | How I would crawl it |
| :--- | :--- | :--- |
| Terms & conditions | Common place for legal seller identity and VAT number. | Search site navigation and likely URL patterns: `/terms`, `/terms-and-conditions`, `/legal`, `/privacy`, `/imprint`. |
| Legal / company / contact pages | Often include statutory identifiers. | Site-wide keyword scan for “VAT”, “VAT Reg”, “VAT registration”, “GB”. |
| PDF invoices / order docs | VAT number is invoice-required; strongest entity context when accessible. | Target PDFs linked from customer/supplier portals, procurement pages, downloadable invoice samples. |
| Footer / checkout | Some companies put VAT numbers in the site footer or transaction flow. | Lightweight rendered-page extraction with provenance. |

### 3.3 Public spend: unusually good structured evidence
I found HMRA/MHRA public-spend CSVs with an explicit supplier VAT registration number field. Example candidate rows from the June 2026 and May 2026 publications included Accenture (UK) Ltd, Bechtle Direct Ltd (UK), BSI Group, CDW Limited, Faculty Science Limited, Acquire Consultant Solutions, Fat Media Limited and Honeyman Water Ltd. I deliberately treat these as discovery candidates, not verified results, because I did not run them through the current authenticated HMRC API in this environment.

**Why this source matters**
This is exactly the shape we want from an open source: a supplier name and a VAT number in the same structured record. The main limitation is coverage bias - public procurement is a sample of suppliers, not a random sample of UK companies.

### 3.4 Bulk web corpora: why Common Crawl changes the architecture
Common Crawl offers a free index and very large monthly web archives. That changes the first-stage question from “How do I crawl four million companies?” to “Which captured pages contain high-value legal/entity phrases, and which companies can I connect them to?” The engineering trade-off is query/index/storage complexity rather than per-site crawling alone.

| Layer | What happens | Failure mode | Mitigation |
| :--- | :--- | :--- | :--- |
| Index discovery | Find URLs/captures likely to contain VAT/legal text. | False positives, stale captures, multiple duplicates. | Keyword + URL-pattern scoring and freshness weighting. |
| Content extraction | Retrieve WARC/WET text for candidate pages. | PDFs, JS-heavy sites, malformed HTML, robots/coverage gaps. | Multiple extraction methods; keep raw evidence. |
| Entity resolution | Map page to Companies House entity. | Parent/subsidiary/trading name collisions. | Name + address + domain + company number where present; contradiction checks. |
| VAT extraction | Regex/NER identifies candidate numbers. | Headers, footer boilerplate, multiple VAT numbers. | Document-span provenance and source-context scoring. |
| Verification | HMRC confirmation. | API quotas, access, stale candidate. | Cache, retry, recheck-on-change, dead-letter queue. |

## 4. Part 2 - proof of concept

**What I can prove from this run**
I can prove that multiple independent open-source classes expose candidate VAT numbers and that Companies House can provide the company population frame. I cannot honestly report a HMRC-verified recall or false-positive rate from this environment because I do not have current HMRC API credentials and the public HMRC service does not support reverse lookup by name. I therefore report “verification sample: 0” rather than inventing a result.

### 4.1 Sample design I would use for the actual measurement
The correct sample must not be “companies known to publish VAT numbers.” I would draw a simple random sample from the latest Companies House live-company snapshot, with a fixed random seed, and record the frame date. I would then stratify a second validation sample by incorporation age and SIC family to detect systematic blind spots.

| Stage | Design | Why |
| :--- | :--- | :--- |
| Population frame | Latest Companies House live-company snapshot. | Defines the denominator and avoids cherry-picking web-visible companies. |
| Primary sample | N=200 simple random draw, fixed seed, no VAT-based prefilter. | Large enough to show source mix and estimate discovery yield with useful uncertainty. |
| Robustness sample | N=100 stratified by age (new/mid/old) and broad SIC groups. | Separates “web absence” from industry/age effects. |
| Discovery | Run all source classes identically for every sampled company. | Allows source-level recall comparison. |
| Verification | Send every candidate VAT to HMRC; publish only exact/strong name-address matches. | Measures false positives instead of assuming them away. |

### 4.2 Measurement definitions

| Metric | Definition | What I would report |
| :--- | :--- | :--- |
| Discovery yield | Companies for which at least one candidate VAT was surfaced / companies sampled. | By source and in aggregate. |
| Verified coverage | Companies with at least one HMRC-confirmed VAT mapping / companies sampled. | Primary product metric. |
| Candidate precision | HMRC-confirmed candidates / candidate numbers submitted. | Measures source noise and entity resolution. |
| False-positive rate | 1 - candidate precision. | Report with N candidates, not just a percentage. |
| Company-level recall | Verified companies found / all sampled companies known to be VAT registered. | Only calculable if the sample contains a trustworthy positive set; otherwise do not call non-discovery “not registered.” |
| Conflict rate | Companies with >1 plausible VAT candidate that resolve to different names/addresses. | Important leading indicator for silent corruption. |

### 4.3 Executed discovery stress-test (not a recall sample)
To avoid presenting cherry-picked examples as a population estimate, I used a small source-validation set only: structured public-spend rows plus company-directory / website examples. The purpose was to confirm that the extraction shapes exist, not to estimate coverage. These rows remain explicitly “candidate_unverified” in the working notes.

| Source class | Observation | What it proves | What it does not prove |
| :--- | :--- | :--- | :--- |
| MHRA public spend CSVs | Supplier name and VAT registration number are co-located in published structured files. | A high-quality, machine-readable discovery source exists. | Nothing about the share of UK companies covered. |
| UK company websites / legal pages | Examples include O2, Balluff UK, Sage UK, Anesco, Transvend and Restore publishing VAT numbers in legal/terms content. | VAT numbers do appear on company-controlled pages and are crawlable. | Nothing about arbitrary-company recall. |
| Companies House / secondary snapshots | Open sample datasets reproduce official company number/name/address fields, but do not add a complete VAT field. | A clean company entity spine is available. | Nothing about VAT ownership. |
| HMRC verification | No authenticated API result was executed in this environment. | The missing step is access/control, not a hidden claim of success. | No precision or false-positive number can be honestly reported. |

### 4.4 Why the missing HMRC run matters
* Without HMRC verification, a scraped VAT number is only a claim made by a page or dataset.
* Without a random sample, high discovery yield is easy to fake by starting with companies that are known to publish their VAT number.
* Without a complete positive reference set, “nothing found” cannot be turned into recall or registration status.
* The next controlled experiment should therefore be small, random and fully verified before any national crawling investment.

## 5. Part 3 - what I would do with real resources
I would build a two-speed system: a customer-specific high-precision enrichment path that can pay for itself quickly, and a reusable national candidate graph that compounds over time.

### 5.1 Architecture

| Layer | Implementation | Output |
| :--- | :--- | :--- |
| Entity spine | Monthly Companies House bulk snapshot + incremental change feed where available. | Company number, legal name, address, SIC, status, history. |
| Source planner | Select page families and sources by expected yield (website/legal, public spend, procurement, records, web corpus). | Target URL/source queue per company. |
| Crawler / retriever | Async fetch with per-domain rate limits, caching, proxy pool only when necessary, JS rendering only for sites that need it. | Raw HTML/PDF/WARC evidence + metadata. |
| Extraction | Regex + document classification + lightweight NER. | Candidate VAT, text span, source type, timestamp. |
| Entity resolution | Company name/address/domain matching; subsidiary/parent guardrails. | Candidate-company edges with scores and reasons. |
| Verifier | HMRC API v2; cached and retried safely. | Authoritative confirmation + returned HMRC entity fields. |
| Decision layer | Strict rules for publish / conflict / unresolved. | Verified mapping or review queue. |
| Monitoring | Coverage, precision audits, drift, source freshness, conflict rate, verifier health. | Production alerts and retraining/reprioritisation signals. |

### 5.2 Cost model: rough order of magnitude
These are planning ranges, not vendor quotes. The right unit economics depend on page depth, proxy use, rendering share and the proportion of companies with a discoverable web presence. I would measure these in the first 10-20k companies before committing to national scale.

| Work item | Rough cost / company | Main cost driver | Comment |
| :--- | :--- | :--- | :--- |
| Customer-specific website discovery + crawl | $0.03-$0.15 | Requests, rendering, storage | 40k suppliers is operationally manageable with targeted page selection. |
| Bulk-corpus candidate discovery | $0.005-$0.05 | Index query + content retrieval + compute | Best economics when the same corpus supports many customers. |
| Proxy / anti-bot overhead | $0.00-$0.20+ | Blocked domains and request volume | Heavy-tailed. This is where “all companies” can become expensive. |
| Entity resolution / human review | $0.005-$0.05 average | Uncertain cases and annotation rate | Spend should follow ambiguity, not every company equally. |
| Verification engineering | Low marginal data cost; non-zero platform cost | API access, retry, caching, quota | Budget the integration and controls even if per-call pricing is nil. |

### 5.3 What breaks first
* Long-tail discovery: companies with no usable website, JS-only pages, regional domains, portals or weak web footprint.
* Entity ambiguity: parent versus subsidiary, re-used brand names, trading names and stale pages.
* Source duplication: multiple sites repeating the same bad VAT number creates false confidence unless provenance is clustered.
* Freshness: a correct VAT number from last year can still be wrong for an invoice today after deregistration or entity change.
* Operational access: anti-bot, rate limiting and third-party source policy changes.

### 5.4 Production monitoring

| Monitor | Trigger | Action |
| :--- | :--- | :--- |
| Verified precision | Random monthly audit / conflict sample shows precision drift. | Pause affected source, review source-specific extraction/entity rules. |
| Source yield | Candidate yield per 1,000 companies falls sharply. | Check source freshness, parser breakage, robots/access changes. |
| Conflict rate | Two plausible VATs per company rises. | Increase human review; do not auto-publish conflicts. |
| Staleness | Source last-seen age crosses SLA. | Re-crawl / re-verify affected company. |
| Verifier health | HMRC status errors or latency rise. | Backoff, queue, cache; never fail open. |
| Entity drift | Company name/address changes without VAT evidence update. | Trigger re-discovery and re-verification. |

## 6. Debate topics

### 6.1 “Can I use the checksum against HMRC?”
Use the checksum locally, not as an HMRC search strategy. For common UK VAT formats the checksum removes most random nine-digit strings, but it still leaves a large set of syntactically plausible values and provides no mapping to a company. A loop that generates valid-looking values and asks HMRC to accept/reject them is effectively turning a verifier into a discovery oracle. Even if a rate limit did not exist, the result would be ambiguous: a valid number tells me that the number exists, not that it belongs to J Smith Building Services Ltd.

**Rule I would encode**
Checksum = candidate filter. HMRC confirmation + entity match = publishable identifier.

### 6.2 Keeping the dataset current
* Full company-frame refresh: monthly Companies House snapshot.
* Event-triggered refresh: name, address, status or website-domain change; filing events that imply business change.
* Candidate refresh: re-crawl high-confidence web sources on a shorter cadence and re-verify existing VATs when evidence changes.
* Negative handling: never store “not registered” purely because no candidate was found; store “no evidence at timestamp T”.
* Historical truth: keep old verified mappings with validity intervals so downstream systems can explain a past invoice match.

### 6.3 How to know the dataset is wrong at scale without a reference set
* Randomly audit a small slice against HMRC continuously; a reference set can be generated by sampling candidates, even if no complete dataset exists.
* Track cross-source disagreement: the same company yielding multiple VATs is a strong error signal.
* Monitor impossible patterns: a VAT candidate repeatedly moving between unrelated company names or addresses.
* Use temporal consistency: a stable VAT should not flip every crawl unless there is corroborating entity change.
* Create source-specific holdouts: do not train/evaluate entirely on the same pages that generated the candidate.
* Measure missingness by cohort (age, SIC, web-presence class, geography) so an aggregate coverage figure cannot hide a systematic blind spot.

### 6.4 Sources I would not sell as authoritative
* Search-engine snippets: they are unstable presentation artifacts, not durable source records.
* Unattributed company directories: useful for discovery, but provenance may be inherited from unknown upstreams.
* Historical documents with stale VATs: useful as evidence of past state, not present truth.
* Any source that produces a VAT number without enough company context to distinguish parent/subsidiary/trading entity.
* A checksum-only result or any inferred VAT derived from formatting rules: inference is not verification.

## 7. Beyond the UK - Germany
Germany is similar in discovery mechanics but different in identifier semantics and verification. I would not copy the UK implementation blindly; I would keep the same architecture and swap the country-specific identifier and verifier modules.

| Dimension | UK | Germany | Implication |
| :--- | :--- | :--- | :--- |
| Identifier source | VAT number is not derived from Companies House company number. | USt-IdNr is a separate identifier issued under German VAT law; it is not the Handelsregister number. | Same entity spine, separate VAT-ID discovery. |
| Website disclosure | VAT number often appears on legal/terms pages, but there is no universal Companies House-style VAT field. | German digital-service provider rules require the USt-IdNr to be shown in the website imprint when the provider has one. | Website crawling is especially relevant. |
| Invoice disclosure | VAT registration number is required on full VAT invoices. | German VAT invoice rules require a tax number or USt-IdNr; intra-EU transactions use the USt-IdNr. | Invoice/PDF discovery remains a useful source class. |
| Verification | HMRC checker / current API v2. | VIES provides EU VAT-number validation, though “invalid” can have activation/status nuances. | The verifier adapter changes, not the whole pipeline. |

**What gets easier / harder**
* Easier: VIES is a public EU-wide verification surface, so the authentication constraint is different from the current HMRC API path.
* Harder: the USt-IdNr is a separate identifier from the company registration number, so there is no simple registry-number transformation to exploit.
* Similar: discovery still depends on web disclosure, invoices, procurement/public records and entity resolution; a missing web candidate is still not proof of non-registration.

**Market-prioritisation principle**
I would prioritise countries where three things align: (1) the target identifier is publicly disclosed often enough to discover, (2) an authoritative verifier is available cheaply and predictably, and (3) the company registry provides a strong entity key. Countries with easy derivation from another public identifier are much cheaper; countries with a separate opaque VAT identifier and weak verification/disclosure are materially harder. I would not label a specific additional country “genuinely hard” without doing the same source trail first.

## 8. Final recommendation

**Go, but sell the right promise**
I would pursue the product for the 40,000-supplier customer, but define the deliverable as a verified VAT enrichment dataset with explicit unresolved/conflict states. I would not promise “every UK company with a VAT number” until a random HMRC-verified sample demonstrates sufficient recall.

* Run the 200-company random sample as the next experiment once HMRC API credentials are available.
* Measure verified coverage, candidate precision, false-positive rate and source-level yield separately.
* If customer-specific coverage clears the commercial threshold, operationalise targeted crawling first because 40k is small enough to manage without a huge national crawler.
* Build the bulk-corpus layer only after the customer sample shows that the remaining gap is web-discovery coverage rather than registration status.
* Keep authoritative verification as a hard gate throughout. The product risk is not a missing VAT number; it is a wrong VAT number that looks plausible.

## Appendix A - source trail
* S1 Companies House - Free Company Data Product - https://download.companieshouse.gov.uk/en_output.html
* S2 GOV.UK - Check a UK VAT number - https://www.gov.uk/check-uk-vat-number
* S3 HMRC Developer Hub - Check a UK VAT number API v2 - https://developer.service.hmrc.gov.uk/api-documentation/docs/api/service/vat-registered-companies-api/2.0
* S4 GOV.UK - VAT invoices: what they must include (VAT Notice 700) - https://www.gov.uk/guidance/vat-invoices-what-they-must-include
* S5 GOV.UK - Company signs, stationery and promotional material - https://www.gov.uk/running-a-limited-company/signs-stationery-and-promotional-material
* S6 Common Crawl - Index / Overview - https://commoncrawl.org/overview
* S7 Common Crawl - Index server - https://index.commoncrawl.org/
* S8 EU Commission - VIES VAT number validation - https://ec.europa.eu/taxation_customs/vies/
* S9 Germany - UStG section 14 (invoices) - https://www.gesetze-im-internet.de/ustg_1980/__14.html
* S10 Germany - UStG section 27a (USt-IdNr) - https://www.gesetze-im-internet.de/ustg_1980/__27a.html
* S11 Germany - Digital Services Act / DDG section 5 (provider information) - https://www.gesetze-im-internet.de/ddg/__5.html
* S12 UK public-spend evidence - MHRA supplier spend CSVs on data.gov.uk - data.gov.uk; monthly MHRA spend-over-£25k publications; field observed: “Supplier VAT registration number”

## Appendix B - limitations of this run
* No live HMRC API credential was available, so no HMRC candidate lookup was executed and no false-positive percentage is claimed.
* The public HMRC checker is interactive and explicitly does not support company-name reverse lookup, so it cannot substitute for the missing API credential in this run.
* Web search results are discovery aids, not authoritative evidence; all candidate examples were kept clearly separate from “verified” counts.
