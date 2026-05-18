```sql
SELECT column_list
FROM table1
[JOIN_TYPE] JOIN table2
ON table1.column_name = table2.column_name;
```


---

## Исходные данные для примеров

**Таблица `employees` (сотрудники):**

| emp_id | name    | dept_id |
|--------|---------|---------|
| 1      | Alice   | 101     |
| 2      | Bob     | 102     |
| 3      | Charlie | NULL    |
| 4      | Diana   | 104     |

**Таблица `departments` (отделы):**

| dept_id | dept_name |
|---------|-----------|
| 101     | Sales     |
| 102     | IT        |
| 103     | HR        |

---

## 1. `INNER JOIN` (или просто `JOIN`)

**Что делает:** Возвращает ТОЛЬКО те строки, где есть совпадение в ОБЕИХ таблицах.

```sql
SELECT e.name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;
```

**Результат:**

| name  | dept_name |
|-------|-----------|
| Alice | Sales     |
| Bob   | IT        |

**Что произошло:**
- ✅ Alice (dept_id=101) → совпало с departments → Sales
- ✅ Bob (dept_id=102) → совпало с departments → IT
- ❌ Charlie (dept_id=NULL) → нет совпадения → исключен
- ❌ Diana (dept_id=104) → нет в departments → исключена
- ❌ HR (dept_id=103) → нет сотрудников → не попал

**Когда использовать:** Когда нужны ТОЛЬКО записи, у которых есть данные в обеих таблицах.

---

## 2. `LEFT JOIN` (или `LEFT OUTER JOIN`)

**Что делает:** Возвращает ВСЕ строки из левой таблицы + совпадения из правой. Если совпадений нет → в колонках правой таблицы будет `NULL`.

```sql
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id;
```

**Результат:**

| name    | dept_name |
|---------|-----------|
| Alice   | Sales     |
| Bob     | IT        |
| Charlie | NULL      |
| Diana   | NULL      |

**Что произошло:**
- ✅ Все сотрудники из `employees` (левая таблица) ВСЕ попали в результат
- ✅ У Alice и Bob → есть департамент
- ❌ У Charlie и Diana → нет департамента, поэтому `NULL`

**Когда использовать:** Когда нужно показать ВСЕ записи из основной таблицы, даже если нет связанных данных.

---

## 3. `RIGHT JOIN` (или `RIGHT OUTER JOIN`)

**Что делает:** Возвращает ВСЕ строки из правой таблицы + совпадения из левой. (По сути, то же что `LEFT JOIN`, но таблицы меняются местами)

```sql
SELECT e.name, d.dept_name
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.dept_id;
```

**Результат:**

| name  | dept_name |
|-------|-----------|
| Alice | Sales     |
| Bob   | IT        |
| NULL  | HR        |

**Что произошло:**
- ✅ Все отделы из `departments` (правая таблица) ВСЕ попали
- ✅ Sales и IT → есть сотрудники
- ❌ HR → нет сотрудников, поэтому `NULL` в колонках employees

**Когда использовать:** Редко (обычно используют `LEFT JOIN`, меняя порядок таблиц).

---

## 4. `FULL JOIN` (или `FULL OUTER JOIN`)

**Что делает:** Возвращает ВСЕ строки из ОБЕИХ таблиц. Если совпадения нет → `NULL` с той стороны, где данных нет.

```sql
SELECT e.name, d.dept_name
FROM employees e
FULL JOIN departments d ON e.dept_id = d.dept_id;
```

**Результат:**

| name    | dept_name |
|---------|-----------|
| Alice   | Sales     |
| Bob     | IT        |
| Charlie | NULL      |
| Diana   | NULL      |
| NULL    | HR        |

**Что произошло:**
- ✅ Все сотрудники (даже без отдела)
- ✅ Все отделы (даже без сотрудников)

**Когда использовать:** Когда нужно увидеть ПОЛНУЮ картину: и сотрудников без отдела, и отделы без сотрудников.

---

## 5. `CROSS JOIN`

**Что делает:** Декартово произведение — КАЖДАЯ строка из первой таблицы соединяется с КАЖДОЙ строкой из второй. (Без условия `ON`)

```sql
SELECT e.name, d.dept_name
FROM employees e
CROSS JOIN departments d;
```

**Результат:** (4 сотрудника × 3 отдела = 12 строк)

| name    | dept_name |
|---------|-----------|
| Alice   | Sales     |
| Alice   | IT        |
| Alice   | HR        |
| Bob     | Sales     |
| Bob     | IT        |
| Bob     | HR        |
| Charlie | Sales     |
| Charlie | IT        |
| Charlie | HR        |
| Diana   | Sales     |
| Diana   | IT        |
| Diana   | HR        |

**Когда использовать:** Очень редко. Например, для генерации тестовых данных или когда нужно соединить каждое с каждым (дни недели × время работы).

---

## Таблица сравнения (шпаргалка)

| Тип JOIN | Что возвращает                                  |
| -------- | ----------------------------------------------- |
| `INNER`  | Только совпадающие строки из обеих таблиц       |
| `LEFT`   | Все строки из левой + совпадения из правой      |
| `RIGHT`  | Все строки из правой + совпадения из левой      |
| `FULL`   | Все строки из обеих таблиц                      |
| `CROSS`  | Каждую строку с каждой (декартово произведение) |

---

## Визуальная диаграмма (классические круги Эйлера)

```
INNER JOIN:
   ┌─────┐
   │ ○○  │  (только пересечение)
   └─────┘

LEFT JOIN:
   ┌─────┐
   │ ●●●○│  (вся левая + пересечение)
   └─────┘

FULL JOIN:
   ┌─────┐
   │ ●●●●│  (все области)
   └─────┘
```

---

## Соединение нескольких таблиц

#### Основное правило

> **Джойнить таблицы можно последовательно: `FROM table1 JOIN table2 ON ... JOIN table3 ON ... JOIN table4 ON ...`**

Каждый следующий `JOIN` присоединяется к РЕЗУЛЬТАТУ предыдущих джойнов.

---

## пример: 3 таблицы

**Схема:**
- `employees` (emp_id, name, dept_id)
- `departments` (dept_id, dept_name, location_id)
- `locations` (location_id, city, country)

**Задача:** Показать имя сотрудника, название отдела и город, где находится отдел.

```sql
SELECT e.name, d.dept_name, l.city
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
JOIN locations l ON d.location_id = l.location_id;
```

### Что происходит по шагам:

**Шаг 1:** `FROM employees e JOIN departments d ON e.dept_id = d.dept_id`

| e.name | d.dept_name | d.location_id |
|--------|-------------|---------------|
| Alice  | Sales       | 100           |
| Bob    | IT          | 200           |

**Шаг 2:** `JOIN locations l ON d.location_id = l.location_id`

| e.name | d.dept_name | l.city           |
| ------ | ----------- | ---------------- |
| Alice  | Sales       | Moscow           |
| Bob    | IT          | Saint-Petersburg |

---

## Смешивание разных типов JOIN

```sql
SELECT e.name, d.dept_name, l.city
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id      -- Все сотрудники
JOIN locations l ON d.location_id = l.location_id;    -- Только у кого есть location
```

**Важно:** Порядок выполнения — СЛЕВА НАПРАВО. Результат первого JOIN становится "левой таблицей" для второго JOIN.

---

## Полный пример с 4 таблицами

**Схема:**
- `employees` (emp_id, name, dept_id, manager_id)
- `departments` (dept_id, dept_name, location_id)
- `locations` (location_id, city, country_id)
- `countries` (country_id, country_name)

**Запрос (сотрудник → отдел → локация → страна):**

```sql
SELECT 
    e.name AS employee_name,
    d.dept_name,
    l.city,
    c.country_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
JOIN locations l ON d.location_id = l.location_id
JOIN countries c ON l.country_id = c.country_id;
```

### Визуализация цепочки:

```
employees ──┬── departments ──┬── locations ──┬── countries
            │                  │               │
            └─ ON dept_id ─────┘               │
                               └─ ON location_id ─┘
                                               └─ ON country_id
```

---
## Ошибка новичка: забыл условие ON

```sql
-- ❌ ОШИБКА (декартово произведение, потом фильтрация)
SELECT e.name, d.dept_name, l.city
FROM employees e, departments d, locations l
WHERE e.dept_id = d.dept_id AND d.location_id = l.location_id;

-- ✅ ПРАВИЛЬНО (явные JOIN)
SELECT e.name, d.dept_name, l.city
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
JOIN locations l ON d.location_id = l.location_id;
```

**Почему первый вариант плох:** Сначала создается гигантская временная таблица (декартово произведение), потом она фильтруется. Это медленно и опасно.

---

## Уровни фильтрации

### Уровень 0: Внутри `ON` (фильтрация ДО соединения)

```sql
SELECT e.name, o.order_amount
FROM employees e
LEFT JOIN orders o ON e.emp_id = o.emp_id AND o.order_amount > 1000;
--                                    ↑ фильтруем заказы ДО того, как их присоединять
```

**Что происходит:** База данных сначала отбирает заказы с `amount > 1000`, а потом делает LEFT JOIN.

---

### Уровень 1: Фильтрация после КАЖДОГО JOIN (в подзапросах или CTE)

```sql
-- Способ 1: Подзапрос на первом JOIN
SELECT e.name, d.dept_name, o.order_amount
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
JOIN (
    SELECT * FROM orders WHERE order_amount > 1000  -- ← фильтр ДО второго JOIN
) o ON e.emp_id = o.emp_id;
```

```sql
-- Способ 2: CTE (читаемее)
WITH filtered_orders AS (
    SELECT * FROM orders WHERE order_amount > 1000  -- ← фильтр ДО всех JOIN
)
SELECT e.name, d.dept_name, fo.order_amount
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
JOIN filtered_orders fo ON e.emp_id = fo.emp_id;
```

---

### Уровень 2: Фильтрация ПОСЛЕ всех JOIN (классический `WHERE`)

```sql
SELECT e.name, d.dept_name, o.order_amount
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
JOIN orders o ON e.emp_id = o.emp_id
WHERE o.order_amount > 1000;  -- ← фильтр ПОСЛЕ того, как всё соединилось
```

---

## Главный совет

> **Представь, что каждый JOIN — это воронка. Чем раньше ты отсеешь ненужные строки, тем меньше данных пойдет через следующие воронки. Фильтруй на каждом этапе, а не только в конце.**

### Приоритет фильтрации (от самого раннего к позднему)

| Уровень     | Где писать         | Когда применяется | Скорость         |
| ----------- | ------------------ | ----------------- | ---------------- |
| 1 (лучший)  | В подзапросе / CTE | ДО любого JOIN    | 🔥 Самый быстрый |
| 2 (хороший) | В `ON`             | В момент JOIN     | 👍 Быстрый       |
| 3 (норма)   | В `WHERE`          | После всех JOIN   | 👌 Средний       |
| 4 (плохой)  | В `HAVING`         | После GROUP BY    | 🐌 Медленный     |

### 1️⃣ Фильтруй КАК МОЖНО РАНЬШЕ

> **Чем меньше строк идет в JOIN, тем быстрее работает запрос**

```sql
-- ❌ ПЛОХО (фильтр после всех JOIN)
SELECT e.name, o.amount
FROM employees e
JOIN orders o ON e.id = o.emp_id
WHERE o.order_date >= '2024-01-01';

-- ✅ ХОРОШО (фильтр ДО JOIN)
SELECT e.name, o.amount
FROM employees e
JOIN orders o ON e.id = o.emp_id AND o.order_date >= '2024-01-01';
```

---

### 2️⃣ LEFT JOIN + WHERE по правой таблице = INNER JOIN

> **Если фильтруешь правую таблицу в WHERE, LEFT JOIN теряет смысл**

```sql
-- ❌ ОШИБКА (превращается в INNER JOIN)
SELECT e.name, o.amount
FROM employees e
LEFT JOIN orders o ON e.id = o.emp_id
WHERE o.amount > 1000;  -- NULL-строки отфильтруются

-- ✅ ПРАВИЛЬНО (фильтр в ON)
SELECT e.name, o.amount
FROM employees e
LEFT JOIN orders o ON e.id = o.emp_id AND o.amount > 1000;
```


---

### 4️⃣ Для 3+ таблиц - используй CTE (Common Table Expression)

```sql
-- ✅ ОПТИМАЛЬНО (фильтруем каждую таблицу до JOIN)
WITH 
-- CTE #1: Активные сотрудники с высокой зарплатой
active_employees AS (
    -- Берём ВСЕХ сотрудников, но сразу отсеиваем ненужных
    SELECT * FROM employees 
    WHERE status = 'ACTIVE'      -- ← Оставляем только работающих
      AND salary > 50000         -- ← Оставляем только высокооплачиваемых
),

-- CTE #2: Недавние заказы (фильтрация по дате)
recent_orders AS (
    -- Берём ВСЕ заказы, но фильтруем по дате
    SELECT * FROM orders 
    WHERE order_date >= '2024-01-01'  -- ← Только заказы с начала года
),

-- CTE #3: Крупные заказы (фильтрация по сумме)
high_value_orders AS (
    -- Берём уже отфильтрованные по дате заказы
    SELECT * FROM recent_orders 
    WHERE amount > 1000           -- ← Только заказы на сумму > 1000
),

-- ГЛАВНЫЙ ЗАПРОС (использует все три CTE)
SELECT 
    ae.name,                 -- ← Имя из CTE active_employees
    hvo.amount,              -- ← Сумма из CTE high_value_orders
    d.dept_name              -- ← Название отдела из таблицы departments
FROM active_employees ae     -- ← Начинаем с отфильтрованных сотрудников
JOIN high_value_orders hvo   -- ← Соединяем с отфильтрованными заказами
    ON ae.id = hvo.emp_id    -- ← Условие: ID сотрудника = ID заказа
JOIN departments d           -- ← Добавляем отделы (фильтрация НЕ нужна)
    ON ae.dept_id = d.id;    -- ← Условие: ID отдела в employees = ID отдела
-- Результат: Активные сотрудники (>50000), их крупные заказы (>1000 за 2024+),
--           с названием их отдела
```




