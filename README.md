# Sergeeva_E_B_DZ1_SXD
CREATE TABLE customers (
    customer_id           integer,
    first_name            varchar(50),
    last_name             varchar(50),
    gender                varchar(20),
    dob                   date,
    job_title             varchar(100),
    job_industry_category varchar(50),
    wealth_segment        varchar(30),
    deceased_indicator    boolean,
    owns_car              boolean,
    address               varchar(200),
    postcode              varchar(10),
    state                 varchar(10),
    country               varchar(50),
    property_valuation    numeric(4,2)
);

CREATE TABLE products (
    product_id    integer,
    brand         varchar(50),
    product_line  varchar(30),
    product_class varchar(30),
    product_size  varchar(10),
    list_price    numeric(10,2),
    standard_cost numeric(10,2)
);

CREATE TABLE orders (
    order_id     integer,
    customer_id  integer,
    order_date   timestamp,
    online_order boolean,
    order_status varchar(20)
);

CREATE TABLE order_items (
    order_item_id              integer,
    order_id                   integer,
    product_id                 integer,
    quantity                   integer,
    item_list_price_at_sale    numeric(10,2),
    item_standard_cost_at_sale numeric(10,2)
);

SELECT 'customers' AS tbl, COUNT(*) FROM customers
UNION ALL SELECT 'products',    COUNT(*) FROM products
UNION ALL SELECT 'orders',      COUNT(*) FROM orders
UNION ALL SELECT 'order_items', COUNT(*) FROM order_items;

SELECT 'customers' AS tbl, COUNT(*) AS rows_total,
       COUNT(DISTINCT customer_id) AS uniq_keys,
       COUNT(*) - COUNT(DISTINCT customer_id) AS duplicates
FROM customers
UNION ALL
SELECT 'products', COUNT(*), COUNT(DISTINCT product_id),
       COUNT(*) - COUNT(DISTINCT product_id) FROM products
UNION ALL
SELECT 'orders', COUNT(*), COUNT(DISTINCT order_id),
       COUNT(*) - COUNT(DISTINCT order_id) FROM orders
UNION ALL
SELECT 'order_items', COUNT(*), COUNT(DISTINCT order_item_id),
       COUNT(*) - COUNT(DISTINCT order_item_id) FROM order_items;

-- первичные ключи
ALTER TABLE customers   ADD PRIMARY KEY (customer_id);
ALTER TABLE products    ADD PRIMARY KEY (product_id);
ALTER TABLE orders      ADD PRIMARY KEY (order_id);
ALTER TABLE order_items ADD PRIMARY KEY (order_item_id);

-- внешние ключи
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id) REFERENCES customers (customer_id);

ALTER TABLE order_items
    ADD CONSTRAINT fk_items_order
    FOREIGN KEY (order_id) REFERENCES orders (order_id);

ALTER TABLE order_items
    ADD CONSTRAINT fk_items_product
    FOREIGN KEY (product_id) REFERENCES products (product_id);

SELECT tc.table_name, tc.constraint_type, tc.constraint_name, kcu.column_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
  ON tc.constraint_name = kcu.constraint_name
WHERE tc.table_schema = 'public'
ORDER BY tc.table_name, tc.constraint_type DESC;

SELECT state, COUNT(DISTINCT country)
FROM customers GROUP BY state
HAVING COUNT(DISTINCT country) > 1;

SELECT postcode, STRING_AGG(DISTINCT state, ', ') AS states
FROM customers GROUP BY postcode
HAVING COUNT(DISTINCT state) > 1;

SELECT job_title, COUNT(DISTINCT job_industry_category) AS industries
FROM customers GROUP BY job_title
HAVING COUNT(DISTINCT job_industry_category) > 1;

-- справочник штатов
CREATE TABLE states (
    state   varchar(10) PRIMARY KEY,
    country varchar(50) NOT NULL
);

INSERT INTO states (state, country)
SELECT DISTINCT state, country FROM customers;

-- связь клиентов со справочником и удаление избыточного поля
ALTER TABLE customers
    ADD CONSTRAINT fk_customers_state
    FOREIGN KEY (state) REFERENCES states (state);

ALTER TABLE customers DROP COLUMN country;

SELECT EXTRACT(YEAR  FROM order_date) AS order_year,
       EXTRACT(MONTH FROM order_date) AS order_month,
       COUNT(*) AS orders_count
FROM orders
GROUP BY order_year, order_month
ORDER BY order_year, order_month;

SELECT product_id, product_line, standard_cost
FROM (
    SELECT product_id, product_line, standard_cost,
           RANK() OVER (PARTITION BY product_line
                        ORDER BY standard_cost DESC) AS rnk
    FROM products
) t
WHERE rnk = 1
ORDER BY product_line;

SELECT o.order_status,
       COUNT(DISTINCT o.order_id) AS orders_count,
       SUM(oi.item_list_price_at_sale * oi.quantity) AS total_amount
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY o.order_status
ORDER BY o.order_status;

SELECT c.customer_id, c.last_name, c.first_name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL
ORDER BY c.customer_id;

SELECT state,
       CASE WHEN property_valuation = max_val THEN 'max' ELSE 'min' END AS val_type,
       property_valuation, last_name, first_name
FROM (
    SELECT state, property_valuation, last_name, first_name,
           MAX(property_valuation) OVER (PARTITION BY state) AS max_val,
           MIN(property_valuation) OVER (PARTITION BY state) AS min_val
    FROM customers
) t
WHERE property_valuation IN (max_val, min_val)
ORDER BY state, val_type;

SELECT o.customer_id,
       MIN(p.product_line)        AS product_line,
       COUNT(DISTINCT o.order_id) AS orders_count
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p    ON p.product_id = oi.product_id
GROUP BY o.customer_id
HAVING COUNT(DISTINCT p.product_line) = 1
ORDER BY o.customer_id;

SELECT p.product_line,
       COUNT(DISTINCT o.order_id) AS orders_total,
       COUNT(DISTINCT o.order_id) FILTER (WHERE o.order_status = 'Cancelled') AS orders_cancelled,
       ROUND(100.0 * COUNT(DISTINCT o.order_id) FILTER (WHERE o.order_status = 'Cancelled')
             / COUNT(DISTINCT o.order_id), 2) AS cancelled_pct
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p    ON p.product_id = oi.product_id
GROUP BY p.product_line
ORDER BY cancelled_pct DESC;

SELECT segment, COUNT(*) AS customers_count
FROM (
    SELECT c.customer_id,
           CASE
               WHEN COUNT(o.order_id) > 10 THEN 'High'
               WHEN COUNT(o.order_id) BETWEEN 5 AND 10 THEN 'Medium'
               ELSE 'Low'
           END AS segment
    FROM customers c
    LEFT JOIN orders o ON o.customer_id = c.customer_id
    GROUP BY c.customer_id
) t
GROUP BY segment
ORDER BY customers_count DESC;

SELECT c.first_name, c.last_name, c.job_title,
       MAX(t.gap_days) AS max_gap_days
FROM (
    SELECT customer_id,
           order_date::date
             - LAG(order_date::date) OVER (PARTITION BY customer_id
                                           ORDER BY order_date) AS gap_days
    FROM orders
) t
JOIN customers c ON c.customer_id = t.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name, c.job_title
HAVING COUNT(*) > 1
ORDER BY max_gap_days DESC;
