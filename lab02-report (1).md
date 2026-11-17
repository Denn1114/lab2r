# Лабораторна робота № 2. Створення складних SQL-запитів

## Загальна інформація

- **Здобувач освіти:** Денис Мазурок
- **Група:** ІПЗ-31
- **Обраний рівень складності:** 3
- **СУБД:** PostgreSQL (Supabase)

## Мета роботи

Набути практичних навичок створення складних SQL-запитів для аналізу даних, навчитися з'єднувати таблиці, використовувати агрегатні функції, групування, підзапити, віконні функції, матеріалізовані представлення та рекурсивні CTE, а також аналізувати продуктивність запитів.

## Виконання завдань

### Рівень 1

#### 1. З'єднання таблиць

**Завдання 1.1. INNER JOIN — список товарів із категоріями та постачальниками**

```sql
SELECT
    p.product_name,
    c.category_name,
    s.company_name,
    p.unit_price
FROM products p
INNER JOIN categories c
    ON p.category_id = c.category_id
INNER JOIN suppliers s
    ON p.supplier_id = s.supplier_id
ORDER BY c.category_name, p.product_name;
```

**Пояснення:** запит об'єднує таблиці `products`, `categories` та `suppliers`. Для кожного товару виводиться його назва, категорія, постачальник і ціна. `INNER JOIN` залишає лише записи, для яких знайдено відповідності в усіх з'єднаних таблицях.

**Результат:** запит призначений для отримання переліку товарів із даними про категорії та постачальників.

**Скріншот:** `screenshots/01_inner_join.png`

#### Завдання 1.2. LEFT JOIN — клієнти з кількістю замовлень

```sql
SELECT
    c.contact_name,
    c.customer_type,
    r.region_name,
    COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
LEFT JOIN regions r
    ON c.region_id = r.region_id
GROUP BY
    c.customer_id,
    c.contact_name,
    c.customer_type,
    r.region_name
ORDER BY order_count DESC;
```

**Пояснення:** запит показує клієнтів, їхній тип, регіон і кількість замовлень. На відміну від `INNER JOIN`, `LEFT JOIN` зберігає всіх клієнтів, навіть якщо вони не мають замовлень.

**Результат:** для клієнтів без замовлень значення `order_count` дорівнює 0, оскільки використовується `COUNT(o.order_id)`.

**Скріншот:** `screenshots/02_left_join.png`

#### Завдання 1.3. Множинне з'єднання — детальна інформація про замовлення

```sql
SELECT
    o.order_id,
    o.order_date,
    c.contact_name,
    p.product_name,
    cat.category_name,
    s.company_name AS supplier,
    oi.quantity,
    oi.unit_price,
    oi.discount,
    oi.quantity * oi.unit_price *
        (1 - COALESCE(oi.discount, 0)) AS total
FROM orders o
JOIN customers c
    ON o.customer_id = c.customer_id
JOIN order_items oi
    ON o.order_id = oi.order_id
JOIN products p
    ON oi.product_id = p.product_id
JOIN categories cat
    ON p.category_id = cat.category_id
JOIN suppliers s
    ON p.supplier_id = s.supplier_id
ORDER BY o.order_date DESC;
```

**Аналіз складності:** запит об'єднує шість таблиць. Спочатку замовлення з'єднуються з клієнтами та позиціями замовлення, після чого додається інформація про товари, категорії та постачальників. Загальна вартість позиції обчислюється з урахуванням знижки.

**Результат:** формується деталізований список замовлень із клієнтами, товарами, категоріями, постачальниками та сумою позиції.

**Скріншот:** `screenshots/03_multiple_join.png`

#### 2. Агрегатні функції

**Завдання 2.1. Статистика товарів за категоріями**

```sql
SELECT
    c.category_name,
    COUNT(p.product_id) AS product_count,
    AVG(p.unit_price) AS avg_price,
    MIN(p.unit_price) AS min_price,
    MAX(p.unit_price) AS max_price
FROM categories c
LEFT JOIN products p
    ON c.category_id = p.category_id
GROUP BY c.category_id, c.category_name
ORDER BY product_count DESC;
```

**Пояснення:** `COUNT` визначає кількість товарів, `AVG` — середню ціну, `MIN` — мінімальну, `MAX` — максимальну. `LEFT JOIN` дозволяє показати також категорії без товарів.

**Результат:** статистика кількості та цін товарів у кожній категорії.

**Скріншот:** `screenshots/04_category_stats.png`

**Завдання 2.2. Продажі за регіонами з використанням HAVING**

```sql
SELECT
    r.region_name,
    COUNT(DISTINCT o.order_id) AS orders_count,
    SUM(
        oi.quantity * oi.unit_price *
        (1 - COALESCE(oi.discount, 0))
    ) AS total_sales,
    AVG(
        oi.quantity * oi.unit_price *
        (1 - COALESCE(oi.discount, 0))
    ) AS avg_sale
FROM regions r
JOIN customers c
    ON r.region_id = c.region_id
JOIN orders o
    ON c.customer_id = o.customer_id
JOIN order_items oi
    ON o.order_id = oi.order_id
GROUP BY r.region_id, r.region_name
HAVING SUM(
    oi.quantity * oi.unit_price *
    (1 - COALESCE(oi.discount, 0))
) > 0
ORDER BY total_sales DESC;
```

**Пояснення:** запит обчислює кількість замовлень, загальну суму та середню суму продажів у кожному регіоні. `HAVING` залишає лише групи, у яких загальна сума продажів більша за нуль.

**Результат:** регіони впорядковані за сумою продажів у спадному порядку.

**Скріншот:** `screenshots/05_region_sales.png`

**Завдання 2.3. Постачальники з кількістю товарів більше 2**

```sql
SELECT
    s.supplier_id,
    s.company_name,
    COUNT(p.product_id) AS products_count,
    AVG(p.unit_price) AS avg_price,
    MIN(p.unit_price) AS min_price,
    MAX(p.unit_price) AS max_price
FROM suppliers s
JOIN products p
    ON s.supplier_id = p.supplier_id
GROUP BY s.supplier_id, s.company_name
HAVING COUNT(p.product_id) > 2
ORDER BY products_count DESC;
```

**Пояснення:** запит групує товари за постачальниками та залишає постачальників, у яких більше двох товарів. Додатково обчислюються середня, мінімальна та максимальна ціни.

**Результат:** перелік постачальників, які мають більше двох товарів.

**Скріншот:** `screenshots/06_supplier_stats.png`

#### 3. Базові підзапити

**Завдання 3.1. Товари з ціною вище середньої по категорії**

```sql
SELECT
    p.product_name,
    p.unit_price,
    c.category_name
FROM products p
INNER JOIN categories c
    ON p.category_id = c.category_id
WHERE p.unit_price > (
    SELECT AVG(p2.unit_price)
    FROM products p2
    WHERE p2.category_id = p.category_id
)
ORDER BY c.category_name, p.unit_price DESC;
```

**Пояснення:** корельований підзапит обчислює середню ціну товарів тієї самої категорії. Зовнішній запит повертає товари, ціна яких перевищує це середнє значення.

**Результат:** товари дорожчі за середню ціну своєї категорії.

**Скріншот:** `screenshots/07_correlated_subquery.png`

**Завдання 3.2. Клієнти із замовленнями у 2024 році**

```sql
SELECT
    c.customer_id,
    c.contact_name,
    c.customer_type
FROM customers c
WHERE c.customer_id IN (
    SELECT o.customer_id
    FROM orders o
    WHERE o.order_date >= '2024-01-01'
      AND o.order_date < '2025-01-01'
)
ORDER BY c.contact_name;
```

**Пояснення:** підзапит повертає ідентифікатори клієнтів, які мали замовлення у 2024 році. Оператор `IN` відбирає відповідних клієнтів із таблиці `customers`.

**Результат:** список клієнтів, які здійснювали замовлення у 2024 році.

**Скріншот:** `screenshots/08_subquery_in.png`

**Завдання 3.3. Товари із загальною кількістю продажів**

```sql
SELECT
    p.product_id,
    p.product_name,
    p.unit_price,
    (
        SELECT COALESCE(SUM(oi.quantity), 0)
        FROM order_items oi
        WHERE oi.product_id = p.product_id
    ) AS total_sold
FROM products p
ORDER BY total_sold DESC;
```

**Пояснення:** підзапит у списку `SELECT` обчислює загальну кількість проданих одиниць для кожного товару. `COALESCE` замінює `NULL` на 0 для товарів без продажів.

**Результат:** товари впорядковані за загальною кількістю продажів.

**Скріншот:** `screenshots/09_subquery_select.png`

**Додаткове завдання. Використання EXISTS**

```sql
SELECT
    c.customer_id,
    c.contact_name,
    c.customer_type
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
)
ORDER BY c.contact_name;
```

**Пояснення:** `EXISTS` перевіряє наявність хоча б одного замовлення для кожного клієнта.

**Результат:** список клієнтів, які мають хоча б одне замовлення.

**Скріншот:** `screenshots/10_subquery_exists.png`

### Рівень 2

#### 4. Складні з'єднання

**Завдання 4.1. RIGHT JOIN — аналіз категорій та товарів**

```sql
SELECT
    c.category_name,
    COUNT(p.product_id) AS products_count,
    COALESCE(AVG(p.unit_price), 0) AS avg_price
FROM products p
RIGHT JOIN categories c
    ON p.category_id = c.category_id
GROUP BY c.category_id, c.category_name
ORDER BY products_count DESC;
```

**Пояснення:** запит показує всі категорії, навіть якщо в них немає товарів. Для категорій без товарів кількість дорівнює 0.

**Результат:** категорії з кількістю товарів і середньою ціною.

**Скріншот:** `screenshots/11_right_join.png`

**Завдання 4.2. Self-join — співробітники та керівники**

```sql
SELECT
    e1.first_name || ' ' || e1.last_name AS employee,
    e1.title AS employee_title,
    e2.first_name || ' ' || e2.last_name AS manager,
    e2.title AS manager_title
FROM employees e1
LEFT JOIN employees e2
    ON e1.reports_to = e2.employee_id
ORDER BY e2.last_name, e1.last_name;
```

**Пояснення:** таблиця `employees` з'єднується сама з собою. Перший псевдонім представляє працівника, другий — його керівника. `LEFT JOIN` зберігає також працівників без керівника.

**Результат:** список співробітників із відповідними керівниками.

**Скріншот:** `screenshots/12_self_join.png`

#### 5. Віконні функції

**Завдання 5.1. Ранжування товарів за ціною в категоріях**

```sql
SELECT
    p.product_name,
    c.category_name,
    p.unit_price,
    RANK() OVER (
        PARTITION BY c.category_name
        ORDER BY p.unit_price DESC
    ) AS price_rank,
    DENSE_RANK() OVER (
        PARTITION BY c.category_name
        ORDER BY p.unit_price DESC
    ) AS price_dense_rank,
    ROW_NUMBER() OVER (
        PARTITION BY c.category_name
        ORDER BY p.unit_price DESC
    ) AS row_num
FROM products p
JOIN categories c
    ON p.category_id = c.category_id
ORDER BY c.category_name, p.unit_price DESC;
```

**Пояснення:** `PARTITION BY` розділяє товари за категоріями. `RANK()` присвоює ранг із пропусками після однакових значень, `DENSE_RANK()` — без пропусків, а `ROW_NUMBER()` — унікальний номер кожному рядку.

**Результат:** товари ранжуються за ціною окремо в кожній категорії.

**Скріншот:** `screenshots/13_window_rank.png`

**Завдання 5.2. Порівняння замовлень із попередніми та наступними**

```sql
SELECT
    o.customer_id,
    o.order_id,
    o.order_date,
    LAG(o.order_date) OVER (
        PARTITION BY o.customer_id
        ORDER BY o.order_date
    ) AS previous_order_date,
    LEAD(o.order_date) OVER (
        PARTITION BY o.customer_id
        ORDER BY o.order_date
    ) AS next_order_date
FROM orders o
ORDER BY o.customer_id, o.order_date;
```

**Пояснення:** `LAG()` повертає дату попереднього замовлення клієнта, а `LEAD()` — дату наступного. `PARTITION BY` виконує порівняння окремо для кожного клієнта.

**Результат:** для кожного замовлення відображаються попередня та наступна дати замовлень цього клієнта, якщо вони існують.

**Скріншот:** `screenshots/14_lag_lead.png`

### Рівень 3

#### 6. Матеріалізоване представлення та рекурсивний запит

**Завдання 6.1. Матеріалізоване представлення для аналізу продажів**

```sql
CREATE MATERIALIZED VIEW mv_monthly_sales AS
SELECT
    EXTRACT(YEAR FROM o.order_date) AS year,
    EXTRACT(MONTH FROM o.order_date) AS month,
    c.category_name,
    r.region_name,
    SUM(
        oi.quantity * oi.unit_price * (1 - oi.discount)
    ) AS total_revenue,
    COUNT(DISTINCT o.order_id) AS orders_count,
    AVG(
        oi.quantity * oi.unit_price * (1 - oi.discount)
    ) AS avg_order_value
FROM orders o
JOIN order_items oi
    ON o.order_id = oi.order_id
JOIN products p
    ON oi.product_id = p.product_id
JOIN categories c
    ON p.category_id = c.category_id
JOIN customers cu
    ON o.customer_id = cu.customer_id
LEFT JOIN regions r
    ON cu.region_id = r.region_id
WHERE o.order_status = 'delivered'
GROUP BY year, month, c.category_name, r.region_name;

CREATE INDEX idx_mv_monthly_sales_date
ON mv_monthly_sales(year, month);
```

**Перевірка представлення:**

```sql
SELECT *
FROM mv_monthly_sales;
```

**Пояснення:** матеріалізоване представлення зберігає результат складного аналітичного запиту. Це дає змогу повторно отримувати агреговані дані без повторного виконання всіх з'єднань та обчислень. Для оновлення даних представлення використовується `REFRESH MATERIALIZED VIEW mv_monthly_sales;`.

**Результат:** агреговані показники продажів доставлених замовлень за роком, місяцем, категорією та регіоном.

**Скріншот:** `screenshots/15_materialized_view.png`

> Якщо представлення вже створене, перед повторним запуском команди створення виконайте `DROP MATERIALIZED VIEW IF EXISTS mv_monthly_sales CASCADE;`.

**Завдання 6.2. Рекурсивний запит для ієрархії співробітників**

```sql
WITH RECURSIVE employee_hierarchy AS (
    SELECT
        employee_id,
        first_name,
        last_name,
        title,
        reports_to,
        0 AS level,
        CAST(last_name || ' ' || first_name AS VARCHAR(1000))
            AS hierarchy_path
    FROM employees
    WHERE reports_to IS NULL

    UNION ALL

    SELECT
        e.employee_id,
        e.first_name,
        e.last_name,
        e.title,
        e.reports_to,
        eh.level + 1,
        CAST(
            eh.hierarchy_path || ' -> ' ||
            e.last_name || ' ' || e.first_name
            AS VARCHAR(1000)
        )
    FROM employees e
    JOIN employee_hierarchy eh
        ON e.reports_to = eh.employee_id
)
SELECT *
FROM employee_hierarchy
ORDER BY hierarchy_path;
```

**Пояснення:** рекурсивний CTE спочатку знаходить співробітників без керівника, а потім послідовно додає їхніх підлеглих. Поле `level` показує рівень ієрархії, а `hierarchy_path` — шлях від керівника до працівника.

**Результат:** ієрархічний список співробітників організації.

**Скріншот:** `screenshots/16_recursive_cte.png`

### Аналіз продуктивності

Для аналізу продуктивності використано `EXPLAIN ANALYZE`, який показує план виконання запиту та фактичний час виконання.

**Запит 1. Аналіз товарів і категорій**

```sql
EXPLAIN ANALYZE
SELECT
    p.product_name,
    c.category_name,
    p.unit_price
FROM products p
JOIN categories c
    ON p.category_id = c.category_id
WHERE p.unit_price > 1000;
```

**Скріншот:** `screenshots/17_explain_products.png`

**Запит 2. Аналіз замовлень клієнтів**

```sql
EXPLAIN ANALYZE
SELECT
    c.customer_id,
    c.contact_name,
    COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.contact_name
ORDER BY order_count DESC;
```

**Скріншот:** `screenshots/18_explain_customers.png`

**Запит 3. Створення індексів для оптимізації**

```sql
CREATE INDEX IF NOT EXISTS idx_orders_customer_date
ON orders(customer_id, order_date);

CREATE INDEX IF NOT EXISTS idx_products_category_price
ON products(category_id, unit_price);

CREATE INDEX IF NOT EXISTS idx_customers_type_region
ON customers(customer_type, region_id);
```

**Пояснення:** створено складені індекси для стовпців, які використовуються під час з'єднання, фільтрації та аналізу даних. Індекс `idx_orders_customer_date` орієнтований на пошук замовлень за клієнтом і датою; `idx_products_category_price` — на відбір товарів за категорією та ціною; `idx_customers_type_region` — на пошук клієнтів за типом і регіоном.

**Перевірка створених індексів:**

```sql
SELECT
    tablename,
    indexname
FROM pg_indexes
WHERE schemaname = 'public'
  AND indexname IN (
      'idx_orders_customer_date',
      'idx_products_category_price',
      'idx_customers_type_region'
  )
ORDER BY tablename;
```

**Скріншот:** `screenshots/19_indexes.png`

**Визначення найповільнішого запиту:** порівняти значення `Execution Time` у результатах перших двох команд `EXPLAIN ANALYZE`. Фактичний найповільніший запит і його час потрібно вписати за результатами запуску в Supabase:

- Запит 1 — Execution Time: **[вписати значення з Supabase]**
- Запит 2 — Execution Time: **[вписати значення з Supabase]**
- Найповільніший запит: **[вказати після порівняння]**

**Запропоновані способи оптимізації:**

1. Створювати індекси для стовпців, які часто використовуються у `JOIN`, `WHERE` та сортуванні.
2. Відбирати потрібні рядки якомога раніше, щоб зменшити обсяг даних для подальшої обробки.
3. Не використовувати `SELECT *`, якщо потрібні лише окремі стовпці.
4. Перевіряти план виконання через `EXPLAIN ANALYZE` до та після оптимізації.
5. Для повторюваної складної аналітики використовувати матеріалізовані представлення.

### Оцінка відповідності завданням

| Завдання | Що реалізовано | Оцінка відповідності |
|---|---|---|
| INNER JOIN | Товари, категорії та постачальники | Виконано |
| LEFT JOIN | Клієнти, замовлення та регіони | Виконано |
| Множинне з'єднання | З'єднання шести таблиць | Виконано |
| Агрегатні функції | COUNT, SUM, AVG, MIN, MAX, GROUP BY, HAVING | Виконано |
| Підзапити | Корельований підзапит, IN, SELECT, EXISTS | Виконано |
| RIGHT JOIN | Аналіз категорій і товарів | Виконано |
| Self-join | Співробітники та керівники | Виконано |
| Віконні функції | ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD, PARTITION BY | Виконано |
| Матеріалізоване представлення | mv_monthly_sales та індекс | Виконано |
| Рекурсивний CTE | Ієрархія співробітників | Виконано |
| Аналіз продуктивності | Два EXPLAIN ANALYZE та створення індексів | Виконано; часи потрібно внести за фактичним результатом |

## Порівняльний аналіз

**Завдання:** знайти топ-5 найдорожчих товарів у кожній категорії.

**Підхід 1. Віконна функція**

```sql
SELECT
    product_name,
    category_id,
    unit_price
FROM (
    SELECT
        p.product_name,
        p.category_id,
        p.unit_price,
        ROW_NUMBER() OVER (
            PARTITION BY p.category_id
            ORDER BY p.unit_price DESC
        ) AS rn
    FROM products p
) ranked_products
WHERE rn <= 5
ORDER BY category_id, unit_price DESC;
```

**Підхід 2. Корельований підзапит**

```sql
SELECT
    p.product_name,
    p.category_id,
    p.unit_price
FROM products p
WHERE (
    SELECT COUNT(*)
    FROM products p2
    WHERE p2.category_id = p.category_id
      AND p2.unit_price > p.unit_price
) < 5
ORDER BY p.category_id, p.unit_price DESC;
```

**Порівняння:** віконна функція нумерує товари в межах кожної категорії та відбирає перші п'ять. Корельований підзапит для кожного товару рахує кількість дорожчих товарів у його категорії. Для великих наборів даних віконний підхід зазвичай зручніший для такого ранжування, але фактичну швидкодію потрібно підтверджувати вимірюванням.

- Час виконання віконного підходу: **[вписати після запуску EXPLAIN ANALYZE]**
- Час виконання підзапиту: **[вписати після запуску EXPLAIN ANALYZE]**

## Висновки

Під час лабораторної роботи було опрацьовано складні SQL-запити в PostgreSQL: з'єднання таблиць різними типами `JOIN`, агрегатні функції та групування, підзапити, віконні функції, матеріалізоване представлення й рекурсивний CTE. Також виконано аналіз планів запитів за допомогою `EXPLAIN ANALYZE` та створено індекси для оптимізації доступу до даних.

Отримані навички дають змогу формувати аналітичні вибірки з кількох таблиць, обчислювати статистичні показники, аналізувати ієрархічні дані та оцінювати продуктивність SQL-запитів.

**Самооцінка:** [вказати оцінку від 3 до 5 після перевірки результатів].

**Обґрунтування:** основні завдання трьох рівнів реалізовано SQL-запитами. Остаточна оцінка продуктивності потребує внесення фактичних значень `Execution Time` із Supabase та додавання скріншотів виконання.
