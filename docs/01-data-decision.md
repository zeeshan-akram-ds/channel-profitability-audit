# 01 - Data Decision

## Chosen dataset

**`bigquery-public-data.google_analytics_sample.ga_sessions_*`**: real session-level data from the Google Merchandise Store, covering August 2016 to August 2017, about 12 months.

### Why this one

- It covers a full 12 months, which matters because the project needs both 3-month and 6-month payback windows. A shorter dataset, such as the newer GA4 sample that covers only about 92 days, would not support that properly.

- It has more than 900,000 session-level rows with real session-to-purchase linkage. `channelGrouping` and `trafficSource` are recorded on the same sessions that convert, so the acquisition and commerce data come from the same real source instead of being stitched together from unrelated companies.

- It contains a repeat-buyer signal through `fullVisitorId`, which is important for the cohort and LTV analysis.

## Options considered and rejected

**Olist.** I already used Olist for a previous project. Its order-level transaction data is strong, but its companion Marketing Funnel dataset tracks **seller acquisition** (MQL → seller → products listed → seller orders), not which channel brought the eventual consumer. Using it as consumer channel attribution would misrepresent what the data actually measures, so I ruled it out.

**A single dataset with everything included.** I looked for one dataset containing real transactions, real channel attribution, and real ad spend. In practice, I did not find a public dataset that provides all three reliably. Datasets that appear to do so either include synthetic fields or have important limitations. Chasing a perfect all-in-one dataset was not worth the tradeoff.

**Combining three unrelated real datasets by channel and time period.** I considered using separate sources for commerce, acquisition, and ad spend and aligning them by channel and time. I rejected this because the sources would represent different companies. A channel's ad spend at Company A has no real relationship with Company B's customers. Matching them by channel and time would create a coincidental relationship rather than a real one.

## What's still missing and how it will be handled

The GA sessions data covers both commerce and acquisition, but two important pieces are missing.

**1. Ad spend by channel and month**

The dataset does not include Google's actual customer acquisition spend. This will therefore be modeled rather than presented as observed data.

The model will use real channel-month session volume from the dataset and apply **period-correct advertising benchmark rates from 2016-2017**, rather than current rates. CPC and CPM have changed substantially over time, so using today's benchmarks for 2016-2017 traffic would distort the estimate.

Organic channels such as Referral, Organic Search, and Direct will have spend set to 0 because no paid media cost needs to be estimated for them.

Every benchmark rate will be documented with its source and year in `docs/03-sql-decisions.md`.

**2. Margin**

The dataset contains `transactionRevenue`, but not profit or COGS. A gross margin assumption will therefore be used to convert revenue into contribution margin.

Instead of relying on one fixed number, the analysis will use a base case plus a sensitivity range. This will show whether the main finding remains true across a reasonable range of margins.

These are explicit assumptions, not real data presented as real. That distinction will remain clear throughout the project.