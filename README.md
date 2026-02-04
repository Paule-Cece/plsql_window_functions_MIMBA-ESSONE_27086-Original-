# plsql_window_functions_MIMBA-ESSONE_27086-Original-

**1. Business Problem Definition**

**Business Context**
*Company:* QuickCart E-Commerce
*Department:* Sales & Marketing Analytics
*Industry:* Online Retail

**Data Challenge**
The sales team lacks sufficient visibility into customer purchasing habits, product performance by region, and sales trends. This situation limits the effectiveness of inventory management, marketing targeting, and loyalty initiatives.

**Expected Outcome**
Identify top customers by region, analyze purchase frequency, segment customers based on their value, and track sales trends to optimize inventory and personalize marketing campaigns.

**✅ Step 2 – Success Criteria (exactly 5 goals – linked to window functions)**

1- Identify the top 5 products per region using RANK().
2- Compute running monthly sales totals per region using SUM() OVER().
3- Measure month-over-month sales growth using LAG().
4- Segment customers into four purchasing groups using NTILE(4).
5- Calculate three-month moving average of sales using AVG() OVER().

**3. Database Schema Design**
   
*Tables*

🧩 Clients

| Column    | Type    | Key        |
| ----------| ------- | -----------|
| client_id | PK      | Primary key|
| full_name | VARCHAR |            |
| region    | VARCHAR |            |
|signup_date| DATE    |            |

🧩Items 

| Column   | Type    | Key       |
| -------- | ------- | ----------|
| item_id  | PK      |Primary key|
| item_name| VARCHAR |           |
| category | VARCHAR |           |
| price    | NUMBER  |           |

🧩Transactions

| Column           | Type                    | Key        |
| ------------     | ------------------------| ---------- |
| transaction_id   | PK                      | Primary key|
| client_id        | FK → Clients(clients_id)| Foreign key|
| item_id          | FK → Items(items_id)    | Foreign key|
| transaction _date| DATE                    |            |
| quantity         | NUMBER                  |            |
| total_amount     | NUMBER                  |            |

**🔗 Relationships**
  .customers (1) —— (N) sales
  .products (1)  —— (N) sales

**👉 Your ER diagram must show:**
   .PK and FK
   .the two relationships above

 **✅ Step 4: Part A- SQL JOINs Implementation**

**4.1 INNER JOIN**
-- Active clients with valid transactions in 2024
SELECT c.client_name, i.item_name, t.transaction_date, t.total_amount
FROM Clients c
INNER JOIN Transactions t ON c.client_id = t.client_id
INNER JOIN Items i ON t.item_id = i.item_id
WHERE t.transaction_date >= '2024-01-01'
ORDER BY t.transaction_date;

*Business Interpretation:** 
Shows all active clients who made purchases in 2024 with item details. Helps identify engaged customers and popular products.

**4.2 LEFT JOIN**
-- Clients who have never made a purchase
SELECT c.client_name, c.region, c.signup_date
FROM Clients c
LEFT JOIN Transactions t ON c.client_id = t.client_id
WHERE t.transaction_id IS NULL
ORDER BY c.signup_date;

*Business Interpretation:**
Identifies registered clients who haven't made any purchases. Targets include Inactive1, Inactive2, Inactive3, Inactive4. Enables win-back campaigns.
 
**4.3 RIGHT JOIN**
-- Items with no sales activity
SELECT i.item_name, i.category, i.price
FROM Transactions t
RIGHT JOIN Items i ON t.item_id = i.item_id
WHERE t.transaction_id IS NULL;

*Business Interpretation:**
Finds unsold items: Unpopular Gadget, Expensive Art, Seasonal Item. Signals pricing or marketing issues requiring attention.

**4.4 FULL OUTER JOIN**
-- Complete view of client-item relationships
SELECT 
    COALESCE(c.client_name, 'NO CLIENT') AS client,
    COALESCE(i.item_name, 'NO ITEM') AS item,
    t.transaction_date,
    t.total_amount
FROM Clients c
FULL OUTER JOIN Transactions t ON c.client_id = t.client_id
FULL OUTER JOIN Items i ON t.item_id = i.item_id
WHERE c.client_id IS NULL OR i.item_id IS NULL OR t.transaction_id IS NULL
ORDER BY client, item;

*Business Interpretation:**
Reveals orphaned records - inactive clients and unsold items. Highlights data gaps for business strategy adjustment.

**4.5 SELF JOIN**
-- Clients from the same region
SELECT 
    c1.client_name AS client1,
    c2.client_name AS client2,
    c1.region,
    c1.signup_date AS client1_signup,
    c2.signup_date AS client2_signup
FROM Clients c1
INNER JOIN Clients c2 ON c1.region = c2.region 
    AND c1.client_id < c2.client_id
WHERE c1.client_name IN ('Mimba', 'Awassi', 'Shema', 'Teta')
    OR c2.client_name IN ('Mimba', 'Awassi', 'Shema', 'Teta')
ORDER BY c1.region, c1.client_name;

*Business Interpretation:**
Identifies client pairs in same regions. Mimba & Alice (North), Awassi & Bob (South), etc. Enables regional community building

**5 Part B — Window Functions Implementation**
   
**5.1 Ranking Functions**
-- Top 3 clients per region by total spending
WITH regional_spending AS (
    SELECT 
        c.region,
        c.client_name,
        SUM(t.total_amount) AS total_spent,
        COUNT(t.transaction_id) AS transaction_count,
        RANK() OVER (PARTITION BY c.region ORDER BY SUM(t.total_amount) DESC) AS rank_by_spending,
        DENSE_RANK() OVER (PARTITION BY c.region ORDER BY COUNT(t.transaction_id) DESC) AS rank_by_frequency
    FROM Clients c
    JOIN Transactions t ON c.client_id = t.client_id
    GROUP BY c.region, c.client_name
)
SELECT region, client_name, total_spent, transaction_count, 
       rank_by_spending, rank_by_frequency
FROM regional_spending
WHERE rank_by_spending <= 3
ORDER BY region, rank_by_spending;

*Interpretation:** 
Shema ranks #1 in East region by spending ($1,194.88). Mimba leads North region. Identifies regional champions for loyalty programs.

**5.2 Aggregate Window Functions**
-- Running monthly revenue with 3-month moving average
WITH monthly_sales AS (
    SELECT 
        DATE_TRUNC('month', transaction_date) AS month,
        SUM(total_amount) AS monthly_revenue,
        COUNT(DISTINCT client_id) AS active_clients
    FROM Transactions
    GROUP BY DATE_TRUNC('month', transaction_date)
)
SELECT 
    TO_CHAR(month, 'Mon YYYY') AS month,
    monthly_revenue,
    active_clients,
    SUM(monthly_revenue) OVER (
        ORDER BY month
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total,
    AVG(monthly_revenue) OVER (
        ORDER BY month
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3month
FROM monthly_sales
ORDER BY month;

*Interpretation:**
Revenue shows seasonal patterns with peaks in Jan ($1,149.93) and dips in Feb ($729.89). 3-month moving average smooths volatility for forecasting.

**5.3 Navigation Functions**
-- Month-over-month growth percentage
WITH monthly_revenue AS (
    SELECT 
        DATE_TRUNC('month', transaction_date) AS month,
        SUM(total_amount) AS revenue,
        COUNT(DISTINCT client_id) AS client_count
    FROM Transactions
    GROUP BY DATE_TRUNC('month', transaction_date)
)
SELECT 
    TO_CHAR(month, 'Mon YYYY') AS current_month,
    revenue AS current_revenue,
    client_count AS active_clients,
    LAG(revenue) OVER (ORDER BY month) AS previous_revenue,
    LAG(client_count) OVER (ORDER BY month) AS previous_clients,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month)) / 
        LAG(revenue) OVER (ORDER BY month) * 100, 
        2
    ) AS revenue_growth_pct,
    ROUND(
        (client_count - LAG(client_count) OVER (ORDER BY month)) / 
        NULLIF(LAG(client_count) OVER (ORDER BY month), 0) * 100, 
        2
    ) AS client_growth_pct
FROM monthly_revenue
ORDER BY month;

*Interpretation:**
February shows -36.5% revenue decline but client base grew 25%. March rebounds with 24.8% growth. Identifies successful client acquisition but lower spending.

**5.4 Distribution Functions**
-- Client segmentation by spending quartiles and cumulative distribution
WITH client_summary AS (
    SELECT 
        c.client_name,
        c.region,
        COUNT(t.transaction_id) AS transaction_count,
        SUM(t.total_amount) AS total_spent,
        MAX(t.transaction_date) AS last_purchase_date
    FROM Clients c
    LEFT JOIN Transactions t ON c.client_id = t.client_id
    GROUP BY c.client_id, c.client_name, c.region
)
SELECT 
    client_name,
    region,
    COALESCE(total_spent, 0) AS total_spent,
    transaction_count,
    NTILE(4) OVER (ORDER BY COALESCE(total_spent, 0) DESC) AS spending_quartile,
    CUME_DIST() OVER (ORDER BY COALESCE(total_spent, 0)) AS cumulative_distribution,
    CASE NTILE(4) OVER (ORDER BY COALESCE(total_spent, 0) DESC)
        WHEN 1 THEN 'Platinum'
        WHEN 2 THEN 'Gold'
        WHEN 3 THEN 'Silver'
        WHEN 4 THEN 'Bronze'
    END AS client_tier
FROM client_summary
ORDER BY total_spent DESC;

*Interpretation:**
Shema is Platinum tier (top 25%), Mimba & Awassi are Gold tier, Teta is Silver tier. Inactive clients fall in Bronze tier. Enables tiered marketing strategies.

**7. Results Analysis**

**7.1 Descriptive (What happened?)**
  .Regional Performance: East region generated highest revenue ($1,275.89) led by Shema
  .Top Product: Premium Headphones contributed 38% of total revenue
  .Client Engagement: 8 active clients vs 4 inactive (67% engagement rate)
  .Seasonality: January peak ($1,149.93), February dip ($729.89

**7.2 Diagnostic (Why did it happen?)**
  .East Region Success: Shema's multiple high-value purchases of Premium Headphones
  .February Dip: Fewer high-ticket items sold despite more clients
  .Inactive Clients: Signed up during promotional periods without converting
  .Unsold Items: Unpopular Gadget priced too high ($999.99)

**7.3 Prescriptive (What should be done next?)**
*Inventory Management:**
  .Increase Premium Headphones stock by 30%
  .Discontinue or discount Unpopular Gadget

*Marketing Strategy:**
 .Launch "Platinum Perks" program for Shema and Mimba
 .Reactivation campaign for inactive clients with 20% discount
 .Regional promotions targeting underperforming areas

*Pricing Optimization:**
 .Introduce bundle deals for slow-moving items
 .Seasonal pricing adjustments based on moving averages

*Client Retention:**
 .Personalize recommendations based on purchase history
 .Implement loyalty tiers with exclusive benefits

  **📚 References**
  
  *Institutional and official sources**
    PostgreSQL – Window Functions : postgresql.org/docs/current/sql-select.html – Section 7.4 sur les fonctions de fenêtre et les cadres de fenêtre.
    
  *Academic Resources**
  	Silberschatz, A., Korth, H.F., Sudarshan, S. "Database System Concepts", 7th ed. – McGraw Hill. Chapitre sur les expressions de fenêtre (Ch. 6).

  *Recommended tools and tutorials**
  	W3Schools – SQL JOINs : w3schools.com/sql/sql_join.asp – Tutorial interactif avec exemples visuels.

*📜 All sources used are properly cited.**
This work is entirely original and created by me.
Any content generated by artificial intelligence has been explicitly identified and used only as an aid, after adaptation.
 
