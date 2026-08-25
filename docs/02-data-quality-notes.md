# 02 - Data Quality Notes

Findings from manually checking the GA sessions dataset (Aug 2016 to Aug 2017) before building the SQL layer.

## Volume & structure

* 903,653 total sessions; 11,515 sessions with a transaction; 9,996 unique converting visitors.
* Conversion rate is about 1.27%. This is normal for e-commerce traffic and is not a data quality issue.
* 1,166 of the 9,996 converters (about 11.7%) have 2 or more transactions. This limits how detailed the cohort and LTV analysis can be.
* Channel-level cohorts are workable. But combining channel with acquisition month will create small and unreliable groups for some channels.

## Channel skew

* Referral (4,593), Organic Search (3,155), and Direct (1,725) account for most converting visitors.
* Paid Search (444) has less volume but is still usable.
* Display (122), Social (100), and Affiliates (9) do not have enough volume for reliable channel-level payback conclusions. These will be marked as "insufficient data"(depending on what I actually see from the results) instead of being forced into a ranking.

## Units

* `transactionRevenue` is stored in micros. It must be divided by 1,000,000 to get the actual currency value.
* This conversion is handled in the code before analysis.

## Apparent duplicates were a false positive

* An initial Excel deduplication pass flagged about 53,637 "duplicate" rows.
* The issue was with the original query. It pulled session-level fields without `visitId`, so different sessions from the same visitor on the same day could look identical.
* Re-running the query with `visitId` included resolves the issue. These were not real duplicate sessions.

## Transaction count field is unreliable

* `totals.transactions` does not represent the actual number of distinct orders.
* This was confirmed across 11,552 converting sessions. The session-level transaction count matched the number of transaction completion hits in only 4 cases.
* Instead, `COUNT(DISTINCT transactionId)` from the hits table is used. Every converting session in this dataset has exactly 1 distinct transaction.
* This also matches the documented approach for this dataset ([OWOX BI reference](https://support.owox.com/hc/en-us/articles/4403601828884-How-to-check-the-number-of-transaction-hits-in-BigQuery-per-date)).
* The exact cause is still unclear. About 97% of affected sessions show the same revenue value across multiple completion hits, which looks consistent with a page refresh or repeated transaction hit.
* A smaller group, about 2.6% or roughly 300 sessions, has different revenue values across hits. The reason for this is still unknown.

## Revenue field is used as-is

* `totals.transactionRevenue` was compared with hit-level `transaction.transactionRevenue` in three ways:

  * Directly summing the hit-level revenue: 0 of 11,552 sessions matched.
  * Checking whether one repeated hit value matched the session total: 0 of 20 sampled sessions matched.
  * Testing revenue minus tax minus shipping: the session total was consistently higher, so this explanation was rejected.
* No available combination of hit-level fields could reproduce `totals.transactionRevenue`.
* For the analysis, `totals.transactionRevenue` is treated as the main revenue figure.
* This is because the dataset's own summary metrics are based on this field, and there is no more reliable alternative available.
* This remains a limitation of the dataset, not a confirmed explanation of why the values differ.

## Outstanding checks for Phase 3/4

* Some rows have unusually high revenue or transaction counts.
* These will be tested using both capped and uncapped versions of the analysis rather than being removed without evidence.
* If the channel profitability ranking stays the same, the anomaly is unlikely to affect the commercial conclusion.
* If the ranking changes, that difference will be reported as a finding.
