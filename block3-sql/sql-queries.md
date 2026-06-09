# SQL-запросы (PostgreSQL)

## 1. Вторая зарплата (Second Highest Salary)

### Условие
Дана таблица `Employee (id INT PRIMARY KEY, salary INT)`. Найти вторую по величине **уникальную** зарплату. Если такой нет — вернуть `NULL`.

### Решение

```sql
-- Способ 1: через OFFSET (идиоматичный для PostgreSQL)
SELECT DISTINCT salary AS "SecondHighestSalary"
FROM Employee
ORDER BY salary DESC
LIMIT 1 OFFSET 1;

-- Если записей меньше двух — OFFSET 1 вернёт пустой набор,
-- что эквивалентно NULL в контексте задачи.
```

```sql
-- Способ 2: через подзапрос + MAX (вернёт NULL гарантированно)
SELECT MAX(salary) AS "SecondHighestSalary"
FROM Employee
WHERE salary < (SELECT MAX(salary) FROM Employee);
```

```sql
-- Способ 3: оконная функция DENSE_RANK (явно возвращает NULL при отсутствии)
SELECT
    CASE WHEN COUNT(*) >= 2 THEN MIN(salary) ELSE NULL END AS "SecondHighestSalary"
FROM (
    SELECT salary
    FROM (
        SELECT DISTINCT salary,
               DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
        FROM Employee
    ) ranked
    WHERE rnk <= 2
) sub;
```

### Пояснение
- **Способ 1** — самый лаконичный. `DISTINCT` обеспечивает уникальность. `OFFSET 1` пропускает максимальную. Если в таблице только одна уникальная зарплата — возвращается пустой набор строк, что трактуется как `NULL`.
- **Способ 2** — классический подход. Гарантированно возвращает `NULL`, если второй зарплаты нет (MAX от пустого набора = NULL).
- **Способ 3** — избыточен для данной задачи, но демонстрирует понимание оконных функций. Не рекомендуется в production из-за лишней сложности.

### Рекомендуемый вариант: **Способ 1** или **Способ 2** — оба просты и эффективны.

---

## 2. Поиск дубликатов email

### Условие
Дана таблица `Person (id INT PRIMARY KEY, email VARCHAR)`. Найти все email-адреса, встречающиеся более одного раза.

### Решение

```sql
SELECT email
FROM Person
GROUP BY email
HAVING COUNT(*) > 1;
```

### Пояснение
- `GROUP BY email` группирует записи по адресу
- `HAVING COUNT(*) > 1` оставляет только группы, где email встречается 2+ раз
- Если нужно также показать количество дубликатов — добавить `COUNT(*) AS duplicate_count` в SELECT

### Вариант с количеством дубликатов

```sql
SELECT email, COUNT(*) AS occurrences
FROM Person
GROUP BY email
HAVING COUNT(*) > 1
ORDER BY occurrences DESC;
```

### Edge-кейсы (важно для QA)
- **NULL-значения:** `COUNT(*)` считает строки с NULL, но `GROUP BY email` помещает все NULL в одну группу. Если в таблице два человека с email = NULL, `HAVING COUNT(*) > 1` их вернёт. Правильно ли это? Зависит от бизнес-правил.
- **Регистр:** `'User@Mail.Ru'` и `'user@mail.ru'` — разные строки. Если нужна регистронезависимая проверка — `GROUP BY LOWER(email)`.
- **Пробелы:** `'user@mail.ru'` и `' user@mail.ru '` — разные. Нужен ли TRIM?

### Более строгий вариант (регистронезависимый + trimmed)

```sql
SELECT LOWER(TRIM(email)) AS normalized_email, COUNT(*) AS occurrences
FROM Person
GROUP BY LOWER(TRIM(email))
HAVING COUNT(*) > 1;
```

---

## 3. Клиенты без заказов

### Условие
Даны таблицы:
- `Customers (id INT PRIMARY KEY, name VARCHAR)`
- `Orders (id INT PRIMARY KEY, customerId INT REFERENCES Customers(id))`

Найти всех клиентов, ни разу не сделавших заказ.

### Решение

```sql
-- Способ 1: LEFT JOIN + проверка на NULL (классический)
SELECT c.id, c.name
FROM Customers c
LEFT JOIN Orders o ON c.id = o.customerId
WHERE o.id IS NULL;
```

```sql
-- Способ 2: NOT EXISTS (часто эффективнее на больших объёмах)
SELECT c.id, c.name
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM Orders o
    WHERE o.customerId = c.id
);
```

```sql
-- Способ 3: NOT IN (осторожно с NULL в подзапросе!)
SELECT c.id, c.name
FROM Customers c
WHERE c.id NOT IN (
    SELECT DISTINCT customerId
    FROM Orders
    WHERE customerId IS NOT NULL  -- обязательно фильтровать NULL!
);
```

### Пояснение

| Способ | Плюсы | Минусы | Когда использовать |
|--------|-------|--------|-------------------|
| LEFT JOIN + IS NULL | Интуитивно понятен, легко читается | Может быть медленнее на больших таблицах | Средние объёмы, читаемость важнее |
| NOT EXISTS | Часто самый быстрый, короткое замыкание | Менее очевиден для начинающих | Большие таблицы, оптимизация |
| NOT IN | Простой синтаксис | **Опасен:** если в подзапросе есть NULL — вернёт пустой результат! | Не рекомендуется. Фильтр `IS NOT NULL` обязателен |

### Рекомендуемый вариант: **Способ 1 (LEFT JOIN) или Способ 2 (NOT EXISTS)**.

---

## Дополнительно: проверка корректности (тестовые данные)

```sql
-- Тестовые данные
INSERT INTO Customers VALUES (1, 'ООО Ромашка'), (2, 'ИП Иванов'), (3, 'АО Заря'), (4, 'ООО Пустота');
INSERT INTO Orders VALUES (1, 1), (2, 1), (3, 2), (4, 3);

-- Запрос: клиенты без заказов
SELECT c.id, c.name
FROM Customers c
LEFT JOIN Orders o ON c.id = o.customerId
WHERE o.id IS NULL;

-- Ожидаемый результат:
-- id | name
--  4 | ООО Пустота
```
