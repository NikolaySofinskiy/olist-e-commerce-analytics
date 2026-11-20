# 🛒 Olist E-Commerce Analytics

**Краткое описание:**  
End-to-end аналитика маркетплейса Olist. Проект демонстрирует полный цикл работы с данными: от загрузки сырых CSV-файлов в PostgreSQL (PGLite) до построения интерактивного BI-дашборда в Apache Superset (Preset).

**Стек:** SQL (PostgreSQL / PGLite), Apache Superset (Preset)  
**Данные:** [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

---

## 🎯 Задача проекта

Провести комплексный анализ данных бразильского маркетплейса для выявления точек роста, оценки логистики и понимания поведения клиентов.

## 🛠 Что было сделано

1. **Подготовка данных:** Загружено 9 датасетов в SQL-песочницу SQLiteOnline (на базе PGLite).
2. **Расчет бизнес-метрик:** Написаны SQL-запросы для расчета выручки (Revenue), количества заказов, среднего чека (AOV), а также динамики по месяцам.
3. **Продуктовая аналитика:** Реализован когортный анализ (Retention) для оценки удержания клиентов и частоты повторных покупок.
4. **Визуализация:** В облачном Superset (Preset) собран интерактивный дашборд с использованием графиков Line, Bar, Pie и Big Number.

## 📈 Ключевые инсайты

* **Сезонность:** Выявлен аномальный пик выручки в ноябре 2017 года, что связано с распродажами в «Черную пятницу». В 2018 году наблюдается стабильный рост заказов.
* **География:** Основная доля продаж сконцентрирована в штате Сан-Паулу (SP). Однако в северных регионах наблюдается высокое время доставки, что может негативно влиять на клиентский опыт.
* **Способы оплаты:** Более 73% клиентов предпочитают оплату кредитными картами, что подтверждает важность надежной интеграции с банковскими шлюзами.
* **Retention:** Основная масса клиентов совершает повторные покупки в первые 1-2 месяца после первого заказа, после чего наблюдается естественный отток (характерно для e-commerce).

---

## 💻 SQL Запросы

Все запросы валидированы в PGLite и адаптированы для работы в Preset.

**1. Динамика выручки и заказов (Line Chart)**

```sql
SELECT 
    TO_CHAR(o.order_purchase_timestamp::TIMESTAMP, 'YYYY-MM') AS month,
    COUNT(DISTINCT o.order_id) AS orders_count,
    ROUND(SUM(oi.price + oi.freight_value)::NUMERIC, 2) AS revenue
FROM orders AS o
LEFT JOIN order_items AS oi ON o.order_id = oi.order_id
WHERE o.order_status = 'delivered'
GROUP BY 1
ORDER BY 1;
```

**2. Топ-10 категорий товаров (Bar Chart)**

```sql
SELECT
    COALESCE(t.product_category_name_english, p.product_category_name) AS category,
    COUNT(*) AS items_sold,
    ROUND(SUM(oi.price)::NUMERIC, 2) AS revenue
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
LEFT JOIN product_category_name_translation t
    ON p.product_category_name = t.product_category_name
GROUP BY 1
ORDER BY revenue DESC
LIMIT 10;
```

**3. Продажи по штатам (Bar/Map Chart)**

```sql
SELECT
    c.customer_state AS state,
    COUNT(DISTINCT o.order_id) AS orders_count,
    ROUND(SUM(oi.price + oi.freight_value)::NUMERIC, 2) AS revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_status = 'delivered'
GROUP BY 1
ORDER BY revenue DESC;
```

**4. Способы оплаты (Pie Chart)**

```sql
SELECT
    payment_type,
    COUNT(*) AS payments_count,
    ROUND(SUM(payment_value)::NUMERIC, 2) AS total_value
FROM order_payments
GROUP BY 1
ORDER BY total_value DESC;
```

**5. Общие KPI (Big Number)**

```sql
SELECT
    COUNT(DISTINCT o.order_id) AS total_orders,
    ROUND(SUM(oi.price + oi.freight_value)::NUMERIC, 2) AS total_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_status = 'delivered';
```

**6. Когортный анализ Retention (Pivot Table)**

```sql
WITH first_orders AS (
    SELECT 
        c.customer_unique_id,
        MIN(o.order_purchase_timestamp::TIMESTAMP) AS first_order_date
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    WHERE o.order_status = 'delivered'
    GROUP BY c.customer_unique_id
),
cohorts AS (
    SELECT 
        customer_unique_id,
        TO_CHAR(first_order_date, 'YYYY-MM') AS cohort_month
    FROM first_orders
),
orders_activity AS (
    SELECT 
        c.customer_unique_id,
        TO_CHAR(o.order_purchase_timestamp::TIMESTAMP, 'YYYY-MM') AS order_month
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    WHERE o.order_status = 'delivered'
)
SELECT 
    co.cohort_month,
    oa.order_month,
    COUNT(DISTINCT oa.customer_unique_id) AS users
FROM cohorts co
JOIN orders_activity oa ON co.customer_unique_id = oa.customer_unique_id
GROUP BY 1, 2
ORDER BY 1, 2;
```
---

## 📂 Структура репозитория

```
olist-e-commerce-analytics/
│
├── README.md               # Главный файл с описанием проекта
├── /sql                    # Папка с SQL-запросами
│   ├── 01_revenue_trend.sql
│   ├── 02_top_categories.sql
│   ├── 03_sales_by_state.sql
│   ├── 04_payment_types.sql
│   ├── 05_kpi_totals.sql
│   └── 06_retention_cohorts.sql
├── /dashboards             # Скриншоты дашбордов из Preset
│   ├── overview.png
│   ├── geography.png
│   └── retention.png
└── /data                   # Описание источника данных
    └── dataset_description.md
```
---

## 🚀 Как запустить проект

1. Загрузите датасет с Kaggle.
2. Импортируйте CSV-файлы в SQLiteOnline (PGLite) или любой PostgreSQL-клиент.
3. Выполните SQL-запросы из папки `/sql`.
4. Для визуализации загрузите данные в Preset (через File Uploads или подключив внешнюю БД) и создайте графики на основе запросов.
