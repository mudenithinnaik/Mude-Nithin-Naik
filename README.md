--Name: Mude Nithin Naik
--CWID: 20030574

-- Report #1: Customer Min/Max Sales with Details and Average Quantity

WITH customer_stats AS (
    SELECT
        customer_id,
        MIN(sales_quantity) AS min_q,
        MAX(sales_quantity) AS max_q,
        AVG(sales_quantity) AS avg_q
    FROM
        sales
    GROUP BY
        customer_id
)

SELECT
    c.customer_name,
    cs.min_q,
    p_min.product_name AS min_prod,
    s_min.sale_date AS min_date,
    s_min.state AS min_state,
    cs.max_q,
    p_max.product_name AS max_prod,
    s_max.sale_date AS max_date,
    s_max.state AS max_state,
    cs.avg_q
FROM
    customer_stats cs
JOIN
    customers c ON cs.customer_id = c.customer_id
LEFT JOIN
    sales s_min ON cs.customer_id = s_min.customer_id AND cs.min_q = s_min.sales_quantity
LEFT JOIN
    products p_min ON s_min.product_id = p_min.product_id
LEFT JOIN
    sales s_max ON cs.customer_id = s_max.customer_id AND cs.max_q = s_max.sales_quantity
LEFT JOIN
    products p_max ON s_max.product_id = p_max.product_id
ORDER BY
    c.customer_name;



-- Report #2: Busiest and Slowest Months per Year

WITH yearly_monthly_sales AS (
    SELECT
        EXTRACT(YEAR FROM sale_date) AS year,
        EXTRACT(MONTH FROM sale_date) AS month,
        SUM(sales_quantity) AS total_sales
    FROM
        sales
    GROUP BY
        EXTRACT(YEAR FROM sale_date),
        EXTRACT(MONTH FROM sale_date)
),

busiest_months AS (
    SELECT
        yms.year,
        yms.month,
        yms.total_sales
    FROM
        yearly_monthly_sales yms
    WHERE
        yms.total_sales = (
            SELECT MAX(total_sales)
            FROM yearly_monthly_sales
            WHERE year = yms.year
        )
),

slowest_months AS (
    SELECT
        yms.year,
        yms.month,
        yms.total_sales
    FROM
        yearly_monthly_sales yms
    WHERE
        yms.total_sales = (
            SELECT MIN(total_sales)
            FROM yearly_monthly_sales
            WHERE year = yms.year
        )
)

SELECT
    b.year,
    b.month AS busiest_month,
    b.total_sales AS busiest_total_q,
    s.month AS slowest_month,
    s.total_sales AS slowest_total_q
FROM
    busiest_months b
JOIN
    slowest_months s ON b.year = s.year
ORDER BY
    b.year;




-- Report #3: Most and Least Popular Months per Product

WITH product_monthly_purchases AS (
    SELECT
        product_id,
        EXTRACT(MONTH FROM sale_date) AS month,
        COUNT(*) AS purchase_count
    FROM
        sales
    GROUP BY
        product_id,
        EXTRACT(MONTH FROM sale_date)
),

most_popular AS (
    SELECT
        pmp.product_id,
        pmp.month AS most_fav_mo,
        pmp.purchase_count
    FROM
        product_monthly_purchases pmp
    WHERE
        pmp.purchase_count = (
            SELECT MAX(pmp2.purchase_count)
            FROM product_monthly_purchases pmp2
            WHERE pmp2.product_id = pmp.product_id
        )
),

least_popular AS (
    SELECT
        pmp.product_id,
        pmp.month AS least_fav_mo,
        pmp.purchase_count
    FROM
        product_monthly_purchases pmp
    WHERE
        pmp.purchase_count = (
            SELECT MIN(pmp2.purchase_count)
            FROM product_monthly_purchases pmp2
            WHERE pmp2.product_id = pmp.product_id
        )
)

SELECT
    p.product_name,
    mp.most_fav_mo,
    lp.least_fav_mo
FROM
    products p
JOIN
    most_popular mp ON p.product_id = mp.product_id
JOIN
    least_popular lp ON p.product_id = lp.product_id
ORDER BY
    p.product_name;



-- Report #4: Average Sales by Season per Customer-Product

WITH sales_with_season AS (
    SELECT
        s.customer_id,
        s.product_id,
        s.sales_quantity,
        s.sale_date,
        s.state,
        EXTRACT(MONTH FROM s.sale_date) AS month,
        CASE
            WHEN EXTRACT(MONTH FROM s.sale_date) IN (3, 4, 5) THEN 'Spring'
            WHEN EXTRACT(MONTH FROM s.sale_date) IN (6, 7, 8) THEN 'Summer'
            WHEN EXTRACT(MONTH FROM s.sale_date) IN (9, 10, 11) THEN 'Fall'
            ELSE 'Winter'
        END AS season
    FROM
        sales s
),

spring_avg AS (
    SELECT
        customer_id,
        product_id,
        AVG(sales_quantity) AS spring_avg
    FROM
        sales_with_season
    WHERE
        season = 'Spring'
    GROUP BY
        customer_id, product_id
),

summer_avg AS (
    SELECT
        customer_id,
        product_id,
        AVG(sales_quantity) AS summer_avg
    FROM
        sales_with_season
    WHERE
        season = 'Summer'
    GROUP BY
        customer_id, product_id
),

fall_avg AS (
    SELECT
        customer_id,
        product_id,
        AVG(sales_quantity) AS fall_avg
    FROM
        sales_with_season
    WHERE
        season = 'Fall'
    GROUP BY
        customer_id, product_id
),

winter_avg AS (
    SELECT
        customer_id,
        product_id,
        AVG(sales_quantity) AS winter_avg
    FROM
        sales_with_season
    WHERE
        season = 'Winter'
    GROUP BY
        customer_id, product_id
),

overall_stats AS (
    SELECT
        customer_id,
        product_id,
        AVG(sales_quantity) AS average,
        SUM(sales_quantity) AS total,
        COUNT(*) AS count
    FROM
        sales_with_season
    GROUP BY
        customer_id, product_id
)

SELECT
    c.customer_name,
    p.product_name,
    sa.spring_avg,
    sm.summer_avg,
    fa.fall_avg,
    wa.winter_avg,
    os.average,
    os.total,
    os.count
FROM
    overall_stats os
JOIN
    customers c ON os.customer_id = c.customer_id
JOIN
    products p ON os.product_id = p.product_id
LEFT JOIN
    spring_avg sa ON os.customer_id = sa.customer_id AND os.product_id = sa.product_id
LEFT JOIN
    summer_avg sm ON os.customer_id = sm.customer_id AND os.product_id = sm.product_id
LEFT JOIN
    fall_avg fa ON os.customer_id = fa.customer_id AND os.product_id = fa.product_id
LEFT JOIN
    winter_avg wa ON os.customer_id = wa.customer_id AND os.product_id = wa.product_id
ORDER BY
    c.customer_name,
    p.product_name;






-- Report #5: Maximum Sales per Quarter per Product with Dates

WITH sales_with_quarter AS (
    SELECT
        s.product_id,
        s.sales_quantity,
        s.sale_date,
        EXTRACT(MONTH FROM s.sale_date) AS month,
        CASE
            WHEN EXTRACT(MONTH FROM s.sale_date) IN (1, 2, 3) THEN 'Q1'
            WHEN EXTRACT(MONTH FROM s.sale_date) IN (4, 5, 6) THEN 'Q2'
            WHEN EXTRACT(MONTH FROM s.sale_date) IN (7, 8, 9) THEN 'Q3'
            ELSE 'Q4'
        END AS quarter
    FROM
        sales s
),

max_q1 AS (
    SELECT
        product_id,
        MAX(sales_quantity) AS q1_max
    FROM
        sales_with_quarter
    WHERE
        quarter = 'Q1'
    GROUP BY
        product_id
),

max_q2 AS (
    SELECT
        product_id,
        MAX(sales_quantity) AS q2_max
    FROM
        sales_with_quarter
    WHERE
        quarter = 'Q2'
    GROUP BY
        product_id
),

max_q3 AS (
    SELECT
        product_id,
        MAX(sales_quantity) AS q3_max
    FROM
        sales_with_quarter
    WHERE
        quarter = 'Q3'
    GROUP BY
        product_id
),

max_q4 AS (
    SELECT
        product_id,
        MAX(sales_quantity) AS q4_max
    FROM
        sales_with_quarter
    WHERE
        quarter = 'Q4'
    GROUP BY
        product_id
),

q1_dates AS (
    SELECT
        s.product_id,
        s.sale_date,
        s.sales_quantity
    FROM
        sales_with_quarter s
    JOIN
        max_q1 mq1 ON s.product_id = mq1.product_id AND s.sales_quantity = mq1.q1_max
    WHERE
        s.quarter = 'Q1'
),

q2_dates AS (
    SELECT
        s.product_id,
        s.sale_date,
        s.sales_quantity
    FROM
        sales_with_quarter s
    JOIN
        max_q2 mq2 ON s.product_id = mq2.product_id AND s.sales_quantity = mq2.q2_max
    WHERE
        s.quarter = 'Q2'
),

q3_dates AS (
    SELECT
        s.product_id,
        s.sale_date,
        s.sales_quantity
    FROM
        sales_with_quarter s
    JOIN
        max_q3 mq3 ON s.product_id = mq3.product_id AND s.sales_quantity = mq3.q3_max
    WHERE
        s.quarter = 'Q3'
),

q4_dates AS (
    SELECT
        s.product_id,
        s.sale_date,
        s.sales_quantity
    FROM
        sales_with_quarter s
    JOIN
        max_q4 mq4 ON s.product_id = mq4.product_id AND s.sales_quantity = mq4.q4_max
    WHERE
        s.quarter = 'Q4'
)

SELECT
    p.product_name,
    mq1.q1_max,
    q1d.sale_date AS q1_date,
    mq2.q2_max,
    q2d.sale_date AS q2_date,
    mq3.q3_max,
    q3d.sale_date AS q3_date,
    mq4.q4_max,
    q4d.sale_date AS q4_date
FROM
    products p
LEFT JOIN
    max_q1 mq1 ON p.product_id = mq1.product_id
LEFT JOIN
    q1_dates q1d ON p.product_id = q1d.product_id AND q1d.sales_quantity = mq1.q1_max
LEFT JOIN
    max_q2 mq2 ON p.product_id = mq2.product_id
LEFT JOIN
    q2_dates q2d ON p.product_id = q2d.product_id AND q2d.sales_quantity = mq2.q2_max
LEFT JOIN
    max_q3 mq3 ON p.product_id = mq3.product_id
LEFT JOIN
    q3_dates q3d ON p.product_id = q3d.product_id AND q3d.sales_quantity = mq3.q3_max
LEFT JOIN
    max_q4 mq4 ON p.product_id = mq4.product_id
LEFT JOIN
    q4_dates q4d ON p.product_id = q4d.product_id AND q4d.sales_quantity = mq4.q4_max
ORDER BY
    p.product_name;



