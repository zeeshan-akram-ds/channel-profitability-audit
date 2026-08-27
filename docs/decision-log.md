# Decision Log - channel-profitability-audit

Chronological record of important decisions, findings, and rejected alternatives.

This is not a task log. It only records decisions that affect the project's scope, method, or interpretation.

---

## Phase 0 - Framing

**Decision:** The core question is whether the ROAS ranking matches the true profitability ranking after CAC, using 3-month and 6-month payback windows.

The scope is limited to acquisition channels.

**Rationale:** I chose this to support Framework #3 in the DM rotation, which focuses on the marketing efficiency hypothesis and did not yet have a case study behind it.

**Out of scope:** Claims about what founders generally do or do not know about their channel economics.

---

## Phase 1 - Data source decision

**Decision:** Use `bigquery-public-data.google_analytics_sample.ga_sessions_*` from the Google Merchandise Store, covering August 2016 to August 2017.

**Rejected alternatives:**

* **GA4 obfuscated sample:** It only covers about 92 days, which is too short for a 6-month payback analysis.

* **Olist:** Its funnel data tracks seller acquisition from MQL to seller orders. It does not show which channel brought the eventual consumer. Using it for consumer CAC or channel attribution would not be valid.

* **Single pre-packaged dataset:** I did not find a reliable public dataset that already combines orders, channel attribution, and ad spend for this use case.

* **Three unrelated real datasets merged by channel and time:** This would combine data from different companies. A relationship created by matching channels and time would not represent a real relationship between the customers and the ad spend.

**Synthetic layer plan:** This will be added later and has not been built yet.

* Spend will use real GA channel volume multiplied by benchmark CAC or CPL rates from 2016 to 2017. Current 2026 rates will not be used.
* Margin will use a base case plus a sensitivity range based on an apparel gross margin of about 40% to 60%.

---

## Phase 2 - Data quality investigation

**Fact:** There are 903,653 total sessions, 11,515 transacting sessions, and 9,996 unique converting visitors. The conversion rate is 1.27%.

**Fact:** Only 1,166 of the 9,996 converters have 2 or more transactions. This means channel-level cohorts are workable, but channel by month cohorts will be sparse.

**Fact:** Referral (4,593), Organic Search (3,155), and Direct (1,725) account for most converters. These channels have $0 estimated spend because they are organic channels.

Paid Search (444) has enough volume to analyze.

Display (122), Social (100), and Affiliates (9) do not have enough volume for reliable conclusions. These channels will be marked as insufficient data(depending on what analysis will show) rather than ranked.

**Fix:** `transactionRevenue` is stored in micros. It is converted to the actual currency value by dividing by 1,000,000.

**Resolved false positive:** An earlier Excel check flagged about 53,000 duplicate rows. The problem came from a query that did not include `visitId`. Different sessions from the same visitor on the same day could look identical. Adding `visitId` resolved the issue. These were not real duplicates.

**Finding:** `totals.transactions` is unreliable across the dataset.

I tested this across 11,552 converting sessions. The session-level `totals.transactions` value matched the number of completed purchase hits in only 4 cases.

`COUNT(DISTINCT transactionId)` returns 1 for essentially every converting session.

**Decision:** The project will use `COUNT(DISTINCT transactionId)` as the transaction count instead of `totals.transactions`.

**External validation:** OWOX BI documentation supports this correction for this dataset.

**Unresolved:** The exact reason for the inflated transaction count is still unclear.

About 97% of affected sessions show the same revenue value repeated across multiple completion hits. This looks consistent with a page refresh or repeated transaction hit, but the exact cause has not been confirmed.

About 2.6%, or roughly 300 sessions, show different revenue values across hits. The reason for these differences is still unknown.

**Finding:** I tested three methods to reconcile hit-level revenue with `totals.transactionRevenue`. None of them worked.

1. Sum of hit-level revenue vs `totals.transactionRevenue`: 0 of 11,552 matched.
2. Single repeated value vs session total: 0 of 20 sampled sessions matched.
3. Hit revenue minus tax minus shipping: the session total was consistently larger, so this explanation was rejected.

**Decision:** Use `totals.transactionRevenue` as-is because there is no better available alternative.

This is still a limitation of the dataset. I am not treating it as a confirmed explanation.

About 300 affected sessions are flagged with `revenue_unverified_flag` for a Phase 4 sensitivity check.

**Decision:** Extreme values will not be removed at this stage.

In Phase 3, the analysis will be run twice: once with values capped at the 99.5th percentile and once with the original uncapped values. Phase 4 will then check whether the channel ranking changes.

---

## Infra decisions

* `.gitignore` excludes `data/raw/`, notebook checkpoints, and the private content-ideas file.
* `data/raw/` contains untouched exports and will not be committed. `data/processed/` will contain the cleaned data used from Phase 3 onward.
* Raw data can be recreated using the documented export queries instead of being stored in the repository.
* The export queries are finalized in `docs/01-data-decision.md`:

  1. `data/raw/ga_sessions_all_2016_2017.csv` contains all sessions across all channels with no transaction filter. It is used as the base for spend and volume analysis.
  2. `data/raw/ga_sessions_transactions_2016_2017.csv` contains the corrected transaction count using `COUNT(DISTINCT transactionId)`, revenue from `totals.transactionRevenue ÷ 1e6`, and the `revenue_unverified_flag`.

---

## Open items carried into Phase 3

* Confirm that `docs/02-data-quality-notes.md`, `.gitignore`, and the export query section are actually committed and pushed.
* About 300 revenue-unverified sessions remain a known limitation. The flag should be carried through the Phase 3 tables.
* The reason for the roughly 2.6% of sessions with genuinely different revenue values across hits is still unknown.

## Phase 3 - Synthetic layer sourcing

**Decision: Paid Search benchmark source**

Use WordStream's *Google AdWords Industry Benchmarks* from February 29, 2016, verified through the Wayback Machine archive.

[Archived WordStream benchmark source](https://web.archive.org/web/20160307012705/http://www.wordstream.com/blog/ws/2016/02/29/google-adwords-industry-benchmarks?utm_source=chatgpt.com)

The study covers 2,367 US client accounts with $34.4M in total spend from Q2 2015.

For the e-commerce vertical, the base case is:

* CTR: 1.66% search / 0.45% display
* CPC: $0.88 search / $0.29 display
* CVR: 1.91% search / 0.96% display
* CPA: $46.07 search / $30.21 display

**Rejected:** An AI-generated Gemini summary of the same source. It gave a different methodology, including $444M in spend, 2,367 accounts, Q4 2015 to Q1 2016, and a CPA of $45.83. These figures do not match the original source.

The AI summary was not used. This reinforces the rule to check AI-generated summaries against the original source before using their figures.

**Unresolved limitation: CPA as CAC assumption**

WordStream's e-commerce CPA is a blended "cost per action" metric. The original article and WordStream's glossary do not clearly say whether the action means a purchase or a lead for this specific e-commerce vertical.

Because of this, using $46.07 as a purchase-level CAC proxy is an assumption, not a verified fact.

This will be carried into Phase 4 as a named limitation. If changing this assumption materially changes the channel ranking, it will be treated as a possible explanation.

---

**Decision: Apparel DTC margin source**

Use two real company S-1 filings from the same FY2016 and FY2017 period.

| Company             | FY2016 GM | FY2017 GM | COGS scope                                       |
| ------------------- | --------: | --------: | ------------------------------------------------ |
| Revolve Group (S-1) |    46.58% |    48.47% | Excludes outbound fulfillment and shipping       |
| Stitch Fix (S-1)    |    44.26% |    44.45% | Includes outbound shipping and styling-box costs |

Sources:

* Revolve Group: sec.gov/Archives/edgar/data/1746618/000156459018023704/ck0001746618-s1.htm
* Stitch Fix: sec.gov/Archives/edgar/data/1576942/000119312517313629/d400510ds1.htm

**Rejected comparables:**

* **The RealReal:** Wrong period because the relevant filing is from 2019. Its consignment model also does not match a normal purchase and resale model.
* **Lands' End:** A multichannel catalog and store retailer, not primarily an e-commerce business.
* **Columbia Sportswear:** Mainly a wholesale manufacturer, so it is not a good DTC comparison.

**Unresolved limitation: COGS definition mismatch**

The margins from Revolve and Stitch Fix are not directly comparable.

Stitch Fix includes shipping and styling-box costs in COGS, while Revolve excludes outbound fulfillment and shipping. Because of this, the roughly 2 to 4 percentage point difference between them is partly caused by how each company defines COGS.

I will not try to normalize the figures. This difference will remain documented as a limitation.

The sensitivity range will use **44.26% to 48.47%**, while Revolve's individual yearly figures will be used as the base case.

**n=2 limitation**

Two companies are better than having no real company benchmarks, but they are still not enough to claim that this represents the apparel e-commerce industry.

The figures represent the disclosed financials of these two companies during this period. They will not be presented as an industry-wide fact.

## Phase 3 - Margin base case finalized

**Decision:** Base-case apparel DTC gross margin = **45.93%**

Sensitivity range = **[44.26%, 48.47%]**

**Derivation:** I calculated the average margin for each company first:

* Revolve: (46.58% + 48.47%) / 2
* Stitch Fix: (44.26% + 44.45%) / 2

I then averaged those two company averages to get **45.93%**.

**What this number means:** This is not an estimate of the "true apparel e-commerce margin" under one accounting standard.

It is a practical midpoint between the margins reported by two real companies. Their COGS definitions are different. Revolve excludes outbound shipping and fulfillment, while Stitch Fix includes them.

I did not try to normalize the two companies. I am using 45.93% as a simple working assumption for the synthetic layer, not as a confirmed industry benchmark.

**Range source:** The **44.26% to 48.47%** range is the actual minimum and maximum of the four underlying figures from Revolve FY2016, Revolve FY2017, Stitch Fix FY2016, and Stitch Fix FY2017.

The range has not been padded or adjusted beyond the figures actually observed.

**Carried limitation:** The COGS scope difference from the previous entry still applies. It has not been resolved. I have only turned the available figures into one working base case.

## Phase 3 - Paid Search benchmark, two-period range finalized

**Source:** Same WordStream article. Both time periods are reported on the page.

[WordStream Google AdWords Industry Benchmarks](https://www.wordstream.com/blog/ws/2016/02/29/google-adwords-industry-benchmarks?utm_source=chatgpt.com)

**Period 1:** Q2 2015, based on 2,367 accounts and $34.4M in spend. This is the original report, verified through the Wayback Machine.

**Period 2:** August 2017 to January 2018, based on 14,197 accounts and more than $200M in spend. This comes from the current live page and was retrieved directly, not from an archived copy.

### E-commerce vertical

| Metric       | Period 1 (2015) | Period 2 (2017-18) |
| ------------ | --------------: | -----------------: |
| CPC (search) |           $0.88 |              $1.16 |
| CPA (search) |          $46.07 |             $45.27 |
| CVR (search) |           1.91% |              2.81% |
| CTR (search) |           1.66% |              2.69% |

**Decision: CPC**

Base case = **$1.02**

Range = **[$0.88, $1.16]**

The base case is the average of the two periods. The range is the actual minimum and maximum from the two periods.

**Decision: CPA**

Base case = **$45.67**

Range = **[$45.27, $46.07]**

Again, the base case is the average and the range uses the actual minimum and maximum.

CPA is much more stable between the two periods than CPC. The CPC range has about a 32% spread, while CPA has only about a 2% spread.

The CVR also increased from 1.91% to 2.81%. This helps explain why CPA stayed relatively stable even though CPC increased. This is a useful internal check on the figures.

**CPA as CAC assumption**

The source provides stronger support for using CPA as a purchase-based CAC proxy, but it does not fully settle the issue.

WordStream's FAQ says conversions are defined based on the "useful measure of success" for each account. It also gives the example that **"For retailers, that may be a sale."**

This supports using e-commerce CPA as a reasonable purchase-based CAC proxy. However, the wording still says "may be", and it is not a definition specifically for this e-commerce dataset.

The assumption is therefore no longer treated as undocumented. It is supported by the source's own explanation for retailers, but it will still be tested in Phase 4 sensitivity analysis.
