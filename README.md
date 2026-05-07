# Netflix_users_EDA
## Goal
This is a synthetic dataset that has data about Netflix users. It features demographic data, such as name, age, and country, the user's subscription type, favorite genre of content, watch time, and last login.
My goal is to conduct exploratory data analysis and uncover some behavioral patterns and/or trends and insights as if I was a data analyst working at Netflix and had to present my analysis results to business 
stakeholders to inform business decisions. I have also included my queries and all or some of the results that I achieved with the queries.

## Dataset
- **Source:** This dataset was downloaded from Kaggle
- **Loaded into BigQuery:** netflix-users-eda.netflix_user_data.netflix_user_data

First I want to see the percentages of subscription types across different countries. This will provide important data for business stakeholders, as stakeholders will likely have certain targets for specific subscription types across different markets.

```sql
SELECT 
    Subscription_Type,
    Country, 
    COUNT(Subscription_Type) AS subscription_count,
    ROUND(COUNT(*) * 100 / SUM(COUNT(*)) OVER (PARTITION BY country),1) AS percentage_of_country_market 
FROM `netflix-users-eda.netflix_user_data.netflix_user_data`
GROUP BY Subscription_Type, Country;
```

<img width="882" height="432" alt="eda 1" src="https://github.com/user-attachments/assets/69a3c956-03bb-4cb1-8bb4-13aba5c52922" />


The data looks more or less uniform across different subscription types, there aren't any outliers. The differences between countries and subscription types vary by 1 or 2 percentage points, sometimes even decimal points.


Next I want to see the connection between subscription type and watch time. This could provide valuable information on whether viewers with more expensive subscription types are likely to watch more or not.
This information can also provide us with insights on ad revenue, as the basic subscription type now includes ads and more watch time = more ad revenue.

```sql
SELECT
    Subscription_Type,
    ROUND(SUM(Watch_Time_Hours), 1) AS total_watch_hours,
    ROUND(SUM(Watch_Time_Hours) * 100.0 / SUM(SUM(Watch_Time_Hours)) OVER (), 1) AS percentage_of_total_watch
FROM `netflix-users-eda.netflix_user_data.netflix_user_data`
GROUP BY Subscription_Type
ORDER BY Subscription_Type
```

<img width="635" height="131" alt="eda 2" src="https://github.com/user-attachments/assets/f11ab4d6-b023-4ce5-bfda-47c83c19d0dc" />


The data tells us that there is a slight difference between the watch times across subscription types, however, the difference is marginal. Watch time is lowest (percentage-wise) for the the standard subscription type.


Following, it makes sense to look at the connection between countries and watch times. This could give business stakeholders relevant information on different markets and which ones are a priority when it comes to increasing watch time.

```sql
SELECT 
    Country,
    ROUND(SUM(Watch_Time_Hours),1) as total_watch_hours, 
    ROUND(SUM(Watch_Time_Hours) * 100 / SUM(SUM(Watch_Time_Hours)) OVER (),1) as percentage_total_watch 
FROM `netflix-users-eda.netflix_user_data.netflix_user_data`
GROUP BY Country
ORDER BY Country;
```

<img width="632" height="366" alt="eda 3" src="https://github.com/user-attachments/assets/0d060eb1-c1d3-4645-b75e-6db59be1f6ff" />


Japan has the smallest percentage of total watch time, while the US, UK, Brazil, Canada, France, and Germany are more or less tied with each other, each contributing to around 10 percent of watch time each. Again, differences between countries are marginal.


Now I want to see the connection between countries and favorite genres - this could provide valuable data to stakeholders responsible for different markets and their content offerings.

```sql
SELECT
Country,
Favorite_Genre,
COUNT(*) AS total_users,
ROUND(COUNT(*) * 100 / SUM(COUNT(*)) OVER (PARTITION BY Country),1) AS percentage_of_country
FROM netflix-users-eda.netflix_user_data.netflix_user_data
GROUP BY Country, Favorite_Genre
ORDER BY Country, percentage_of_country DESC;
```

<img width="886" height="505" alt="eda 6" src="https://github.com/user-attachments/assets/764f03c3-9495-4f3e-83a5-a0247e267d5c" />

I included a small snippet of the table for a general understanding of the result - here we can see that Australian users like documentaries most and action least, while Brazilian users like romance most and Sci-Fi least. This snippet would already be useful for content executives when deciding what content to post on specific markets.

The logical thing is to also look at the connection between the age of users and their favorite genres - this could give stakeholders valuable insights when targeting specific age groups and deciding on the most attractive content offerings.

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
ORDER BY Age_Group, genre_rank
```

<img width="1037" height="501" alt="eda 4" src="https://github.com/user-attachments/assets/08ad251a-3ef2-45d9-9648-4ac358b2b471" />
<br>
<img width="1040" height="492" alt="eda 5" src="https://github.com/user-attachments/assets/e9333d1f-372d-40e5-888e-0fb3733a4710" />


This table gives us a clear picture of the rank of all genres across the standard age groups that are usually used in research and analysis (this is part of the table, as it is a rather large one).
