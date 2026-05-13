# SQL Analytics Portfolio
### Victor Uzoagba · Business & Data Analysis

A collection of SQL queries written for fintech analytics use cases — covering user segmentation,
cohort retention, and transaction anomaly detection. Queries are written in standard SQL
compatible with PostgreSQL and BigQuery. Schema context is provided for each.

---

## Table of Contents
1. [RFM User Segmentation](#1-rfm-user-segmentation)
2. [Monthly Cohort Retention](#2-monthly-cohort-retention)
3. [Rolling 7-Day Transaction Anomaly Detection](#3-rolling-7-day-transaction-anomaly-detection)
4. [High-Value vs Low-Value User Breakdown](#4-high-value-vs-low-value-user-breakdown)
5. [Dormant User Identification](#5-dormant-user-identification)
6. [Failed Transaction Rate by Channel](#6-failed-transaction-rate-by-channel)

---

## 1. RFM User Segmentation

**What this solves:** Not all users are equal. This query scores every user on Recency (how recently
they transacted), Frequency (how often), and Monetary value (how much) — then buckets them into
segments like Champions, At Risk, and Dormant. Product and marketing teams use this to personalise
campaigns and prioritise retention efforts.

**Schema:** `transactions(user_id, amount, status, created_at)`

```sql
WITH base AS (
    SELECT
        user_id,
        MAX(created_at)                          AS last_txn_date,
        COUNT(*)                                 AS txn_count,
        SUM(amount)                              AS total_spent
    FROM transactions
    WHERE status = 'success'
    GROUP BY user_id
),

rfm_scores AS (
    SELECT
        user_id,
        last_txn_date,
        txn_count,
        total_spent,

        -- Recency: lower days since last txn = higher score
        NTILE(5) OVER (ORDER BY last_txn_date DESC)  AS r_score,

        -- Frequency: more transactions = higher score
        NTILE(5) OVER (ORDER BY txn_count ASC)       AS f_score,

        -- Monetary: higher spend = higher score
        NTILE(5) OVER (ORDER BY total_spent ASC)     AS m_score
    FROM base
),

segmented AS (
    SELECT
        *,
        ROUND((r_score + f_score + m_score) / 3.0, 2) AS rfm_avg,
        CASE
            WHEN r_score >= 4 AND f_score >= 4            THEN 'Champion'
            WHEN r_score >= 3 AND f_score >= 3            THEN 'Loyal'
            WHEN r_score >= 4 AND f_score <= 2            THEN 'New User'
            WHEN r_score <= 2 AND f_score >= 3            THEN 'At Risk'
            WHEN r_score = 1 AND f_score = 1              THEN 'Dormant'
            ELSE 'Needs Attention'
        END AS segment
    FROM rfm_scores
)

SELECT
    segment,
    COUNT(*)                            AS user_count,
    ROUND(AVG(total_spent), 2)          AS avg_spend,
    ROUND(AVG(txn_count), 1)            AS avg_transactions,
    ROUND(AVG(
        EXTRACT(DAY FROM NOW() - last_txn_date)
    ), 0)                               AS avg_days_since_last_txn
FROM segmented
GROUP BY segment
ORDER BY user_count DESC;
```

**Output:** One row per segment showing size, average spend, and recency — ready to feed into
a Metabase or Looker dashboard.

---

## 2. Monthly Cohort Retention

**What this solves:** Retention is the most honest metric for product health. This query groups
users by the month they first transacted (their cohort), then tracks what percentage return to
transact in each subsequent month. A healthy fintech product shows a retention curve that
flattens rather than drops to zero.

**Schema:** `transactions(user_id, created_at, status)`

```sql
WITH first_txn AS (
    -- Each user's acquisition month
    SELECT
        user_id,
        DATE_TRUNC('month', MIN(created_at)) AS cohort_month
    FROM transactions
    WHERE status = 'success'
    GROUP BY user_id
),

activity AS (
    -- Every month a user was active
    SELECT DISTINCT
        t.user_id,
        DATE_TRUNC('month', t.created_at) AS activity_month
    FROM transactions t
    WHERE t.status = 'success'
),

cohort_activity AS (
    SELECT
        f.cohort_month,
        a.activity_month,
        -- How many months after acquisition is this activity?
        EXTRACT(MONTH FROM AGE(a.activity_month, f.cohort_month)) AS month_number,
        COUNT(DISTINCT a.user_id)                                  AS active_users
    FROM first_txn f
    JOIN activity a ON f.user_id = a.user_id
    GROUP BY f.cohort_month, a.activity_month
),

cohort_sizes AS (
    SELECT
        cohort_month,
        COUNT(DISTINCT user_id) AS cohort_size
    FROM first_txn
    GROUP BY cohort_month
)

SELECT
    ca.cohort_month,
    cs.cohort_size,
    ca.month_number,
    ca.active_users,
    ROUND(100.0 * ca.active_users / cs.cohort_size, 1) AS retention_pct
FROM cohort_activity ca
JOIN cohort_sizes cs ON ca.cohort_month = cs.cohort_month
WHERE ca.month_number BETWEEN 0 AND 12
ORDER BY ca.cohort_month, ca.month_number;
```

**Output:** A pivot-ready table (cohort month × month number) where `retention_pct` becomes the
heatmap values in a cohort retention grid.

---

## 3. Rolling 7-Day Transaction Anomaly Detection

**What this solves:** Fraud, system errors, and sudden user behaviour shifts often show up as
spikes or drops in daily transaction volume or value. This query flags days where the transaction
count or total value deviates more than 2 standard deviations from the rolling 7-day average —
a lightweight statistical anomaly signal that can trigger alerts.

**Schema:** `transactions(id, user_id, amount, status, created_at)`

```sql
WITH daily_stats AS (
    SELECT
        DATE(created_at)      AS txn_date,
        COUNT(*)              AS txn_count,
        SUM(amount)           AS total_volume,
        AVG(amount)           AS avg_txn_value
    FROM transactions
    WHERE status IN ('success', 'failed')
    GROUP BY DATE(created_at)
),

rolling AS (
    SELECT
        txn_date,
        txn_count,
        total_volume,
        avg_txn_value,

        -- 7-day rolling average and standard deviation for count
        AVG(txn_count)   OVER (ORDER BY txn_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
            AS rolling_avg_count,
        STDDEV(txn_count) OVER (ORDER BY txn_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
            AS rolling_std_count,

        -- 7-day rolling average and standard deviation for volume
        AVG(total_volume)   OVER (ORDER BY txn_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
            AS rolling_avg_volume,
        STDDEV(total_volume) OVER (ORDER BY txn_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
            AS rolling_std_volume
    FROM daily_stats
)

SELECT
    txn_date,
    txn_count,
    total_volume,
    ROUND(rolling_avg_count, 1)   AS expected_count,
    ROUND(rolling_avg_volume, 2)  AS expected_volume,

    -- Z-scores: how many standard deviations away from the rolling mean?
    ROUND((txn_count   - rolling_avg_count)  / NULLIF(rolling_std_count,  0), 2) AS count_z_score,
    ROUND((total_volume - rolling_avg_volume) / NULLIF(rolling_std_volume, 0), 2) AS volume_z_score,

    CASE
        WHEN ABS((txn_count - rolling_avg_count) / NULLIF(rolling_std_count, 0)) > 2
          OR ABS((total_volume - rolling_avg_volume) / NULLIF(rolling_std_volume, 0)) > 2
        THEN 'ANOMALY'
        ELSE 'Normal'
    END AS flag
FROM rolling
ORDER BY txn_date DESC;
```

**Output:** A daily log with anomaly flags — pipe this into a Metabase alert or Slack notification
to catch issues before they escalate.

---

## 4. High-Value vs Low-Value User Breakdown

**What this solves:** Understanding the revenue concentration across users helps prioritise
product investment. This query answers: what percentage of users drive 80% of transaction volume?
(A fintech equivalent of the Pareto principle check.)

**Schema:** `transactions(user_id, amount, status, created_at)`

```sql
WITH user_totals AS (
    SELECT
        user_id,
        SUM(amount)  AS lifetime_value,
        COUNT(*)     AS txn_count,
        MIN(created_at) AS first_seen,
        MAX(created_at) AS last_seen
    FROM transactions
    WHERE status = 'success'
    GROUP BY user_id
),

ranked AS (
    SELECT
        *,
        NTILE(4) OVER (ORDER BY lifetime_value DESC) AS value_quartile,
        SUM(lifetime_value) OVER ()                  AS grand_total
    FROM user_totals
)

SELECT
    CASE value_quartile
        WHEN 1 THEN 'Top 25% (High Value)'
        WHEN 2 THEN 'Upper-Mid 25%'
        WHEN 3 THEN 'Lower-Mid 25%'
        WHEN 4 THEN 'Bottom 25% (Low Value)'
    END                                          AS user_tier,
    COUNT(*)                                     AS user_count,
    ROUND(SUM(lifetime_value), 2)                AS tier_revenue,
    ROUND(AVG(lifetime_value), 2)                AS avg_ltv,
    ROUND(AVG(txn_count), 1)                     AS avg_transactions,
    ROUND(100.0 * SUM(lifetime_value) / MAX(grand_total), 1) AS pct_of_total_revenue
FROM ranked
GROUP BY value_quartile
ORDER BY value_quartile;
```

**Output:** Four rows showing revenue concentration — the input for an executive "where does our
money actually come from" slide.

---

## 5. Dormant User Identification

**What this solves:** Dormant users are a re-engagement opportunity. This query flags users who
were once active (at least 2 transactions) but have not transacted in the last 60 days — and
ranks them by their historic value so the team can prioritise who to win back first.

**Schema:** `transactions(user_id, amount, status, created_at)`,
`users(user_id, phone, email, signup_date)`

```sql
WITH user_activity AS (
    SELECT
        user_id,
        COUNT(*)                           AS total_txns,
        SUM(amount)                        AS lifetime_value,
        MAX(created_at)                    AS last_txn_date,
        MIN(created_at)                    AS first_txn_date
    FROM transactions
    WHERE status = 'success'
    GROUP BY user_id
)

SELECT
    u.user_id,
    u.phone,
    u.email,
    ua.total_txns,
    ROUND(ua.lifetime_value, 2)            AS lifetime_value,
    DATE(ua.last_txn_date)                 AS last_active,
    EXTRACT(DAY FROM NOW() - ua.last_txn_date) AS days_dormant,

    -- Priority score for re-engagement: higher LTV + longer dormancy = target first
    ROUND(
        ua.lifetime_value * 0.7 +
        EXTRACT(DAY FROM NOW() - ua.last_txn_date) * 0.3
    , 2)                                   AS reengagement_priority_score
FROM user_activity ua
JOIN users u ON ua.user_id = u.user_id
WHERE
    ua.total_txns >= 2
    AND ua.last_txn_date < NOW() - INTERVAL '60 days'
ORDER BY reengagement_priority_score DESC
LIMIT 500;
```

**Output:** A ranked list of dormant users exportable directly to a CRM or marketing tool for
targeted push notifications or offers.

---

## 6. Failed Transaction Rate by Channel

**What this solves:** Failed transactions damage user trust and hide revenue. This query breaks
down failure rates by payment channel (card, USSD, wallet, bank transfer) so engineering and
product can identify which channels need urgent attention — and at what transaction value thresholds
failures tend to cluster.

**Schema:** `transactions(id, user_id, amount, status, channel, created_at)`

```sql
WITH channel_stats AS (
    SELECT
        channel,
        DATE_TRUNC('week', created_at)         AS week,
        COUNT(*)                                AS total_txns,
        COUNT(*) FILTER (WHERE status = 'failed')  AS failed_txns,
        SUM(amount) FILTER (WHERE status = 'failed') AS failed_volume,
        AVG(amount) FILTER (WHERE status = 'failed') AS avg_failed_amount
    FROM transactions
    GROUP BY channel, DATE_TRUNC('week', created_at)
)

SELECT
    channel,
    week,
    total_txns,
    failed_txns,
    ROUND(100.0 * failed_txns / NULLIF(total_txns, 0), 2) AS failure_rate_pct,
    ROUND(failed_volume, 2)                                AS failed_volume,
    ROUND(avg_failed_amount, 2)                            AS avg_failed_txn_value,

    -- Flag channels with failure rate above 5%
    CASE
        WHEN (100.0 * failed_txns / NULLIF(total_txns, 0)) > 5 THEN 'High Failure'
        WHEN (100.0 * failed_txns / NULLIF(total_txns, 0)) > 2 THEN 'Watch'
        ELSE 'Healthy'
    END AS channel_health
FROM channel_stats
ORDER BY week DESC, failure_rate_pct DESC;
```

**Output:** A weekly channel health table that turns into a Metabase line chart tracking failure
rate over time per channel.

---

## About

These queries are written against hypothetical fintech schemas typical of digital payment platforms
operating in the Nigerian market (wallet-based transactions, agent banking, mobile-first user base).

**Tools I work with:** Microsoft Excel · Google Sheets · SQL (PostgreSQL / BigQuery) · Looker Studio

📧 victoruzogba@gmail.com
