# Student-Performance-Overview

In this project, I analyzed a dataset of 6,600 students to identify the primary drivers of academic performance. My goal was to move beyond simple descriptive statistics and determine which factors meaningfully impact exam scores — especially across different socioeconomic groups.

Using MySQL, Python, and Tableau, I built a full analysis pipeline that moves from raw data to actionable insight. The focus of the project is instructional equity: understanding which interventions create the strongest academic lift for high-risk students.

## Objectives

- Manually implement Pearson’s Correlation in MySQL

- Evaluate tutoring impact across socioeconomic segments

- Benchmark students using SQL window functions

- Validate findings using Python modeling

- Build an interactive equity dashboard in Tableau

## Tech Stack

- MySQL – Data engineering and statistical modeling

- Python (Pandas, Seaborn, Matplotlib, Scikit-Learn) – EDA and feature importance

- Tableau – Interactive equity analysis dashboard

- SQL Approach (MySQL)

Because MySQL does not include a native CORR() function, I manually implemented the Pearson Correlation Coefficient using aggregate math inside CTEs.

I structured the analysis in three stages:

1. Data Preparation

   - Created a Change_In_Score field

   - Classified students into socioeconomic tiers

   - Cleaned and standardized fields

2. Benchmarking with Window Functions

   - AVG() over partitions for school comparison

   - RANK() for within-school ranking

   - PERCENT_RANK() for global percentile

   - LAG() to measure peer performance gaps

3. Manual Pearson Correlation

   - I implemented the formula:

      r = ( nΣXY − ΣXΣY ) / √[(nΣX² − (ΣX)²)(nΣY² − (ΣY)²)]

   -  This allowed me to measure correlation between:

      - Tutoring Sessions and Exam Score

      - Attendance Percentage and Exam Score

      - Hours Studied and Exam Score

Core SQL Query
/* ANALYSIS OBJECTIVE: 
   1. Manual implementation of Pearson's Correlation (MySQL compatible). 
   2. Evaluation of Tutoring impact across socioeconomic segments.
   3. Student analysis using Window Functions.
*/

```
WITH Student_Cleaned AS (
    SELECT *, 
        (Exam_Score - Previous_Scores) AS Change_In_Score,
        CASE 
            WHEN Family_Income = 'Low' AND Access_to_Resources = 'Low' THEN 'High Risk'
            WHEN Family_Income = 'High' AND Access_to_Resources = 'High' THEN 'Resource Rich'
            ELSE 'Moderate' 
        END AS Socioeconomic_Status
    FROM studentperformancefactors
),

Performance_Stats AS (
    SELECT *,
        AVG(Exam_Score) OVER(PARTITION BY School_Type) AS School_Avg_Score,
        RANK() OVER(PARTITION BY School_Type ORDER BY Exam_Score DESC) AS School_Rank,
        PERCENT_RANK() OVER(ORDER BY Exam_Score) AS Global_Percentile
    FROM Student_Cleaned
),

Growth_Analysis AS (
    SELECT *, 
        LAG(Exam_Score) OVER(PARTITION BY School_Type ORDER BY School_Rank) AS Next_Highest_Score,
        Exam_Score - LAG(Exam_Score) OVER(PARTITION BY School_Type ORDER BY School_Rank) AS Gap_To_Peer
    FROM Performance_Stats
)

SELECT 
    School_Type,
    Socioeconomic_Status, 
    COUNT(*) AS Student_Count, 
    ROUND(AVG(Exam_Score), 2) AS Avg_Exam_Score,
    ROUND(AVG(Change_In_Score), 2) AS Avg_Growth,

    ROUND((COUNT(*) * SUM(Tutoring_Sessions * Exam_Score) - SUM(Tutoring_Sessions) * SUM(Exam_Score)) / 
        (SQRT((COUNT(*) * SUM(Tutoring_Sessions * Tutoring_Sessions) - POW(SUM(Tutoring_Sessions), 2)) *
              (COUNT(*) * SUM(Exam_Score * Exam_Score) - POW(SUM(Exam_Score), 2))
        )), 3
    ) AS Tutoring_Correlation_Coefficient,

    ROUND((COUNT(*) * SUM(Attendance_Pct * Exam_Score) - SUM(Attendance_Pct) * SUM(Exam_Score)) / 
        (SQRT((COUNT(*) * SUM(Attendance_Pct * Attendance_Pct) - POW(SUM(Attendance_Pct), 2)) *
              (COUNT(*) * SUM(Exam_Score * Exam_Score) - POW(SUM(Exam_Score), 2))
        )), 3
    ) AS Attendance_Correlation_Coefficient,

    ROUND((COUNT(*) * SUM(Hours_Studied * Exam_Score) - SUM(Hours_Studied) * SUM(Exam_Score)) / 
        (SQRT((COUNT(*) * SUM(Hours_Studied * Hours_Studied) - POW(SUM(Hours_Studied), 2)) *
              (COUNT(*) * SUM(Exam_Score * Exam_Score) - POW(SUM(Exam_Score), 2))
        )), 3
    ) AS Study_Correlation_Coefficient

FROM Growth_Analysis
GROUP BY School_Type, Socioeconomic_Status
HAVING COUNT(*) > 1
ORDER BY School_Type, Avg_Exam_Score DESC;
```
Python Validation

After computing correlations in SQL, I connected the dataset to Python for deeper validation.

   - Generated correlation heatmaps to detect multicollinearity (See my_heatmap.png)

   - Compared feature importance rankings to SQL correlation outputs

   - The modeling confirmed that hours studied and attendance were consistently strong predictors of exam performance.

Tableau Dashboard

I built an interactive dashboard focused on instructional equity.

   - It includes:

      - Equity gap scatter plots

      - Teacher quality vs income analysis

      - Risk segmentation quadrants

      - Filters for school type, gender, and learning disabilities

The goal was to make the findings usable for stakeholders rather than purely analytical.

Key Findings

Tutoring Impact
For high-risk students, tutoring sessions showed the strongest correlation with score improvement (0.72), making it the highest-leverage intervention identified.

Teacher Quality
High-quality teaching reduced the performance gap typically associated with lower income levels, suggesting teacher placement strategy plays a major role in equity.

Attendance Threshold
There is a noticeable performance drop once attendance falls below 80%, regardless of other factors.
