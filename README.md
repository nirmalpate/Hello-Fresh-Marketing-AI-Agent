# Hello-Fresh-Marketing-AI-Agent
use role accountadmin;

create or replace role fresh_role;

grant role fresh_role to user  brandenburg2025;

create or replace warehouse hello_fresh
    with warehouse_size = 'medium'
    auto_suspend = 60
    auto_resume = true
    INITIALLY_SUSPENDED = TRUE;

grant usage on warehouse hello_fresh to role fresh_role;

create or replace database hello_database;

create or replace schema hello_database.raw;
create or replace schema hello_database.staging;
create or replace schema hello_database.marts;

ALTER SESSION SET DATE_INPUT_FORMAT = 'DD/MM/YYYY';

use database hello_database;

-- dim_geography
create or replace table hello_database.raw.dim_geography(
    geography_key integer,
    country_code varchar(5),
    country_name varchar(50),
    region varchar(25),
    currency_code varchar(15),
    created_at timestamp_ntz);

create or replace table hello_database.raw.dim_marketing_channel(
    channel_key integer,
    channel_id varchar(15),
    channel_name varchar(50),
    channel_type varchar(35),
    platform varchar(25),
    is_paid boolean,
    created_at timestamp_ntz);

-- dim_campaign

create or replace table hello_database.raw.dim_campaign(
    campaign_key integer,
    campaign_id  varchar(50),
    campaign_name varchar(50),
    channel_key integer,
    campaign_type varchar(50),
    campaign_start_date DATE,
    campaign_end_date date,
    budget_allocated integer,
    target_country varchar(10),
    create_at timestamp_ntz
);


create or replace table hello_database.raw.dum_customer(
    customer_key integer,
    customer_id varchar(15),
    first_order_date timestamp_ntz,
    country_code varchar(10),
    city varchar(15),
    customer_segment varchar(15),
    customer_status varchar(15),
    created_at timestamp_ntz,
    updated_at timestamp_ntz
);

create or replace table hello_database.raw.fact_customer_orders(
    order_id varchar(15),
    customer_key integer,
    order_date DATE,
    country_code varchar(12),
    order_number integer,
    is_first_order boolean,
    order_value decimal(10,4),
    discount_amount decimal(10,4),
    net_revenue decimal(10,4),
    attributed_channel_key integer,
    attributed_campaign_key integer,
    created_at timestamp_ntz);



    
CREATE TABLE hello_database.raw.FACT_MARKETING_PERFORMANCE (
    PERFORMANCE_ID INTEGER,
    DATE DATE,
    CHANNEL_KEY INTEGER,
    CAMPAIGN_KEY INTEGER, -- Nullable, will handle NULLs on load
    COUNTRY_CODE VARCHAR(2),
    SPEND DECIMAL(18, 4),
    IMPRESSIONS INTEGER,
    CLICKS INTEGER,
    CONVERSIONS INTEGER,
    REVENUE DECIMAL(18, 4),
    CTR DECIMAL(6, 4), -- Click-Through Rate
    CPC DECIMAL(10, 4), -- Cost Per Click
    CPA DECIMAL(10, 4), -- Cost Per Acquisition
    ROAS DECIMAL(10, 4), -- Return on Ad Spend
    CREATED_AT TIMESTAMP_NTZ
);

-- table fact_customer_cohorts

create or replace table hello_database.raw.fact_customer_cohorts(
    cohort_id integer,
    cohort_month date,
    month_number integer,
    channel_key integer,
    country_code varchar(15),
    cohort_size integer,
    active_customers integer,
    retention_rate decimal(10,4),
    cohort_revenue decimal(18,4),
    avg_revenue_per_customer decimal(10,4),
    created_at timestamp_ntz

);


-- INSERT CLEANED DATA IN MARTS;

create or replace table hello_database.Marts.dim_geography(
    geography_key integer,
    country_code varchar(5),
    country_name varchar(50),
    region varchar(25),
    currency_code varchar(15),
    created_at timestamp_ntz);

 
INSERT INTO hello_database.marts.dim_geography
SELECT
    geography_key,
    country_code,
    country_name,
    region,
    currency_code,
    created_at
    from hello_database.raw.dim_geography;


create or replace table hello_database.marts.dim_marketing_channel(
    channel_key integer,
    channel_id varchar(15),
    channel_name varchar(50),
    channel_type varchar(35),
    platform varchar(25),
    is_paid boolean,
    created_at timestamp_ntz);

insert into hello_database.marts.dim_marketing_channel(
    select 
    channel_key,
    channel_id,
    channel_name,
    channel_type,
    platform,
    is_paid,
    created_at
from hello_database.raw.dim_marketing_channel);

create or replace table hello_database.marts.dim_campaign(
    campaign_key integer,
    campaign_id  varchar(50),
    campaign_name varchar(50),
    channel_key integer,
    campaign_type varchar(50),
    campaign_start_date DATE,
    campaign_end_date date,
    budget_allocated integer,
    target_country varchar(10),
    create_at timestamp_ntz
);

insert into hello_database.marts.dim_campaign(
    select 
        campaign_key,
        campaign_id,
        campaign_name,
        channel_key,
        campaign_type,
        campaign_start_date,
        campaign_end_date,
        budget_allocated,
        target_country,
        create_at
        from hello_database.raw.dim_campaign);



--MARKETING VIEW
 

create view marketing as
select  dc.campaign_id,
    dc.campaign_key,
    dc.campaign_name,
    dc.campaign_start_date,
    dc.campaign_end_date,
    dc.budget_allocated,
    dmc.channel_key,
    dmc.channel_name,
    dmc.channel_type
from hello_database.marts.dim_marketing_channel dmc
join hello_database.marts.dim_campaign dc 
on dmc.channel_key = dc.channel_key;


create or replace table hello_database.MARTS.dum_customer(
    customer_key integer,
    customer_id varchar(15),
    first_order_date timestamp_ntz,
    country_code varchar(10),
    city varchar(15),
    customer_segment varchar(15),
    customer_status varchar(15),
    created_at timestamp_ntz,
    updated_at timestamp_ntz
);

insert into hello_database.MARTS.DUM_CUSTOMER(
    select CUSTOMER_KEY,
           CUSTOMER_ID,
           FIRST_ORDER_DATE,
           COUNTRY_CODE,
           CITY,
           CUSTOMER_SEGMENT,
           CUSTOMER_STATUS,
           CREATED_AT,
           UPDATED_AT
        FROM HELLO_DATABASE.RAW.DUM_CUSTOMER
        
        );

create or replace table hello_database.MARTS.fact_customer_orders(
    order_id varchar(15),
    customer_key integer,
    order_date DATE,
    country_code varchar(12),
    order_number integer,
    is_first_order boolean,
    order_value decimal(10,4),
    discount_amount decimal(10,4),
    net_revenue decimal(10,4),
    attributed_channel_key integer,
    attributed_campaign_key integer,
    created_at timestamp_ntz);


INSERT INTO HELLO_DATABASE.MARTS.FACT_CUSTOMER_ORDERS(
    SELECT 
        ORDER_ID,
        CUSTOMER_KEY,
        ORDER_DATE,
        COUNTRY_CODE,
        ORDER_NUMBER,
        IS_FIRST_ORDER,
        ORDER_VALUE,
        DISCOUNT_AMOUNT,
        NET_REVENUE,
        ATTRIBUTED_CHANNEL_KEY,
        ATTRIBUTED_CAMPAIGN_KEY,
        CREATED_AT
    FROM HELLO_DATABASE.RAW.FACT_CUSTOMER_ORDERS);

 

 CREATE TABLE hello_database.marts.FACT_MARKETING_PERFORMANCE (
    PERFORMANCE_ID INTEGER,
    DATE DATE,
    CHANNEL_KEY INTEGER,
    CAMPAIGN_KEY INTEGER, -- Nullable, will handle NULLs on load
    COUNTRY_CODE VARCHAR(2),
    SPEND DECIMAL(18, 4),
    IMPRESSIONS INTEGER,
    CLICKS INTEGER,
    CONVERSIONS INTEGER,
    REVENUE DECIMAL(18, 4),
    CTR DECIMAL(6, 4), -- Click-Through Rate
    CPC DECIMAL(10, 4), -- Cost Per Click
    CPA DECIMAL(10, 4), -- Cost Per Acquisition
    ROAS DECIMAL(10, 4), -- Return on Ad Spend
    CREATED_AT TIMESTAMP_NTZ
);


INSERT INTO HELLO_DATABASE.MARTS.FACT_MARKETING_PERFORMANCE(
    SELECT 
        PERFORMANCE_ID,
        DATE,
        CHANNEL_KEY,
        CAMPAIGN_KEY,
        COUNTRY_CODE,
        SPEND,
        IMPRESSIONS,
        CLICKS, CONVERSIONS,
        REVENUE,
        CTR,
        CPC,
        CPA,
        ROAS,
        CREATED_AT
    FROM HELLO_DATABASE.RAW.FACT_MARKETING_PERFORMANCE);


CREATE OR REPLACE VIEW hello_database.marts.marketing_view AS
SELECT 
    C1.channel_key,
    C1.campaign_id,
    C1.campaign_name,
    C1.campaign_start_date,
    C1.campaign_type,
    C1.campaign_end_date,
    C1.budget_allocated,
    C2.channel_name,
    C2.channel_type,
    C2.is_paid,
    C2.platform,
    C3.campaign_key AS fact_campaign_key,
    C3.clicks,
    C3.conversions,
    C3.country_code,
    C3.cpa,
    C3.cpc,
    C3.ctr,
    C3.date,
    C3.impressions,
    C3.performance_id,
    C3.revenue,
    C3.spend
FROM hello_database.marts.dim_campaign C1
JOIN hello_database.marts.dim_marketing_channel C2
    ON C1.channel_key = C2.channel_key
JOIN hello_database.marts.fact_marketing_performance C3
    ON C3.campaign_key = C1.campaign_key;


create or replace table hello_database.MARTS.fact_customer_cohorts(
    cohort_id integer,
    cohort_month date,
    month_number integer,
    channel_key integer,
    country_code varchar(15),
    cohort_size integer,
    active_customers integer,
    retention_rate decimal(10,4),
    cohort_revenue decimal(18,4),
    avg_revenue_per_customer decimal(10,4),
    created_at timestamp_ntz

);


INSERT INTO HELLO_DATABASE.MARTS.FACT_CUSTOMER_COHORTS(
    SELECT COHORT_ID,
           COHORT_MONTH,
           MONTH_NUMBER,
           CHANNEL_KEY,
           COUNTRY_CODE,
           COHORT_SIZE,
           ACTIVE_CUSTOMERS,
           RETENTION_RATE,
           COHORT_REVENUE,
           AVG_REVENUE_PER_CUSTOMER,
           CREATED_AT
        FROM HELLO_DATABASE.RAW.FACT_CUSTOMER_COHORTS);
    
 

-- FACT_CAMPAIGN_PERFORMANCE PART 2



CREATE OR REPLACE VIEW MARTS.FACT_CAMPAIGN_PERFORMANCE AS
SELECT 
    -- Grain fields (One row per day per campaign per channel per country)
    C3.DATE,
    C1.CAMPAIGN_KEY,
    C1.CHANNEL_KEY,
    C3.COUNTRY_CODE,
    
    -- Core metrics (aggregated to daily grain)
    SUM(C3.IMPRESSIONS) AS IMPRESSIONS,
    SUM(C3.CLICKS) AS CLICKS,
    SUM(C3.CONVERSIONS) AS CONVERSIONS,
    SUM(C3.REVENUE) AS REVENUE,
    SUM(C3.SPEND) AS SPEND,
    
    -- Efficiency metrics (division-safe)
    ZEROIFNULL(SUM(C3.REVENUE)) / NULLIF(SUM(C3.SPEND), 0) AS ROAS,
    ZEROIFNULL(SUM(C3.CLICKS)) / NULLIF(SUM(C3.IMPRESSIONS), 0) AS CTR,
    ZEROIFNULL(SUM(C3.SPEND)) / NULLIF(SUM(C3.CLICKS), 0) AS CPC,
    ZEROIFNULL(SUM(C3.SPEND)) / NULLIF(SUM(C3.CONVERSIONS), 0) AS CPA,
    
    -- Business metrics
    ZEROIFNULL(SUM(C3.REVENUE)) / NULLIF(SUM(C3.CONVERSIONS), 0) AS AVG_ORDER_VALUE,
    
    -- Metadata
    CURRENT_TIMESTAMP() AS LAST_UPDATED
    
FROM DIM_CAMPAIGN C1
JOIN FACT_MARKETING_PERFORMANCE C3 
    ON C3.CAMPAIGN_KEY = C1.CAMPAIGN_KEY
    
GROUP BY 
    C3.DATE,
    C1.CAMPAIGN_KEY,
    C1.CHANNEL_KEY,
    C3.COUNTRY_CODE;


SELECT *
FROM HELLO_DATABASE.MARTS.FACT_CAMPAIGN_PERFORMANCE
ORDER BY DATE;




---- ✅ ANALYTICS schema — Model-ready features for Python
---- ✅ DECISIONS schema — Stores automated recommendations


CREATE SCHEMA IF NOT EXISTS ANALYTICS;


-- ANALYTICAL SCHEMA (ENGINEERED FEATURES FOR PYTHON SCHEMA)
-- GRAIN : DAILY PER CAMPAIGN PER CHANNEL



CREATE OR REPLACE VIEW HELLO_DATABASE.ANALYTICS.CAMPAIGN_FEATURES AS


SELECT F1.DATE,
       F1.CAMPAIGN_KEY,
       F1.COUNTRY_CODE,
       F1.CHANNEL_KEY,
       -- CURRENT PERFORMANCE
       F1.REVENUE,
       F1.SPEND,
       F1.CONVERSIONS,
       F1.ROAS,
       F1.CPA,
       F1.CTR,
       -- ROLLING AVERAGES

       AVG(F1.ROAS) OVER(PARTITION BY F1.CAMPAIGN_KEY ORDER BY F1.DATE ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ROLLING_7DAY_ROAS,
       AVG(F1.SPEND) OVER(PARTITION BY F1.CAMPAIGN_KEY ORDER BY F1.DATE ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ROLLING_7DAY_SPEND,
       AVG(F1.REVENUE) OVER(PARTITION BY F1.CAMPAIGN_KEY ORDER BY F1.DATE ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ROLLING_7DAY_REVENUE,

       -- PERFORMANCE TRENDS

       CASE WHEN F1.ROAS > AVG(F1.ROAS) OVER(PARTITION BY F1.CAMPAIGN_KEY ORDER BY F1.DATE ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) THEN 'IMPROVING'
            WHEN F1.ROAS < AVG(F1.ROAS) OVER(PARTITION BY F1.CAMPAIGN_KEY ORDER BY F1.DATE ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) THEN 'DECLINING'
            ELSE 'STABLE'
            END AS PERFORMANCE_TREND,
        -- BUDGET ALLOCATION

        F2.BUDGET_ALLOCATED,
        SUM(F1.SPEND) OVER(PARTITION BY F1.CAMPAIGN_KEY ORDER BY F1.DATE) AS CUMULATIVE_SPEND,
        SUM(F1.SPEND) OVER(PARTITION BY F1.CAMPAIGN_KEY ORDER BY F1.DATE) / F1.SPEND AS BUDGET_UTILISATION_RATE,

        -- CAMPAIGN AGE

        DATEDIFF(DAY, F2.CAMPAIGN_START_DATE,F1.DATE)AS DAYS_SINCE_START,

        -- SEASONALITY INDICATOR

        DAYOFWEEK(F1.DATE) AS DAY_OF_WEEK,
        CASE WHEN DAYOFWEEK(F1.DATE) IN (0,6) THEN 'WEEKEND'
             ELSE 'WEEKDAY'
             END AS DAY_TYPE

FROM HELLO_DATABASE.MARTS.FACT_CAMPAIGN_PERFORMANCE F1
JOIN HELLO_DATABASE.MARTS.DIM_CAMPAIGN F2
ON F1.CAMPAIGN_KEY = F2.CAMPAIGN_KEY;

SELECT *
FROM HELLO_DATABASE.ANALYTICS.CAMPAIGN_FEATURES
    
-- ----------------------------------------------------------------------------
-- ANALYTICS.CHANNEL_PERFORMANCE_SUMMARY
-- Purpose: Channel-level aggregates for optimization logic
-- Grain: Daily per channel
-- ----------------------------------------------------------------------------


CREATE OR REPLACE VIEW ANALYTICS.CHANNEL_PERFORMANCE_SUMMARY AS
WITH DailyChannelMetrics AS (
    -- Step 1: Perform all standard aggregations
    SELECT 
        F.DATE,
        F.CHANNEL_KEY,
        CH.CHANNEL_NAME,
        CH.CHANNEL_TYPE,
        CH.IS_PAID,
        COUNT(DISTINCT F.CAMPAIGN_KEY) AS ACTIVE_CAMPAIGNS,
        SUM(F.SPEND) AS TOTAL_SPEND,
        SUM(F.REVENUE) AS TOTAL_REVENUE,
        SUM(F.CONVERSIONS) AS TOTAL_CONVERSIONS,
        -- Calculate the average ROAS for the group to be used in the window function
        AVG(F.ROAS) AS AVG_DAILY_ROAS, 
        STDDEV(F.ROAS) AS ROAS_VOLATILITY,
        MIN(F.ROAS) AS MIN_CAMPAIGN_ROAS,
        MAX(F.ROAS) AS MAX_CAMPAIGN_ROAS,
        MEDIAN(F.ROAS) AS MEDIAN_CAMPAIGN_ROAS
    FROM MARTS.FACT_CAMPAIGN_PERFORMANCE F
    JOIN MARTS.DIM_MARKETING_CHANNEL CH 
        ON F.CHANNEL_KEY = CH.CHANNEL_KEY
    GROUP BY 
        F.DATE,
        F.CHANNEL_KEY,
        CH.CHANNEL_NAME,
        CH.CHANNEL_TYPE,
        CH.IS_PAID
)
-- Step 2: Apply the Window Function to the aggregated results
SELECT 
    *,
    -- Efficiency ratios calculated after aggregation to avoid division errors
    ZEROIFNULL(TOTAL_REVENUE) / NULLIF(TOTAL_SPEND, 0) AS CHANNEL_ROAS,
    ZEROIFNULL(TOTAL_SPEND) / NULLIF(TOTAL_CONVERSIONS, 0) AS CHANNEL_CPA,
    
    -- Now we can average the AVG_DAILY_ROAS over 7 days
    AVG(AVG_DAILY_ROAS) OVER (
        PARTITION BY CHANNEL_KEY 
        ORDER BY DATE 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS ROLLING_7DAY_CHANNEL_ROAS,
    
    CURRENT_TIMESTAMP() AS LAST_UPDATED
FROM DailyChannelMetrics;

 SELECT * FROM
 HELLO_DATABASE.ANALYTICS.CHANNEL_PERFORMANCE_SUMMARY

 -- DEICISON SCHEMA 
 -- STORES BUDGET RECOMMENDATIONS GIVEN BY AGENTS
 
CREATE SCHEMA IF NOT EXISTS DECISIONS;


CREATE OR REPLACE TABLE DECISIONS.CAMPAIGN_RECOMMENDATIONS (
    RECOMMENDATION_ID VARCHAR(50) PRIMARY KEY,
    DATE_GENERATED TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    EFFECTIVE_DATE DATE NOT NULL,
    
    -- Campaign identifiers
    CAMPAIGN_KEY NUMBER(38,0),
    CHANNEL_KEY NUMBER(38,0),
    COUNTRY_CODE VARCHAR(3),
    
    -- Current state
    CURRENT_DAILY_BUDGET NUMBER(18,2),
    CURRENT_ROAS NUMBER(10,4),
    
    -- Recommendation
    RECOMMENDED_DAILY_BUDGET NUMBER(18,2),
    BUDGET_CHANGE_AMOUNT NUMBER(18,2),
    BUDGET_CHANGE_PCT NUMBER(10,2),
    
    -- Projections
    EXPECTED_ROAS NUMBER(10,4),
    EXPECTED_DAILY_REVENUE NUMBER(18,2),
    EXPECTED_DAILY_CONVERSIONS NUMBER(10,2),
    
    -- Decision rationale
    DECISION_REASON VARCHAR(500),
    AGENT_TYPE VARCHAR(50), -- 'PLANNING' or 'OPTIMIZATION'
    CONFIDENCE_SCORE NUMBER(5,2), -- 0-100
    
    -- Guardrail checks
    PASSED_GUARDRAILS BOOLEAN,
    GUARDRAIL_NOTES VARCHAR(500),
    
    -- Implementation status
    STATUS VARCHAR(20) DEFAULT 'PENDING', -- PENDING, APPROVED, REJECTED, IMPLEMENTED
    IMPLEMENTED_AT TIMESTAMP_NTZ,
    
    -- Metadata
    CREATED_BY VARCHAR(100) DEFAULT 'PYTHON_AGENT',
    MODEL_VERSION VARCHAR(20)
);



CREATE OR REPLACE TABLE DECISIONS.CAMPAIGN_SCENARIOS (
    SCENARIO_ID VARCHAR(50) PRIMARY KEY,
    DATE_GENERATED TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    SIMULATION_DATE DATE NOT NULL,
    
    -- Campaign identifiers
    CAMPAIGN_KEY NUMBER(38,0),
    CHANNEL_KEY NUMBER(38,0),
    
    -- Scenario definition
    SCENARIO_TYPE VARCHAR(50), -- BASE, BEST_CASE, WORST_CASE, DEMAND_SHOCK, COST_INCREASE, BUDGET_CUT
    SCENARIO_NAME VARCHAR(100),
    
    -- Baseline (current/expected)
    BASELINE_REVENUE NUMBER(18,2),
    BASELINE_SPEND NUMBER(18,2),
    BASELINE_ROAS NUMBER(10,4),
    
    -- Scenario results
    SCENARIO_REVENUE NUMBER(18,2),
    SCENARIO_SPEND NUMBER(18,2),
    SCENARIO_ROAS NUMBER(10,4),
    
    -- Impact analysis
    REVENUE_CHANGE_AMOUNT NUMBER(18,2),
    REVENUE_CHANGE_PCT NUMBER(10,2),
    ROAS_CHANGE NUMBER(10,4),
    
    -- Scenario parameters (stored as JSON)
    ASSUMPTIONS VARIANT, -- e.g., {"demand_drop_pct": -20, "duration_days": 14}
    
    -- Risk assessment
    LIKELIHOOD VARCHAR(20), -- HIGH, MEDIUM, LOW
    IMPACT_SEVERITY VARCHAR(20), -- CRITICAL, HIGH, MEDIUM, LOW
    
    -- Recommended actions
    MITIGATION_STRATEGY VARCHAR(500),
    
    -- Metadata
    CREATED_BY VARCHAR(100) DEFAULT 'SCENARIO_AGENT',
    MODEL_VERSION VARCHAR(20)
);


-- BUDGET_ALLOCATION

-- CHANNEL LEVEL BUDGET ALLOCATION

CREATE OR REPLACE TABLE DECISIONS.BUDGET_ALLOCATION (
    ALLOCATION_ID VARCHAR(50) PRIMARY KEY,
    DATE_GENERATED TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    EFFECTIVE_DATE DATE NOT NULL,
    
    -- Channel identifiers
    CHANNEL_KEY NUMBER(38,0),
    CHANNEL_NAME VARCHAR(100),
    
    -- Current allocation
    CURRENT_BUDGET NUMBER(18,2),
    CURRENT_ALLOCATION_PCT NUMBER(5,2),
    CURRENT_CHANNEL_ROAS NUMBER(10,4),
    
    -- Recommended allocation
    RECOMMENDED_BUDGET NUMBER(18,2),
    RECOMMENDED_ALLOCATION_PCT NUMBER(5,2),
    EXPECTED_CHANNEL_ROAS NUMBER(10,4),
    
    -- Changes
    REALLOCATION_AMOUNT NUMBER(18,2),
    REALLOCATION_PCT NUMBER(10,2),
    
    -- Justification
    JUSTIFICATION VARCHAR(500),
    PRIORITY_LEVEL VARCHAR(20), -- HIGH, MEDIUM, LOW
    
    -- Impact projection
    EXPECTED_REVENUE_IMPACT NUMBER(18,2),
    EXPECTED_EFFICIENCY_GAIN NUMBER(10,4),
    
    -- Status
    STATUS VARCHAR(20) DEFAULT 'PENDING',
    
    -- Metadata
    CREATED_BY VARCHAR(100) DEFAULT 'OPTIMIZATION_AGENT',
    MODEL_VERSION VARCHAR(20)
);


-- ----------------------------------------------------------------------------
-- DECISIONS.AGENT_EXECUTION_LOG
-- Purpose: Tracks agent runs for monitoring and debugging
-- ----------------------------------------------------------------------------

CREATE OR REPLACE TABLE DECISIONS.AGENT_EXECUTION_LOG (
    EXECUTION_ID VARCHAR(50) PRIMARY KEY,
    EXECUTION_START TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    EXECUTION_END TIMESTAMP_NTZ,
    
    AGENT_NAME VARCHAR(50), -- PLANNING, OPTIMIZATION, SCENARIO, EXPLANATION
    EXECUTION_STATUS VARCHAR(20), -- SUCCESS, FAILED, PARTIAL
    
    RECORDS_PROCESSED NUMBER(10,0),
    RECOMMENDATIONS_GENERATED NUMBER(10,0),
    
    ERROR_MESSAGE VARCHAR(1000),
    EXECUTION_DURATION_SECONDS NUMBER(10,2),
    
    -- Configuration used
    GUARDRAILS_CONFIG VARIANT,
    MODEL_VERSION VARCHAR(20)
);


 SHOW DATABASES LIKE 'HELLO_DATABASE';


SELECT * FROM HELLO_DATABASE.DECISIONS.CAMPAIGN_RECOMMENDATIONS 
ORDER BY DATE_GENERATED DESC 
LIMIT 10;


select *
from hello_database.decisions.budget_allocation;


 Creating a professional README for an AI agent requires a balance between technical setup and business value. Below is a structured template tailored for your **Snowflake Python AI Agent** for marketing prediction.

---

# 🚀 Snowflake Marketing AI Agent

An autonomous Python-based AI agent designed to run directly within the **Snowflake Data Cloud**. This agent leverages **Snowpark Python** and **Cortex AI** to analyze historical marketing data, predict campaign performance, and provide actionable optimization strategies.

 

---

## 🌟 Overview

This agent bridges the gap between raw marketing data and strategic decision-making. By utilizing Snowflake's native processing power, it avoids data movement and provides high-concurrency predictions for:

* **Customer Churn:** Identifying at-risk segments before they leave.
* **Lead Scoring:** Prioritizing high-value prospects for sales teams.
* **Budget Optimization:** Predicting which channels will yield the highest ROI.

## ✨ Key Features

* **Native Snowflake Execution:** Built using Snowpark Python for secure, in-warehouse processing.
* **Agentic Reasoning:** Uses LLM-based orchestration to choose between SQL tools and Python prediction scripts.
* **Natural Language Interface:** Marketers can ask, *"Which campaign should I increase the budget for next month?"*
* **Predictive Modeling:** Integrated Scikit-learn/XGBoost models for accurate forecasting.

## 🏗 Architecture

The agent follows a modular design to ensure scalability and security.

1. **Snowflake Cortex:** Handles the LLM reasoning and natural language processing.
2. **Snowpark Python:** Executes the heavy-lifting predictive logic and data transformations.
3. **Streamlit (Optional):** Provides a front-end UI for non-technical users to interact with the agent.

---

## ⚙️ Prerequisites

Before deployment, ensure you have the following:

* **Snowflake Account** with Anaconda integration enabled.
* **Privileges:** `CREATE FUNCTION`, `CREATE PROCEDURE`, and `USAGE` on the target warehouse.
* **Python 3.8+** for local development and testing.

## 🚀 Installation & Setup

1. **Clone the Repository**
```bash
git clone https://github.com/your-username/snowflake-marketing-agent.git
cd snowflake-marketing-agent

```


2. **Configure Snowflake Credentials**
Create a `config.py` or use environment variables:
```python
SNOWFLAKE_CONN_PARAMS = {
    "account": "your_account_locator",
    "user": "your_user",
    "password": "your_password",
    "role": "MARKETING_ANALYST",
    "warehouse": "COMPUTE_WH",
    "database": "MARKETING_DB",
    "schema": "AI_AGENT"
}

```


3. **Deploy to Snowflake**
Run the setup script to create the necessary Stages and UDFs:
```bash
python deploy_agent.py

```



---

## 🛠 Usage

Once deployed, you can interact with the agent via the Snowflake Python API or a Streamlit interface.

**Example Query:**

```python
response = agent.ask("Predict the conversion rate for our 'Summer Sale' email campaign.")
print(response)

```

**Common Interaction Tasks:**

* "Analyze why the engagement dropped in the Northeast region."
* "Generate a forecast for Q4 ad spend based on last year's performance."

---

## 🛡 Safety & Governance

* **RBAC Compliance:** The agent strictly follows Snowflake’s Role-Based Access Control. It only sees data the executing role is permitted to see.
* **Data Privacy:** No data leaves the Snowflake security perimeter; all LLM processing is done via Snowflake Cortex.
* **Auditability:** Every prediction and query is logged in the `AGENT_LOGS` table for transparency.

---

 
