# 📊 Olist E-commerce: Unit Economics & Marketplace Growth Drivers

**Стек:** PostgreSQL · Apache Superset  
**Роль:** Data Analyst (портфолио-проект, смоделированный под задачи двусторонней e-com платформы)  
**Ссылка на GitHub:** `https://github.com/NikolaySofinskiy/olist-e-commerce-analytics`

---

## 1. Зачем этот проект (и почему он релевантен для Ozon)

Маркетплейс живёт на двух вещах: **средний чек (AOV)** и **повторные покупки (Retention)**. Но за обоими стоят операционные и финансовые рычаги: рассрочки, скорость первой доставки, концентрация селлеров по регионам. Этот проект декомпозирует выручку Olist до уровня, на котором можно принимать решения:

```
Выручка = Количество заказов × средний чек
  │          │                     │
  │          │                     └── платёжные инструменты (installments) → готовность платить больше
  │          └── Retention → качество первого опыта (delivery speed)
  └── География селлеров → концентрация рисков
```

**Бизнес-вопросы, на которые отвечает проект:**

1. Рассрочка — это инструмент роста чека или каннибализации объёма?  
2. Что сильнее предсказывает второй заказ: цена, категория или скорость первой доставки?  
3. Насколько опасна концентрация выручки в топ-20% селлеров и одном регионе?

---

## 2. Данные и допущения (важно для интерпретации)

Использован публичный датасет **Brazilian E-Commerce Public Dataset by Olist** (Kaggle): ~99k заказов, 2016–2018, таблицы `orders`, `order_items`, `order_payments`, `customers`, `sellers`.

**Критическое ограничение, которое я явно фиксирую:** в публичной версии датасета `customer_id` сбрасывается на каждый заказ, поэтому «повторная покупка» определяется через `customer_unique_id`. Это может занижать реальный retention Olist как бизнеса, но **для сравнения когорт внутри датасета этого достаточно**. Все выводы по retention сформулированы как *относительные* (быстрее vs медленнее доставка внутри одной когорты), а не как абсолютный бенчмарк.

---

## 3. Ключевые инсайты и SQL-реализация

### 3.1. Платёжная аналитика: рассрочка и AOV

**Гипотеза:** если рассрочка доступна, покупатель готов взять более дорогой товар, не отказываясь от покупки.

**SQL (PostgreSQL):**

````sql
WITH order_values AS (
    SELECT o.order_id, SUM(oi.price) AS total_price
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY o.order_id
),
payment_stats AS (
    SELECT
        op.payment_type,
        COUNT(DISTINCT op.order_id) AS orders_count,
        ROUND(AVG(ov.total_price), 2) AS avg_aov,
        ROUND(AVG(op.payment_installments), 1) AS avg_installments
    FROM order_payments op
    JOIN order_values ov ON op.order_id = ov.order_id
    GROUP BY op.payment_type
)
SELECT
    payment_type,
    orders_count,
    avg_aov,
    avg_installments,
    ROUND((avg_aov / (SELECT AVG(avg_aov) FROM payment_stats) - 1) * 100, 1)
        AS aov_diff_from_avg_pct
FROM payment_stats
ORDER BY avg_aov DESC;
````
