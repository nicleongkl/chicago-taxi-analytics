[README.md](https://github.com/user-attachments/files/29942392/README.md)
# Chicago Taxi Trips — Data Analytics Case Study

## Overview

This project analyzes the Chicago Taxi Trips public dataset using a production-ready data pipeline built on Google Cloud Platform. The pipeline transforms raw trip data into analytical models that answer four business questions.

## Tech Stack

- **Data Warehouse:** Google BigQuery
- **Transformation:** GCP Dataform
- **Visualization:** Looker Studio
- **Version Control:** GitHub
- **Source Table:** `bigquery-public-data.chicago_taxi_trips.taxi_trips`

## Project Structure

```
definitions/
├── staging/
│   └── stg_taxi_trips.sqlx          # Cleaned raw data
├── intermediate/
│   ├── int_driver_shifts.sqlx       # Reconstructed driver shifts
│   └── int_us_holidays.sqlx         # US federal holidays reference
└── marts/
    ├── q1_top_tip_earners.sqlx      # Q1: Top 100 tip earners
    ├── q2_top_overworkers.sqlx      # Q2: Top 100 overworkers
    ├── q3_holiday_impact.sqlx       # Q3: Holiday impact on trips
    ├── q4_bonus_insights.sqlx       # Q4: Peak demand by hour/day
    └── q4_payment_tipping.sqlx      # Q4: Tipping by payment type
```

## Data Modeling Approach

The pipeline follows a three-layer architecture:

- **Staging:** Cleans the raw dataset by removing rows with null critical fields (taxi_id, timestamps), filtering out negative fares/tips, zero-distance trips, and trips where the end time precedes the start time.
- **Intermediate:** Contains reusable building blocks — the shift reconstruction model and the holiday reference table.
- **Marts:** Final analytical models that directly answer each question and power the Looker Studio dashboard.

## Pipeline Automation

The entire pipeline is automated through GCP Dataform. All models use `${ref()}` 
dependencies, so Dataform resolves the execution order automatically:

1. `stg_taxi_trips` runs first (staging)
2. `int_driver_shifts` and `int_us_holidays` run next (intermediate)
3. All mart models run last (Q1–Q4)

To re-run the full pipeline: open the Dataform repository → Start Execution → 
select "All actions" → Execute. Dataform handles the dependency graph and runs 
everything in the correct order.

This can also be scheduled via Dataform's workflow configurations to run on a 
recurring basis (e.g., daily) for production use.

## Questions & Methodology

### Q1: Top 100 Tip Earners

**Definition:** The 100 taxi IDs with the highest total tip earnings in the last 3 months of available data.

**Assumptions:**
- "Last 3 months" refers to the last 3 months of data present in the dataset, not the current calendar date, since the dataset is not updated in real time.
- Tips are summed per taxi_id and ranked. Additional context metrics (total trips, average tip per trip, tip as percentage of fare) are included to provide a fuller picture.

### Q2: Top 100 Overworkers

**Definition:** The 100 taxi IDs that most consistently work long shifts without taking adequate rest breaks.

**Defining a "Shift" — Data-Driven Approach:**

Rather than assuming an arbitrary shift boundary, we used the data to determine what constitutes a shift:

1. We first eliminated all gaps of 8+ hours between trips (clearly rest periods, not active waiting).
2. We analyzed the distribution of remaining between-trip gaps:
   - Median gap: 30 minutes
   - P75: 60 minutes
   - P90: 135 minutes (~2.25 hours)
   - P95: 195 minutes (~3.25 hours)
3. We selected the P90 value (135 minutes) as the shift boundary: if a driver has no trip for 135+ minutes, the shift is considered over. This is grounded in the observation that 90% of active between-trip waits are shorter than this.

**Defining a "Long Shift":**

With shifts reconstructed using the 135-minute boundary, we analyzed shift duration distribution:
- Median shift: 2 hours
- P75: 6 hours
- P90: 9 hours
- P95: 11 hours

We defined a "long shift" as 6+ hours (the P75 threshold), meaning shifts longer than what 75% of all observed shifts last.

**Ranking Methodology:**

The ranking uses two factors:
1. **Primary: `pct_long_shifts`** — the percentage of a driver's shifts that are 6+ hours. Captures "regularly have a long shift."
2. **Tiebreaker: `pct_short_breaks`** — the percentage of between-shift breaks that are under 8 hours. Captures "without taking at least 8 hours break."

**Why not a hard filter on 8-hour breaks?**

We considered filtering out any driver who ever took an 8-hour break but rejected this approach. Most drivers take proper rest sometimes — the overworkers are those who do it less often than they should. A hard filter would eliminate drivers who took a single 8-hour break but otherwise consistently overwork. The percentage-based ranking captures the pattern of behavior rather than penalizing isolated instances of rest.

A minimum threshold of 10 total shifts is applied to ensure statistical relevance.

### Q3: Holiday Impact on Trips

**Definition:** Comparison of average daily trip volume on US public holidays versus non-holidays.

**Assumptions:**
- Holidays included: New Year's Day, Independence Day, Thanksgiving, and Christmas Day — the four major US federal holidays — from 2013 to 2024.
- Comparison is based on average daily trip count, average daily fare, and average daily tips for holiday vs. non-holiday days.
- Individual holiday breakdown is provided to identify which holidays have the greatest impact.

### Q4: Bonus Actionable Insights

**Insight 1: Peak Demand by Hour and Day of Week**

- **Finding:** Trip volume and average fares vary significantly by hour and day of week.
- **Business Value:** Taxi companies can optimize driver scheduling to match demand patterns, reducing idle time during low-demand periods and ensuring adequate coverage during peaks. Drivers can maximize earnings by targeting high-demand time slots.

**Insight 2: Payment Type and Tipping Behavior**

- **Finding:** Cash payments show significantly lower reported tips compared to card payments.
- **Business Value:** This suggests cash tips are likely underreported rather than absent. Encouraging or incentivizing card-based payments could increase reported tip revenue, improve tax compliance, and give fleet operators better visibility into actual driver earnings.

## Looker Studio Dashboard

https://datastudio.google.com/s/svhDpNqQ3T4

## How to Run

1. Set up a GCP project with billing enabled
2. Enable the BigQuery and Dataform APIs
3. Create a Dataform repository and connect to this GitHub repo
4. Create a development workspace
5. Execute all models (Dataform handles dependency ordering)
6. Connect the marts tables to Looker Studio for visualization
