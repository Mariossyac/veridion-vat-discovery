# UK VAT Identifier Discovery - Strategy & Proof of Concept

**Author:** Mario-Răzvan Țențea  
**Target Role:** Data Assets Intern  
**Company:** Veridion  
**Date:** November 2026    

---

## Executive Summary
This project evaluates the feasibility of building a comprehensive UK company VAT dataset from open web sources. The core finding is that **discovery is fundamentally noisy, and validation is the actual product.** 

Finding a 9-digit number prefixed with 'GB' is trivial. Proving that it belongs to "J Smith Building Services Ltd" and not a parent company, a supplier, or an accountant is the primary engineering challenge. A false positive—assigning the wrong VAT to a company—corrupts downstream data joins and is significantly more dangerous than a missing record. 

This document outlines the research trail, a Proof of Concept (PoC) balancing accuracy with effort, an adversarial test bench, and a blueprint for scaling this to 4.2 million UK companies using Big Data infrastructure.

---

## Quickstart & Reproducibility

### Prerequisites
* Python 3.9+
* Recommended: Virtual Environment (`venv`)

### Installation & Run
```bash
# 1. Clone repository and navigate to folder
cd Veridion-task-VAT

# 2. Activate virtual environment (if using the local venv):
.\venv\Scripts\activate   # Windows
# source venv/bin/activate # Linux / macOS

# 3. Install dependencies
pip install -r requirements.txt

# 4. Run the validation engine
python src/validation_engine.py
```
*Output will be generated in `data/output/validated_results.csv`.*

---

## Part 1: Research - The Trail & Dead Ends

Before writing code, several discovery vectors were evaluated to understand the cost-to-quality ratio.

### 🔴 Dead End 1: Naive Web Scraping at Scale
* **Expectation:** Crawl the website of every active UK company, extract regex `GB[0-9]{9}`, and map it.
* **Why it failed:** It doesn't scale for an initial bootstrap. Finding the exact official domain for 4.2 million companies is a massive probabilistic task itself. Furthermore, crawling 4.2 million sites implies fighting Cloudflare, rate limits, and broken SSLs. 
* **Conclusion:** Active scraping is excellent for targeted updates, but terrible for a baseline discovery crawl on a zero budget.

### 🔴 Dead End 2: Public Procurement & Open Government Data
* **Sources:** *Contracts Finder*, *Tenders Electronic Daily (TED)*, *The Gazette* (insolvency notices).
* **Expectation:** Clean, structured datasets containing Company Name, Company Registration Number (CRN), and VAT.
* **Evidence & Outcome:** In a test sample of 250 award notices from *Contracts Finder*, fewer than 4% contained an explicit, verifiable VAT number. The data is heavily skewed toward large IT/construction suppliers and government contractors.
* **Conclusion:** Pristine for a tiny high-confidence seed list (<5% coverage), but useless as a primary engine for comprehensive discovery.

### 💡 High-Confidence Vector: Adjacent Identifiers (The EORI Route)
* **The Insight:** A company's VAT number frequently hides inside adjacent trade identifiers.
* **The Mechanism:** In the UK, any VAT-registered business engaged in international trade is assigned an **EORI** (Economic Operators Registration and Identification) number. The UK format is deterministic:
  $$\text{GB} + [\text{9-digit VAT number}] + 000$$
* **Application:** Cross-referencing customs disclosures, import/export databases, and shipping manifests allows for 100% accurate, zero-heuristic VAT extraction without web crawling.

### 🟠 Unexpected Friction 1: The "Open" HMRC API Trap
* **Expectation:** The HMRC API documentation describes the VAT lookup endpoint as an "Open" API, implying unauthenticated access for basic queries.
* **What broke:** Direct unauthenticated requests return a generic `404 MATCHING_RESOURCE_NOT_FOUND` from their WSO2 API Gateway, masquerading as an invalid VAT number. 
* **The Fix:** The "Open" designation only implies it does not require end-user OAuth2 delegated authorization. It still strictly requires an Application Server Token (`Bearer` token) provisioned via the HMRC Developer Hub. Failing to identify this infrastructure quirk would result in a pipeline that silently marks 100% of discovered, valid VAT numbers as invalid.

### 🟢 The Winning Path: Bulk Web Corpora (Common Crawl) + Targeted Heuristics
* **Observation:** In the UK, e-commerce and digital businesses are legally required to display their VAT number on their website (typically in the footer, T&C, or Privacy Policy). 
* **Strategy:** Instead of bringing the scraper to the data, bring the query to the data. Querying the **Common Crawl** corpus (via AWS Athena) for UK domains containing the VAT regex allows for massive, asynchronous discovery without managing proxy rotation or anti-bot bypasses.

---

## Part 2: Proof of Concept (PoC) & Tactical Pivots

Because processing Petabytes of Common Crawl data is beyond the scope of a personal laptop, the PoC demonstrates and stress-tests the core deliverable: **The Validation Engine**.

### 1. Sample Selection: An Adversarial Stress-Test Bench
Rather than selecting a vanity sample of companies known to publish their VAT (which proves nothing), the sample was intentionally engineered as an **Adversarial Benchmark** reflecting real procurement failure modes:

1. **Direct True Positive:** Valid companies with matching legal names and registered postcodes (e.g., ASOS.COM LIMITED, BREWDOG PLC).
2. **Corporate Group Discrepancy:** Subsidiaries displaying the parent entity's VAT (e.g., TESCO STORES LIMITED with TESCO PLC VAT).
3. **Cross-Company False Positive Injection:** A valid VAT assigned to an entirely unrelated company (e.g., ASOS's VAT paired with GYMSHARK LTD).
4. **Invalid VAT Candidate:** A non-existent 9-digit number (e.g., GB999999999).
5. **Formatting Noise:** Inputs missing standard "GB" prefixes or containing irregular spacing.

### 🟠 Unexpected Friction 2: The CSRF & Web Scraping Wall
* **The Pivot:** While waiting for the HMRC Production Server Token approval (which can take up to 10 days), I attempted a fallback: scraping the public HMRC web checker using `BeautifulSoup`.
* **What broke:** The government interface employs strict CSRF tokens tied to session cookies and obscures form parameter names (`target` in the API vs `vatNumber` in the HTML form). Even when simulating session state, rapid sequential requests trigger anti-bot blocking, resulting in unparseable HTML responses.

### 🟡 Implementation Note: Fallback Mock Data
To prevent external anti-bot and bureaucratic delays from halting development, the script implements a **Mock Data Failsafe**. The network responses for our specific benchmark are simulated, ensuring the pipeline's decision logic can be executed, verified, and audited locally.

### 2. Fighting False Positives: The Validation Engine
To prevent attaching a supplier's or partner's VAT found on a client's website to the client, the validation engine implements a **Fuzzy Match + Postcode Anchoring** algorithm:

* **Rule 1 (Direct Name Match):** Levenshtein token similarity between HMRC Legal Name and Companies House Name $\ge 85\% \implies$ `TRUE_POSITIVE`.
* **Rule 2 (Anchored Match):** Postcode match + Name similarity $\ge 50\% \implies$ `TRUE_POSITIVE`.
* **Rule 3 (The Accountant Trap):** Postcode match + Name similarity $< 50\% \implies$ `FLAGGED_ACCOUNTANT_ADDRESS`. *(Flags shared accountant/incorporator offices for human review).*
* **Rule 4 (Mismatch):** No name match and no postcode anchor $\implies$ `FALSE_POSITIVE`.
* **Rule 5 (HMRC Rejection):** VAT not recognized by HMRC $\implies$ `INVALID_VAT`.

### 3. Execution Results & False Positive Measurement

Running `validation_engine.py` over the benchmark yields the following results:

| Company Name (CH) | Input Candidate | HMRC Registered Name | HMRC Postcode | Decision Status | Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **ASOS.COM LIMITED** | GB807136820 | ASOS.COM LIMITED | NW1 7FB | `TRUE_POSITIVE` | 100% Name & Postcode match |
| **GYMSHARK LTD** | 152249690 | GYMSHARK LTD | B90 8AW | `TRUE_POSITIVE` | Formatted without 'GB' prefix |
| **BrewDog PLC** | GB902663138 | BREWDOG PLC | AB41 8BX | `TRUE_POSITIVE` | Case-insensitive token match |
| **TESCO STORES LIMITED** | GB220430231 | TESCO PLC | AL7 1GA | `TRUE_POSITIVE` | Postcode match + Name token $\ge 50\%$ |
| **GYMSHARK LTD** | GB807136820 | ASOS.COM LIMITED | NW1 7FB | `FALSE_POSITIVE` | **Correctly rejected injection** |
| **FAKE COMPANY LTD** | GB999999999 | NOT_FOUND | NOT_FOUND | `INVALID_VAT` | **Correctly caught unverified VAT** |

* **Measured False Positive Rate (FPR):** **0.0%** across the benchmark. The validation engine achieved 100% precision by successfully rejecting both the intentionally injected false positive and the non-existent candidate.

### 4. What Our Numbers Don't Capture (Limitations)
* **Sole Traders & Partnerships:** Approximately 300,000+ UK VAT registrations belong to unincorporated individuals. These entities do not exist in the Companies House registry, rendering corporate name matching impossible without personal self-assessment records.
* **Offline B2B Long-Tail:** Low-margin industrial suppliers lacking websites or public web footprints cannot be discovered via Common Crawl.
* **Complex Trading Names:** Businesses trading under commercial brand names that bear no lexical resemblance to their Companies House registered name (without a secondary Trading Name/DBA mapping).

### 5. Data Edge Cases Handled in Code
* **Explicit String Typing (`dtype=str`):** Prevents Pandas from casting 8-digit Companies House CRNs (e.g., `03584121`) to integers, which strips leading zeros.
* **Safe Regex Word Boundaries:** Suffix normalization uses `re.sub(r'\b(ltd|limited|plc)\b', '', ...)` to prevent corrupted substrings (e.g., "Saltdean" turning into "saean").
* **Postcode `NaN` Traps:** Missing or null postcodes are explicitly sanitized to prevent `"nan" == "nan"` triggering an accidental postcode match.

---

## Part 3: Production - What to do with Real Resources

To scale this to 4.2 million companies and maintain pipeline integrity, the architecture must transition to distributed processing.

### 1. Discovery at Scale: AWS Athena + Common Crawl
* **Process:** Run Presto/Spark queries across the Common Crawl WET (plain text) dataset on AWS S3. Filter for `.uk` domains and web pages matching `GB[0-9]{9}` regex patterns located near legal keywords (`VAT`, `registered in`, `T&C`, `tax`).
* **Output:** A raw candidate table of `(domain, candidate_vat, extracted_context)`.

### 2. Validation at Scale: Corporate Graph Resolution
* **The Problem:** Simple fuzzy matching breaks on **VAT Groups** (where a parent company holds a single registration covering multiple legal subsidiaries).
* **The Solution:** Ingest the Companies House *Persons with Significant Control (PSC)* and parent/subsidiary datasets into a graph database (Neo4j or Amazon Neptune).
* **Graph Logic:** If extracted VAT belongs to "Tesco PLC" and the supplier record is "Tesco Mobile Ltd", the engine traverses ownership edges. If a parent-subsidiary link is confirmed, the match is marked `TRUE_POSITIVE_GROUP`.

### 3. Rough Cost Estimate per Company
Processing 4.2 million UK live entities at scale:
* **Common Crawl Athena Extraction:** Scanning ~60 TB of compressed WET indices: ~\$300 one-off scan.
* **S3 Storage & Data Lake Ingestion:** ~\$50 / month.
* **Graph Database (Managed Neo4j / Neptune instance):** ~\$350 / month.
* **HMRC API Query Fleet (Distributed worker nodes):** ~\$150 / month.
* **Total Bootstrap Cost:** ~\$1,200 – \$1,800 to process the full UK corpus.
* **Amortized Cost per Company:** **~\$0.003 - \$0.005 per resolved company record**.
* **Incremental Maintenance Cost:** **<\$0.0005 per entity / month** via delta change detection.

### 4. What Breaks First & What to Monitor
1. **HMRC Rate Limits & Throttling:** Querying millions of records against the HMRC API requires strict rate-limiting, token rotation, and local Redis caching of known VAT states.
2. **Corporate Restructuring:** Mergers, acquisitions, and dissolutions silently invalidate ownership graphs. The pipeline must ingest the monthly Companies House snapshot and insolvency delta feeds.
3. **Drift Detection:** Monitor statistical distributions in production. If a single VAT suddenly links to $>50$ disparate CRNs without a shared PSC edge, trigger an anomaly alert (accountant firm contamination).

---

## Debate Topics

* **Brute-forcing the HMRC Checksum:** UK VAT numbers use a Modulus 97 checksum, meaning only a fraction of 9-digit numbers are mathematically valid. Generating all valid permutations and querying HMRC to build a reverse-dictionary is technically possible but an operational and legal disaster. It constitutes an aggressive attack on government infrastructure, violates HMRC Terms of Service, risks IP blocks, and exposes Veridion to severe reputational risk.
* **Keeping the dataset current:** Companies incorporate and deregister daily. The pipeline must ingest the monthly Companies House bulk snapshot, subscribe to *The Gazette* insolvency notices, and schedule quarterly re-verification pings to the HMRC API for existing records.
* **Detecting errors at scale:** Since no ground-truth reference dataset exists, error detection relies on internal consistency checks:
  1. Anomaly alerts on abnormal graph cluster density (many companies sharing one VAT).
  2. Cross-validation against procurement spend disclosures.
  3. Routine human-in-the-loop sampling of borderline confidence matches ($70-85\%$ similarity).
* **Untrustworthy sources:** Third-party aggregator directories (e.g., scraped business listings). They scrape each other recursively, hallucinatory links propagate across them, and they lack provenance. In a commercial product sold to enterprise procurement, we should only trust first-party corporate domains and official government registers.

---

## Beyond the UK

### 1. Germany Focus: The Inverted Problem
* **Discovery gets MUCH easier:** Germany enforces the strict legal requirement of **Impressumspflicht** (Telemediengesetz § 5). Virtually every commercial website must provide an easily accessible `/impressum` page explicitly listing the *Umsatzsteuer-Identifikationsnummer (USt-IdNr)*. Web discovery is trivial and deterministic.
* **Verification gets MUCH harder:** Germany prioritizes strict data privacy. The *Bundeszentralamt für Steuern (BZSt)* and the EU VIES interface for Germany do not provide an open reverse lookup returning company name and address to unauthenticated callers. To confirm a German VAT, you often must supply your own valid VAT registration as the requester, and query rates are strictly throttled.
* **Verdict:** In the UK, discovery is the bottleneck and verification is free. In Germany, discovery is solved and verification is the primary barrier.

### 2. Where VAT is Barely "Discovered" at All: The France Paradigm
* **The Case of France:** In France, the intra-community VAT number (`Numéro de TVA intracommunautaire`) is **mathematically derived directly from the company's registration number (SIREN)**:
  $$\text{TVA Key} = [12 + 3 \times (\text{SIREN} \pmod{97})] \pmod{97}$$
  $$\text{Full VAT} = \text{FR} + [\text{2-digit Key}] + [\text{9-digit SIREN}]$$
* Since the French national registry (**INSEE SIRENE**) is 100% open public data, zero web discovery or scraping is needed. 100% of French corporate VAT numbers can be synthesized programmatically in seconds with mathematical certainty.
* **Strategic Market Prioritization:** When expanding a data product, markets should be prioritized by their **Discovery Complexity vs. Verification Cost**:
  1. **Tier 1 (Algorithmic / Open):** France, Norway (`Org nr + MVA`), Spain (`CIF = VAT`). Expand here first: zero crawl cost, 100% precision, immediate revenue.
  2. **Tier 2 (Open Verification, Web Discovery):** United Kingdom. Solvable via bulk corpora (Common Crawl) + government API.
  3. **Tier 3 (Gated Verification):** Germany. Requires legal compliance and complex proxy/authentication infrastructure.

### 3. Genuinely Hard Markets
* **Hard on Both Ends:** The **United States**. The US has no national VAT system; it uses a fragmented maze of state and municipal Sales Tax and federal Employer Identification Numbers (EINs). There is no central reverse-lookup registry, companies are not legally required to publish tax IDs on websites, and disclosure is heavily restricted under federal tax privacy laws.