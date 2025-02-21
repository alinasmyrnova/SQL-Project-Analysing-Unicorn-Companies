# SQL Project: Analyzing Unicorn Companies

## Project Overview  
This project focuses on analyzing trends among high-growth companies, commonly known as "unicorns." The main goal is to understand which industries generate the highest valuations and how frequently new billion-dollar startups emerge.  

## Data Source  
The database used in this project is from **DataCamp** and is available in **PostgreSQL**. It is called **"unicorns"** and contains data on unicorn companies up to the year **2021**.
You can access the dataset and explore it using **DataCamp's DataLab**:  
🔗 [DataCamp Unicorns Database](https://www.datacamp.com/datalab/w/02a4c191-3c7a-46de-ad7a-e9c708c51327/edit)  

### What is a Unicorn Company?  
A **unicorn company** is a privately held startup with a valuation exceeding **$1 billion**. These companies are often backed by venture capital and operate in various industries.  

### Database Structure  
The **unicorns** database consists of multiple tables, providing insights into different aspects of high-growth startups:  

![Database Schema](https://github.com/user-attachments/assets/e6679f9a-cc2e-469d-82eb-3f31e2d91574)  

## Data_Analysis
### What companies have the biggest return on investment? 
```sql
-- Finding top 10 companies with the highest ROI
SELECT 
    c.company, 
	c.country,
	i.industry,
    f.valuation, 
    f.funding, 
    ROUND((f.valuation - f.funding) / NULLIF(f.funding, 0), 2) AS ROI  
FROM funding AS f
JOIN companies AS c USING (company_id)
JOIN industries AS i USING (company_id)
WHERE f.valuation IS NOT NULL AND f.funding > 0 -- Exclude companies with missing data
ORDER BY ROI DESC 
LIMIT 10;
```
![image](https://github.com/user-attachments/assets/e7778573-9f2c-419b-bd2d-7c63bfd9d59a)

### Which countries produce the most unicorns?  Which cities are unicorn hubs?
```sql
-- Finding the number of unicorns per country
SELECT country, 
       COUNT(company_id) AS num_unicorns 
FROM companies 
GROUP BY country  
ORDER BY num_unicorns DESC
LIMIT 10;
```
![image](https://github.com/user-attachments/assets/939212ac-b72b-42a3-90bc-640f24fe0ae5)

```sql
-- Finding the number of unicorns per city (Unicorn hubs)
SELECT c.city, c.country,  
       COUNT(c.company_id) AS num_unicorns  
FROM companies AS c  
GROUP BY c.city, c.country  
ORDER BY num_unicorns DESC  
LIMIT 10;
```
![image](https://github.com/user-attachments/assets/d6dbe7af-7988-43b2-9ed5-707967ed0834)

## How long does it take for startups in different industries to become unicorns?
```sql
-- Calculate the average time (in years) from founding to unicorn status per industry
SELECT i.industry,  
       ROUND(AVG(EXTRACT(YEAR FROM d.date_joined) - d.year_founded)::numeric, 2) AS avg_time_to_unicorn_years  
FROM industries AS i  
JOIN dates AS d USING (company_id)  
WHERE EXTRACT(year FROM d.date_joined) IS NOT NULL  
GROUP BY i.industry  
ORDER BY avg_time_to_unicorn_years ASC;
```
![image](https://github.com/user-attachments/assets/77a32eae-5666-4b00-b16c-bce6bb2b21c0)

## What is the average growth rate per industry for last 5 years?
```sql
-- Calculate the average growth rate per industry for last 5 years (2017-2021)
WITH unicorn_counts AS (
    -- Count the number of unicorns per industry per year for 2017-2021
    SELECT i.industry,
           EXTRACT(YEAR FROM d.date_joined) AS year,
           COUNT(d.company_id) AS num_unicorns
    FROM industries AS i
    JOIN dates AS d USING (company_id)
    WHERE EXTRACT(YEAR FROM d.date_joined) BETWEEN 2017 AND 2021
    GROUP BY i.industry, year
),
growth_rate AS (
    -- Calculate year-over-year growth rate for each industry from 2017-2021
    SELECT industry,
           year,
           num_unicorns,
           LAG(num_unicorns) OVER (PARTITION BY industry ORDER BY year) AS prev_year_unicorns,
           CASE 
               WHEN LAG(num_unicorns) OVER (PARTITION BY industry ORDER BY year) > 0
               THEN (num_unicorns - LAG(num_unicorns) OVER (PARTITION BY industry ORDER BY year)) * 100.0 / LAG(num_unicorns) OVER (PARTITION BY industry ORDER BY year)
               ELSE NULL
           END AS growth_rate
    FROM unicorn_counts
)
-- Calculate the average growth rate
SELECT industry,
       ROUND (AVG(growth_rate),2) AS avg_growth_rate
FROM growth_rate
WHERE growth_rate IS NOT NULL 
GROUP BY industry
ORDER BY avg_growth_rate DESC;
```
![image](https://github.com/user-attachments/assets/677b6b15-7dc2-4677-adae-9cfca4b1f7fa)

## Which industries produced the most unicorns in 2019, 2020, and 2021, and what was their average valuation in billions?
``` sql
-- Finding the top industries that produced the most unicorns in 2019, 2020, and 2021

-- Count how many unicorns were created in each industry and finding top 3 industries 
WITH cte AS (
	SELECT i.industry, 
		COUNT (i.company_id) AS num_unicorns
	FROM industries AS i
	JOIN dates AS d 
	USING (company_id)
	WHERE EXTRACT(year FROM d.date_joined) in ('2019', '2020', '2021')
	GROUP BY i.industry
	ORDER BY num_unicorns DESC
	LIMIT 3
	)
,
-- Gets yearly unicorn counts and average valuation per industry 
cte2 AS (
    SELECT i.industry, 
           EXTRACT(YEAR FROM d.date_joined) AS year_became_unicorn, 
           COUNT(i.company_id) AS num_unicorns, 
           ROUND(AVG(f.valuation), 2) AS avg_valuation
    FROM industries AS i
    JOIN dates AS d USING (company_id)
    JOIN funding AS f USING (company_id)
    WHERE EXTRACT(YEAR FROM d.date_joined) IN (2019, 2020, 2021)
    GROUP BY i.industry, year_became_unicorn
)
-- Calculates valuation in billions for top 3 industries based on unicorn count
SELECT cte2.industry, 
       cte2.year_became_unicorn AS year, 
       cte2.num_unicorns, 
       ROUND(AVG(cte2.avg_valuation) / 1000000000, 2) AS average_valuation_billions
FROM cte2 
JOIN (SELECT industry FROM cte ORDER BY num_unicorns DESC LIMIT 3) AS top_industries
ON cte2.industry = top_industries.industry
GROUP BY cte2.industry, cte2.year_became_unicorn, cte2.num_unicorns
ORDER BY cte2.year_became_unicorn DESC, cte2.num_unicorns DESC;
```
![image](https://github.com/user-attachments/assets/5ef6e074-3be5-4b02-8969-16f17ec23f83)

