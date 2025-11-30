# Домашнее задание №3  
Новиков Иван

 

---

## 1. Развернуты таблицы для выполнения ДЗ. 

Напоминаю, у меня PostgreSQL и pgAdmin были развернуты в Docker под WSL2.
Данные были скорректированы / очищены.


Сначала я как "честный самаритянин" начала заново разворачивать все таблицы:

https://disk.yandex.ru/i/8Le76BR_hm74oQ ,

https://disk.yandex.ru/i/L7hl3Bel4Cmo-A ,

https://disk.yandex.ru/i/nvvuuAbrROdzCA , 

https://disk.yandex.ru/i/Rn6mw1K03eOE1A ,

 о когда я начал импортировать данные, я увидел, что *идут теже ошибки, что в и прошлый раз*. Далее я полез сразнивать поля таблиц (все 1к1 вроде) и данные в таблицых
(тоже совпали, при беглом анализе).
Соотв. далее я вернулся к таблицам от ДЗ №2, чтобы работать с ними =)

## 2. Задания по запросам ДЗ №3

**Запрос 1.**
*Вывести распределение (количество) клиентов по сферам деятельности, отсортировав результат по убыванию количества*

```sql

SELECT
    job_industry_category,
    COUNT(*) AS customer_count
	
FROM customer

WHERE job_industry_category IS NOT NULL

GROUP BY job_industry_category
ORDER BY customer_count  DESC;


```
**Результат:**

| job_industry_category | customer_count |
| --------------------- | -------------- |
| Manufacturing         | 799            |
| Financial Services    | 774            |
| n/a                   | 656            |
| Health                | 602            |
| Retail                | 358            |
| Property              | 267            |
| IT                    | 223            |
| Entertainment         | 136            |
| Argiculture           | 113            |
| Telecommunications    | 72             |


Скриншоты: https://disk.yandex.ru/i/Pa1pYNg0r3QYhA


.



**Запрос 2.**
*Найти общую сумму дохода (list_price*quantity) по всем подтвержденным заказам за каждый месяц по сферам деятельности клиентов. Отсортировать по году, месяцу и сфере деятельности*

```sql


SELECT
    EXTRACT(YEAR  FROM o.order_date) AS year,
    EXTRACT(MONTH FROM o.order_date) AS month,
    c.job_industry_category,
	
    SUM(oi.item_list_price_at_sale * oi.quantity) AS total_revenue
	
FROM orders o
JOIN customer c ON c.customer_id = o.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
-- JOIN product p ON p.product_id = oi.product_id

WHERE o.order_status = 'Approved'

GROUP BY
    year, month,
    c.job_industry_category
	
ORDER BY
    year, month,
    c.job_industry_category;
    

```
**Результат:**
см. скриншот, итого 120 строк (10 категорий на каждый месяц)
Скриншоты: https://disk.yandex.ru/i/3Z1rT_hbPGsN8w 


.




**Запрос 3.**
*Вывести количество уникальных онлайн-заказов для всех брендов в рамках подтвержденных заказов клиентов из сферы IT. Включить бренды, у которых нет онлайн-заказов от IT-клиентов, с количеством 0*

```sql

WITH it_orders AS (
    SELECT DISTINCT o.order_id, p.brand
    FROM orders o
	
    JOIN customer c ON c.customer_id = o.customer_id
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN product p ON p.product_id = oi.product_id
	
    WHERE o.order_status = 'Approved'
		AND o.online_order = true
		AND c.job_industry_category = 'IT'
)

SELECT
    p.brand,
    COUNT(DISTINCT it_orders.order_id) AS online_order_count
	
FROM product p
LEFT JOIN it_orders ON it_orders.brand = p.brand -- для брендов без заказов

GROUP BY p.brand
ORDER BY p.brand;


```
**Результат:**

| brand          | online_order_count |
| -------------- | ------------------ |
| Giant Bicycles | 102                |
| Norco Bicycles | 59                 |
| OHM Cycles     | 69                 |
| Solex          | 101                |
| Trek Bicycles  | 78                 |
| WeareA2B       | 87                 |


Скриншоты: https://disk.yandex.ru/i/M0GbNqor4jAClg


.




**Запрос 4.1 (вариант с GROUP BY)** 
*Найти по всем клиентам: сумму всех заказов (общего дохода), максимум, минимум и количество заказов, а также среднюю сумму заказа по каждому клиенту. Отсортировать результат по убыванию суммы всех заказов и количества заказов. Выполнить двумя способами: используя только GROUP BY и используя только оконные функции. Сравнить результат*

```sql

-- 4.1 GROUP BY

-- сначала считаю сумму по каждому заказу
WITH order_totals AS (
    SELECT o.order_id, o.customer_id,
        SUM(oi.quantity * p.list_price) AS order_amount
    
	FROM orders o
	
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN product p ON p.product_id = oi.product_id
	
    GROUP BY o.order_id, o.customer_id
)
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
	
    COALESCE(SUM(order_amount), 0) AS total_amount,
    COALESCE(MAX(order_amount), 0) AS max_order_amount,
    COALESCE(MIN(order_amount), 0) AS min_order_amount,
    COALESCE(COUNT(order_id),  0) AS order_count,
    COALESCE(AVG(order_amount), 0) AS avg_order_amount
	
FROM customer c

LEFT JOIN order_totals ot
       ON ot.customer_id = c.customer_id -- клиенты без заказов тоже попадают
	   
GROUP BY
    c.customer_id,
    c.first_name,
    c.last_name
ORDER BY
    total_amount DESC,
    order_count DESC;
	


```
**Результат:**
см. скриншоты

Скриншоты: 
https://disk.yandex.ru/i/1HY-_cf_mxFGkQ , 

только рез-ы
https://disk.yandex.ru/i/5ipao-Cr3MY9Ug


.**Запрос 4.2 (только с оконными функциями)** 
*Найти по всем клиентам: сумму всех заказов (общего дохода), максимум, минимум и количество заказов, а также среднюю сумму заказа по каждому клиенту. Отсортировать результат по убыванию суммы всех заказов и количества заказов. Выполнить двумя способами: используя только GROUP BY и используя только оконные функции. Сравнить результат*


```sql

-- 4.2 оконные функции


-- сначала считаю сумму по каждому заказу
WITH order_totals AS (
    SELECT o.order_id, o.customer_id,
        SUM(oi.quantity * p.list_price) AS order_amount
    
	FROM orders o
	
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN product p ON p.product_id = oi.product_id
	
    GROUP BY o.order_id, o.customer_id
),
customer_metrics AS (
    SELECT
        c.customer_id,
        c.first_name,
        c.last_name,
		
        -- окно со всеми строками одного клиента
        COALESCE(SUM(ot.order_amount) OVER w, 0) AS total_amount,
        COALESCE(MAX(ot.order_amount) OVER w, 0) AS max_order_amount,
        COALESCE(MIN(ot.order_amount) OVER w, 0) AS min_order_amount,
        COALESCE(COUNT(ot.order_id) OVER w, 0) AS order_count,
        COALESCE( AVG(ot.order_amount) OVER w, 0) AS avg_order_amount,
		
        ROW_NUMBER() OVER w AS rn -- по одной строке на клиента
		
    FROM customer c
    LEFT JOIN order_totals ot ON ot.customer_id = c.customer_id
	
    WINDOW w AS (PARTITION BY c.customer_id)
)

SELECT
    customer_id,
    first_name,
    last_name,
    total_amount,
    max_order_amount,
    min_order_amount,
    order_count,
    avg_order_amount

FROM customer_metrics

WHERE rn = 1 -- один ряд на клиента

ORDER BY total_amount DESC, order_count DESC;
	
	


```
**Результат:** работает :)
см. скриншоты
https://disk.yandex.ru/i/FOF2a5ySamLyrQ


**Запрос 5.**
*Найти имена и фамилии клиентов с топ-3 минимальной и топ-3 максимальной суммой транзакций за весь период (учесть клиентов, у которых нет заказов)*

```sql

-- 5. топ3 по минимуму и максмуму суммы заказов

WITH order_totals AS (
    SELECT o.order_id, o.customer_id,
        SUM(oi.quantity * p.list_price) AS order_amount
		
    FROM orders o
	
    JOIN order_items oi ON oi.order_id  = o.order_id
    JOIN product p ON p.product_id = oi.product_id
	
    GROUP BY o.order_id, o.customer_id
),
customer_sum AS (
    SELECT
        c.customer_id, c.first_name, c.last_name,
        COALESCE(SUM(ot.order_amount), 0) AS total_amount -- 0, если заказов нет
    
	FROM customer c
	
    LEFT JOIN order_totals ot  ON ot.customer_id = c.customer_id
		   
    GROUP BY
        c.customer_id,
        c.first_name,
        c.last_name
),
ranked AS (
    SELECT
        customer_id,
        first_name,
        last_name,
        total_amount,
		
        DENSE_RANK() OVER (ORDER BY total_amount ASC)  AS rnk_min,
        DENSE_RANK() OVER (ORDER BY total_amount DESC) AS rnk_max
		
    FROM customer_sum
)
SELECT customer_id, first_name, last_name, total_amount

FROM ranked WHERE rnk_min <= 3  OR rnk_max <= 3

ORDER BY total_amount ASC, customer_id;


```
**Результат, Скриншоты:** 
запрос:  https://disk.yandex.ru/i/TY11N8DhlLgrDg
результаты: https://disk.yandex.ru/i/MM5vzPqdrC5bYQ 
.




**Запрос 6.**
*Вывести только вторые транзакции клиентов (если они есть). Решить с помощью оконных функций. Если у клиента меньше двух транзакций, он не должен попасть в результат*

```sql

WITH ordered_orders AS (

    SELECT o.order_id, o.customer_id, o.order_date,
        ROW_NUMBER() OVER (
            PARTITION BY o.customer_id
            ORDER BY o.order_date, o.order_id -- нумерация зак=в клиента
        ) AS rn
		
    FROM orders o
)

SELECT c.customer_id, c.first_name, c.last_name, o.order_id, o.order_date
FROM ordered_orders o

JOIN customer c ON c.customer_id = o.customer_id

WHERE o.rn = 2 -- только 2 транзакция

ORDER BY c.customer_id;


```

**Результат, Скриншоты:** 
запрос: https://disk.yandex.ru/i/DLsj8jq6jcgLWA
результаты: https://disk.yandex.ru/i/3RVoU-exFcakCg
.




.




**Запрос 7.**
*Вывести имена, фамилии и профессии клиентов, а также длительность максимального интервала (в днях) между двумя последовательными заказами. Исключить клиентов, у которых только один или меньше заказов*

```sql

-- 7 максимальный интервал 

WITH ordered_orders AS (
    SELECT o.customer_id, o.order_id, o.order_date,
        LAG(o.order_date) OVER (
            PARTITION BY o.customer_id
            ORDER BY o.order_date, o.order_id
        ) AS prev_order_date
    FROM orders o
),

intervals AS (
    SELECT customer_id, order_id, order_date, prev_order_date,
		
        -- разница дат: сколько дней между текущим и предыдущим заказом
        CASE
            WHEN prev_order_date IS NULL THEN NULL
            ELSE order_date - prev_order_date
        END AS diff_days
		
    FROM ordered_orders
),
max_intervals AS (
    SELECT customer_id, MAX(diff_days) AS max_diff_days
    FROM intervals
    WHERE diff_days IS NOT NULL -- клиенты с 1 заказом не попадают
    GROUP BY customer_id
)
SELECT c.customer_id, c.first_name, c.last_name, c.job_title, 
    max_intervals.max_diff_days AS max_interval_days
	
FROM max_intervals
JOIN customer c ON c.customer_id = max_intervals.customer_id

ORDER BY max_interval_days DESC, c.customer_id;


```
**Результат, Скриншоты:** 
запрос: https://disk.yandex.ru/i/wua_bu_j6_x1YA 
результаты: https://disk.yandex.ru/i/ppSFJsCqx3TtbQ
.


**Запрос 8.**
*Найти топ-5 клиентов (по общему доходу) в каждом сегменте благосостояния (wealth_segment). Вывести имя, фамилию, сегмент и общий доход. Если в сегменте менее 5 клиентов, вывести всех*

```sql

-- 8 - то5 клиентов по доходу в каждом сегменте 

WITH order_totals AS (
    SELECT o.order_id, o.customer_id,
        SUM(oi.quantity * p.list_price) AS order_amount
    FROM orders o
	
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN product p ON p.product_id = oi.product_id
	
    GROUP BY o.order_id, o.customer_id
),
customer_sum AS (
    SELECT c.customer_id, c.first_name, c.last_name, c.wealth_segment,
        COALESCE(SUM(ot.order_amount), 0) AS total_amount
		
    FROM customer c
	
    LEFT JOIN order_totals ot ON ot.customer_id = c.customer_id
    GROUP BY c.customer_id, c.first_name, c.last_name, c.wealth_segment
),

ranked AS (
    SELECT customer_id, first_name, last_name, wealth_segment, total_amount,
	
        DENSE_RANK() OVER ( 
            PARTITION BY wealth_segment
            ORDER BY total_amount DESC
        ) AS rnk
		
    FROM customer_sum
)

SELECT customer_id, first_name, last_name, wealth_segment, total_amount

FROM ranked WHERE rnk <= 5 -- топ-5 в каждом сегменте

ORDER BY wealth_segment, total_amount DESC, customer_id;


```
**Результат, Скриншоты:** 
запрос: https://disk.yandex.ru/i/c9Q-sdjs66DnmQ
результаты: https://disk.yandex.ru/i/ppSFJsCqx3TtbQ
.
.






.
.

