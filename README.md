# Netflix Users — Exploratory Data Analysis

## Goal
This project applies exploratory data analysis to a synthetic Netflix user dataset, approached from the perspective of a business analyst presenting findings to stakeholders. Each query is framed around a concrete business question, and results are interpreted in terms of actionable insights.

## Dataset

| Field | Description |
|---|---|
| `User_ID` | Unique identifier for each user |
| `Name` | Randomly generated name |
| `Subscription_Type` | Basic, Standard, or Premium |
| `Country` | User's country of residence |
| `Age` | User age (used to derive age groups) |
| `Favorite_Genre` | User's preferred content genre |
| `Watch_Time_Hours` | Total watch time logged |
| `Last_Login` | Date of most recent activity |

- **Source:** [[Kaggle — Netflix Userbase Dataset]]([url](https://www.kaggle.com/datasets/smayanj/netflix-users-database))
- **Loaded into BigQuery:** `netflix-users-eda.netflix_user_data.netflix_user_data`

## Key Findings

- Subscription type distribution is highly uniform across all countries, with no market showing a meaningful skew toward any tier.
- Watch time is roughly equal across subscription types — subscription tier does not appear to predict engagement level.
- The US, UK, Brazil, Canada, France, and Germany each account for ~10% of total watch time; Japan is the lowest contributor.
- Genre preferences vary meaningfully by country (e.g. Australians favor documentaries; Brazilians favor romance) — useful signal for regional content strategy.
- Genre preferences also vary by age group, with clear patterns that can inform content targeting by demographic.

---

## Queries

### 1. Subscription type distribution by country

**Business question:** Are certain subscription tiers over- or under-represented in specific markets? Stakeholders with regional revenue targets will want to know if a given market skews toward Basic or Premium.

```sql
SELECT 
    Subscription_Type,
    Country, 
    COUNT(Subscription_Type) AS subscription_count,
    ROUND(COUNT(*) * 100 / SUM(COUNT(*)) OVER (PARTITION BY Country), 1) AS percentage_of_country_market 
FROM `netflix-users-eda.netflix_user_data.netflix_user_data`
GROUP BY Subscription_Type, Country;
```

<img width="882" height="432" alt="eda 1" src="https://github.com/user-attachments/assets/69a3c956-03bb-4cb1-8bb4-13aba5c52922" />

**Finding:** Distribution is highly uniform — subscription type shares vary by only 1–2 percentage points across countries, with no country showing a meaningful outlier. This suggests that tier preference is not strongly driven by geography in this dataset.

---

### 2. Watch time by subscription type

**Business question:** Do higher-tier subscribers watch more? And given that Basic now includes ads, does watch time translate to ad revenue opportunity?

```sql
SELECT
    Subscription_Type,
    ROUND(SUM(Watch_Time_Hours), 1) AS total_watch_hours,
    ROUND(SUM(Watch_Time_Hours) * 100.0 / SUM(SUM(Watch_Time_Hours)) OVER (), 1) AS percentage_of_total_watch
FROM `netflix-users-eda.netflix_user_data.netflix_user_data`
GROUP BY Subscription_Type
ORDER BY Subscription_Type;
```

<img width="635" height="131" alt="eda 2" src="https://github.com/user-attachments/assets/f11ab4d6-b023-4ce5-bfda-47c83c19d0dc" />

**Finding:** Watch time is nearly identical across tiers, with Standard being marginally the lowest. The lack of a strong relationship between subscription tier and engagement suggests that pricing strategy should not rely on the assumption that Premium users are heavier viewers.

---

### 3. Watch time by country

**Business question:** Which markets represent the largest share of total engagement? This informs where to prioritize content investment and infrastructure.

```sql
SELECT 
    Country,
    ROUND(SUM(Watch_Time_Hours), 1) AS total_watch_hours, 
    ROUND(SUM(Watch_Time_Hours) * 100 / SUM(SUM(Watch_Time_Hours)) OVER (), 1) AS percentage_total_watch 
FROM `netflix-users-eda.netflix_user_data.netflix_user_data`
GROUP BY Country
ORDER BY Country;
```

<img width="632" height="366" alt="eda 3" src="https://github.com/user-attachments/assets/0d060eb1-c1d3-4645-b75e-6db59be1f6ff" />

**Finding:** The US, UK, Brazil, Canada, France, and Germany each contribute roughly 10% of total watch time. Japan is the lowest contributor. The near-equal distribution across most markets suggests no single country is a disproportionate driver of engagement — which may also reflect the synthetic nature of the dataset.

---

### 4. Favorite genre by country

**Business question:** Do content preferences differ by market? Regional content executives need this to prioritize what to license or produce for each country.

```sql
SELECT
    Country,
    Favorite_Genre,
    COUNT(*) AS total_users,
    ROUND(COUNT(*) * 100 / SUM(COUNT(*)) OVER (PARTITION BY Country), 1) AS percentage_of_country
FROM `netflix-users-eda.netflix_user_data.netflix_user_data`
GROUP BY Country, Favorite_Genre
ORDER BY Country, percentage_of_country DESC;
```

<img width="886" height="505" alt="eda 6" src="https://github.com/user-attachments/assets/764f03c3-9495-4f3e-83a5-a0247e267d5c" />

**Finding (sample shown):** Australian users favor documentaries and least prefer action; Brazilian users favor romance and least prefer Sci-Fi. Even a partial view of this table is actionable for content executives making regional acquisition decisions.

---

### 5. Favorite genre by age group

**Business question:** Does genre preference shift with age? This informs demographic targeting and helps match content recommendations to the right audience segments.

*Note: Age is bucketed into standard research segments using a CASE statement in a subquery, then ranked within each group using a window function.*

```sql
SELECT
    Age_Group,
    Favorite_Genre,
    total_users,
    ROUND(total_users * 100.0 / SUM(total_users) OVER (PARTITION BY Age_Group), 1) AS percentage_of_age_group,
    RANK() OVER (PARTITION BY Age_Group ORDER BY total_users DESC) AS genre_rank
FROM (
    SELECT
        CASE
            WHEN Age < 18 THEN 'Under 18'
            WHEN Age BETWEEN 18 AND 24 THEN '18-24'
            WHEN Age BETWEEN 25 AND 34 THEN '25-34'
            WHEN Age BETWEEN 35 AND 44 THEN '35-44'
            WHEN Age BETWEEN 45 AND 54 THEN '45-54'
            WHEN Age >= 55 THEN '55+'
        END AS Age_Group,
        Favorite_Genre,
        COUNT(*) AS total_users
    FROM `netflix-users-eda.netflix_user_data.netflix_user_data`
    GROUP BY Age_Group, Favorite_Genre
)
ORDER BY Age_Group, genre_rank;
```

<img width="1037" height="501" alt="eda 4" src="https://github.com/user-attachments/assets/08ad251a-3ef2-45d9-9648-4ac358b2b471" />
<br>
<img width="1040" height="492" alt="eda 5" src="https://github.com/user-attachments/assets/e9333d1f-372d-40e5-888e-0fb3733a4710" />

**Finding:** Clear genre preferences emerge across age groups, offering a foundation for age-targeted content recommendations and marketing. Full table not shown due to size — the pattern across all groups follows a consistent structure of ranked genre preferences per segment.
