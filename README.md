# Olist E-Commerce Analytics

Аналитика маркетплейса Olist: GMV, AOV, Retention, воронка заказов, сегментация селлеров по Парето и концентрация по штатам.
**Стек:** PostgreSQL, Apache Superset.

## Содержание
- [Данные](#данные)
- [Метрики и логика](#метрики-и-логика)
- [SQL-запросы](#sql-запросы)
- [Дашборд Superset](#дашборд-superset)
- [Ключевые выводы](#ключевые-выводы)
- [Рекомендации](#рекомендации)
- [Проверка перед запуском](#проверка-перед-запуском)

## Данные

Датасет [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce). Основные таблицы:

| Таблица | Ключевые поля |
|---|---|
| `olist_orders_dataset` | `order_id`, `customer_id`, `order_status`, `order_purchase_timestamp`, `order_approved_at`, `order_delivered_carrier_date`, `order_delivered_customer_date` |
| `olist_order_items_dataset` | `order_id`, `seller_id`, `price`, `freight_value` |
| `olist_order_payments_dataset` | `order_id`, `payment_type`, `payment_installments`, `payment_value` |
| `olist_sellers_dataset` | `seller_id`, `seller_state` |
| `olist_customers_dataset` | `customer_id`, `customer_unique_id`, `customer_state` |

> **Важно:** для Retention используем `customer_unique_id` (реальный покупатель), а не `customer_id` — иначе когорты будут раздуты.

## Метрики и логика

- **GMV** — сумма `payment_value` по оплаченным заказам (`order_status NOT IN ('canceled','unavailable')`).
- **AOV (средний чек)** — `SUM(payment_value) / COUNT(DISTINCT order_id)`.
- **Retention N-го месяца** — доля покупателей когорты, совершивших повторную покупку через N месяцев после первой.
- **Воронка статусов** — распределение заказов по этапам жизненного цикла (`created → approved → shipped → delivered`).
- **Парето селлеров** — разбивка селлеров на 5 групп по выручке и доля каждой группы в общей выручке.
- **Концентрация по штатам** — доля выручки по `customer_state`.

## SQL-запросы

### 1. GMV, заказы, средний чек (динамика по месяцам)

```sql
-- 1.1 Динамика GMV, количества заказов и AOV по месяцам
WITH paid AS (
    SELECT
        o.order_id,
        DATE_TRUNC('month', o.order_purchase_timestamp) AS month,
        p.payment_value
    FROM olist_orders_dataset o
    JOIN olist_order_payments_dataset p USING (order_id)
    WHERE o.order_status NOT IN ('canceled', 'unavailable')
)
SELECT
    month,
    SUM(payment_value)                       AS gmv,
    COUNT(DISTINCT order_id)                 AS orders,
    ROUND(SUM(payment_value)::numeric
          / COUNT(DISTINCT order_id), 2)     AS aov
FROM paid
GROUP BY month
ORDER BY month;


-- 1.2 Средний чек с рассрочкой (installments > 1) и без (installments = 1)
WITH pay_agg AS (
    SELECT
        order_id,
        payment_value,
        MAX(payment_installments) AS installments
    FROM olist_order_payments_dataset
    GROUP BY order_id, payment_value
)
SELECT
    CASE WHEN installments > 1 THEN 'с рассрочкой' ELSE 'без рассрочки' END AS segment,
    COUNT(DISTINCT order_id)                AS orders,
    ROUND(AVG(payment_value)::numeric, 2)   AS avg_check,
    ROUND(SUM(payment_value)::numeric, 2)   AS revenue
FROM pay_agg
GROUP BY segment;


-- 1.3 Сравнение AOV с рассрочкой и без — по месяцам (для графика в Superset)
WITH pay_agg AS (
    SELECT
        p.order_id,
        p.payment_value,
        MAX(p.payment_installments) AS installments,
        o.order_purchase_timestamp
    FROM olist_order_payments_dataset p
    JOIN olist_orders_dataset o USING (order_id)
    GROUP BY p.order_id, p.payment_value, o.order_purchase_timestamp
)
SELECT
    DATE_TRUNC('month', order_purchase_timestamp) AS month,
    ROUND(AVG(payment_value) FILTER (WHERE installments = 1)::numeric, 2) AS aov_no_installment,
    ROUND(AVG(payment_value) FILTER (WHERE installments > 1)::numeric, 2) AS aov_with_installment
FROM pay_agg
GROUP BY month
ORDER BY month;
```

2. Retention (когортный анализ)
```sql
-- 2.1 Retention по когортам (первый месяц покупки) и месяцам удержания
WITH first_purchase AS (
    SELECT
        c.customer_unique_id,
        MIN(DATE_TRUNC('month', o.order_purchase_timestamp)) AS cohort_month
    FROM olist_orders_dataset o
    JOIN olist_customers_dataset c USING (customer_id)
    WHERE o.order_status NOT IN ('canceled', 'unavailable')
    GROUP BY c.customer_unique_id
),
purchases AS (
    SELECT DISTINCT
        c.customer_unique_id,
        DATE_TRUNC('month', o.order_purchase_timestamp) AS purchase_month
    FROM olist_orders_dataset o
    JOIN olist_customers_dataset c USING (customer_id)
    WHERE o.order_status NOT IN ('canceled', 'unavailable')
),
cohort_size AS (
    SELECT cohort_month, COUNT(DISTINCT customer_unique_id) AS cohort_users
    FROM first_purchase
    GROUP BY cohort_month
)
SELECT
    fp.cohort_month,
    EXTRACT(MONTH FROM AGE(p.purchase_month, fp.cohort_month))::int AS month_offset,
    COUNT(DISTINCT p.customer_unique_id)                             AS retained_users,
    cs.cohort_users,
    ROUND(100.0 * COUNT(DISTINCT p.customer_unique_id) / cs.cohort_users, 2) AS retention_pct
FROM first_purchase fp
JOIN purchases p ON p.customer_unique_id = fp.customer_unique_id
JOIN cohort_size cs ON cs.cohort_month = fp.cohort_month
GROUP BY fp.cohort_month, month_offset, cs.cohort_users
ORDER BY fp.cohort_month, month_offset;

-- 2.2 Упрощённо: retention 2-го месяца (для ключевой цифры 3%)
WITH first_purchase AS (
    SELECT
        c.customer_unique_id,
        MIN(DATE_TRUNC('month', o.order_purchase_timestamp)) AS cohort_month
    FROM olist_orders_dataset o
    JOIN olist_customers_dataset c USING (customer_id)
    WHERE o.order_status NOT IN ('canceled', 'unavailable')
    GROUP BY c.customer_unique_id
),
second_month AS (
    SELECT DISTINCT
        c.customer_unique_id,
        DATE_TRUNC('month', o.order_purchase_timestamp) AS purchase_month
    FROM olist_orders_dataset o
    JOIN olist_customers_dataset c USING (customer_id)
    WHERE o.order_status NOT IN ('canceled', 'unavailable')
)
SELECT
    ROUND(100.0
      * COUNT(DISTINCT CASE
            WHEN sm.purchase_month = fp.cohort_month + INTERVAL '1 month'
            THEN fp.customer_unique_id END)
      / COUNT(DISTINCT fp.customer_unique_id), 2) AS retention_month2_pct
FROM first_purchase fp
LEFT JOIN second_month sm USING (customer_unique_id);
```

3. Воронка статусов заказа

```sql
-- 3.1 Распределение заказов по статусам
SELECT
    order_status,
    COUNT(*) AS orders,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) AS share_pct
FROM olist_orders_dataset
GROUP BY order_status
ORDER BY orders DESC;

-- 3.2 Воронка по ключевым этапам (для графика в Superset)
-- В Olist один заказ имеет один статус, поэтому воронку строим по этапам жизненного цикла.
WITH base AS (
    SELECT
        order_id,
        order_status,
        order_approved_at,
        order_delivered_carrier_date,
        order_delivered_customer_date
    FROM olist_orders_dataset
)
SELECT '1. created'          AS stage, COUNT(*) AS orders FROM base
UNION ALL
SELECT '2. approved',   COUNT(*) FROM base WHERE order_approved_at IS NOT NULL
UNION ALL
SELECT '3. shipped',    COUNT(*) FROM base WHERE order_delivered_carrier_date IS NOT NULL
UNION ALL
SELECT '4. delivered',  COUNT(*) FROM base WHERE order_delivered_customer_date IS NOT NULL
UNION ALL
SELECT '5. canceled',   COUNT(*) FROM base WHERE order_status = 'canceled'
ORDER BY stage;
```

4. Сегментация селлеров по Парето
```sql
-- 4.1 5 групп селлеров по выручке (от больших к меньшим) и их доля в общей выручке
WITH seller_revenue AS (
    SELECT
        oi.seller_id,
        SUM(oi.price + oi.freight_value) AS revenue
    FROM olist_order_items_dataset oi
    JOIN olist_orders_dataset o USING (order_id)
    WHERE o.order_status NOT IN ('canceled', 'unavailable')
    GROUP BY oi.seller_id
),
ranked AS (
    SELECT
        seller_id,
        revenue,
        NTILE(5) OVER (ORDER BY revenue DESC) AS quintile
    FROM seller_revenue
)
SELECT
    quintile,
    COUNT(*)                                                   AS sellers,
    ROUND(SUM(revenue)::numeric, 2)                            AS revenue,
    ROUND(100.0 * SUM(revenue) / SUM(SUM(revenue)) OVER (), 2) AS revenue_share_pct
FROM ranked
GROUP BY quintile
ORDER BY quintile;


-- 4.2 Проверка «правила Парето»: доля выручки топ-20% селлеров
WITH seller_revenue AS (
    SELECT
        oi.seller_id,
        SUM(oi.price + oi.freight_value) AS revenue
    FROM olist_order_items_dataset oi
    JOIN olist_orders_dataset o USING (order_id)
    WHERE o.order_status NOT IN ('canceled', 'unavailable')
    GROUP BY oi.seller_id
),
ranked AS (
    SELECT
        seller_id,
        revenue,
        ROW_NUMBER() OVER (ORDER BY revenue DESC) AS rn,
        COUNT(*) OVER ()                           AS total_sellers
    FROM seller_revenue
)
SELECT
    ROUND(100.0 * SUM(CASE WHEN rn <= total_sellers * 0.2 THEN revenue END)
          / SUM(revenue), 2) AS top20_revenue_share_pct
FROM ranked;

```
5. Концентрация по штатам

```sql
-- 5.1 Доля выручки по штатам покупателей
WITH state_revenue AS (
    SELECT
        c.customer_state,
        SUM(p.payment_value) AS revenue
    FROM olist_orders_dataset o
    JOIN olist_order_payments_dataset p USING (order_id)
    JOIN olist_customers_dataset c USING (customer_id)
    WHERE o.order_status NOT IN ('canceled', 'unavailable')
    GROUP BY c.customer_state
)
SELECT
    customer_state,
    ROUND(revenue::numeric, 2)                                          AS revenue,
    ROUND(100.0 * revenue / SUM(revenue) OVER (), 2)                    AS revenue_share_pct,
    ROUND(100.0 * SUM(revenue) OVER (ORDER BY revenue DESC)
          / SUM(revenue) OVER (), 2)                                    AS cumulative_share_pct
FROM state_revenue
ORDER BY revenue DESC;
```

Дашборд Superset
Дашборд «Olist Marketplace Analytics» — 4 блока.

Блок 1. GMV / Orders / AOV

Big Number + Trend Line: GMV за месяц (запрос 1.1).

Big Number + Trend Line: количество заказов за месяц (1.1).

Big Number + Trend Line: средний чек за месяц (1.1).

Bar Chart: средний чек с рассрочкой vs без (1.2 / 1.3).

Блок 2. Retention & Воронка

Line Chart: динамика retention по месяцам удержания (2.1; ось X — month_offset, серии — cohort_month).

Funnel Chart или Bar Chart: воронка статусов заказа (3.2). Карточка с отвалом approved → shipped.

Блок 3. Селлеры (Парето)

Table: 5 квинтилей — количество селлеров, выручка, доля в выручке (4.1).

Big Number: доля топ-20% селлеров в выручке (4.2).

Блок 4. География

Bar Chart / Map: концентрация выручки по штатам с кумулятивной долей (5.1).

Ключевые выводы
AOV с рассрочкой выше на ~30% — рассрочка является драйвером среднего чека.

Retention на 2-й месяц — ~3% — крайне низкое удержание, основной риск бизнеса.

Узкое место воронки — отвал на этапе approved → передан перевозчику.

Топ-20% селлеров дают ~80% выручки — высокая концентрация и риск зависимости от крупных продавцов.

Выручка сконцентрирована в ограниченном числе штатов — географический риск.

Рекомендации
Стимулировать рассрочку в категориях с низким AOV — потенциальный рост чека до +30%.

Программа удержания: триггерные коммуникации на 1–2 месяц после первой покупки (retention 3% — точка роста).

Разбор отвала approved → shipped: SLA по передаче заказа перевозчику, алерты по «зависшим» заказам.

Диверсификация селлерской базы: поддержка селлеров из 2–3 квинтиля, снижение зависимости от топ-20%.

Географическая диверсификация: логистика/промо в штатах за пределами топ-концентрации.
