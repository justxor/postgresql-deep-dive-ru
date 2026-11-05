# PostgreSQL: задачи с подвохом

Шесть самостоятельных задач для PostgreSQL 16. Сначала попробуйте решить их сами, затем откройте разбор. Каждая задача содержит данные и ожидаемый результат. Выполняйте всё в отдельной учебной базе из [основного курса](README.md), без реальных данных. В заданиях 1-4 используются CTE, они ничего не сохраняют; задания 5-6 создают учебные таблицы.

## 1. Последнее событие с одинаковым временем

У пользователя может быть несколько событий в одну секунду. Найдите **одно последнее событие каждого пользователя**; при равном времени выбирайте больший id. Отсортируйте результат по user_id.

```sql
WITH events(id, user_id, happened_at, event) AS (
  VALUES (1, 10, timestamp '2026-01-01 10:00', 'login'),
         (2, 10, timestamp '2026-01-01 10:00', 'logout'),
         (3, 20, timestamp '2026-01-02 12:00', 'login'),
         (4, 20, timestamp '2026-01-01 12:00', 'login')
)
-- Напишите запрос здесь
SELECT * FROM events;
```

**Ожидается:** для пользователя 10 - id 2, для пользователя 20 - id 3. Почему JOIN по MAX(happened_at) может вернуть лишнюю строку?

<details><summary>Решение и объяснение</summary>

```sql
WITH events(id, user_id, happened_at, event) AS (
  VALUES (1, 10, timestamp '2026-01-01 10:00', 'login'),
         (2, 10, timestamp '2026-01-01 10:00', 'logout'),
         (3, 20, timestamp '2026-01-02 12:00', 'login'),
         (4, 20, timestamp '2026-01-01 12:00', 'login')
), ranked AS (
  SELECT *, row_number() OVER (
    PARTITION BY user_id ORDER BY happened_at DESC, id DESC
  ) AS rn FROM events
)
SELECT user_id, id, event FROM ranked WHERE rn = 1 ORDER BY user_id;
```

Одного MAX времени недостаточно для выбора при ничьей: обе строки пользователя 10 совпадут с максимумом. Составной порядок делает выбор детерминированным. В PostgreSQL эту задачу также можно решить через DISTINCT ON.
</details>

## 2. Периоды активности с пропущенными днями

Найдите непрерывные серии дней активности каждого пользователя. Повторная активность в один день считается одним днём. В результате нужны user_id, начало, конец и длина серии.

```sql
WITH activity(user_id, day) AS (
  VALUES (1, date '2026-01-01'), (1, date '2026-01-02'),
         (1, date '2026-01-02'), (1, date '2026-01-04'),
         (2, date '2026-01-03'), (2, date '2026-01-04'),
         (2, date '2026-01-05')
)
SELECT * FROM activity;
```

**Ожидается:** у пользователя 1 серии 1-2 января длиной 2 и 4 января длиной 1; у пользователя 2 серия 3-5 января длиной 3.

<details><summary>Решение и объяснение</summary>

```sql
WITH activity(user_id, day) AS (
  VALUES (1, date '2026-01-01'), (1, date '2026-01-02'),
         (1, date '2026-01-02'), (1, date '2026-01-04'),
         (2, date '2026-01-03'), (2, date '2026-01-04'),
         (2, date '2026-01-05')
), days AS (
  SELECT DISTINCT user_id, day FROM activity
), numbered AS (
  SELECT user_id, day,
         day - (row_number() OVER (PARTITION BY user_id ORDER BY day))::integer AS grp
  FROM days
)
SELECT user_id, min(day) AS start_day, max(day) AS end_day, count(*) AS days
FROM numbered GROUP BY user_id, grp ORDER BY user_id, start_day;
```

При последовательных датах разность даты и порядкового номера постоянна. DISTINCT до нумерации исключает повторные события. Для данных с часовым поясом сначала определите бизнес-часовой пояс, по которому событие относится к дню.
</details>

## 3. Товары, купленные во всех заказах

Найдите товары, встречающиеся **в каждом** заказе клиента. Повтор товара внутри одного заказа не должен влиять на результат. Клиентов без заказов в результат не включайте.

```sql
WITH items(customer_id, order_id, product) AS (
  VALUES (1, 101, 'book'), (1, 101, 'pen'), (1, 101, 'pen'),
         (1, 102, 'pen'), (1, 102, 'book'), (1, 103, 'pen'),
         (2, 201, 'book'), (2, 202, 'book'), (2, 202, 'mouse')
)
SELECT * FROM items;
```

**Ожидается:** (1, pen), (2, book).

<details><summary>Решение и объяснение</summary>

```sql
WITH items(customer_id, order_id, product) AS (
  VALUES (1, 101, 'book'), (1, 101, 'pen'), (1, 101, 'pen'),
         (1, 102, 'pen'), (1, 102, 'book'), (1, 103, 'pen'),
         (2, 201, 'book'), (2, 202, 'book'), (2, 202, 'mouse')
), order_counts AS (
  SELECT customer_id, count(DISTINCT order_id) AS n
  FROM items GROUP BY customer_id
)
SELECT i.customer_id, i.product
FROM items i JOIN order_counts c USING (customer_id)
GROUP BY i.customer_id, i.product, c.n
HAVING count(DISTINCT i.order_id) = c.n
ORDER BY i.customer_id, i.product;
```

DISTINCT предотвращает влияние повтора товара. В этой постановке список заказов выводится из items. В реальной схеме заказы без позиций хранятся отдельно: знаменатель нужно считать по таблице заказов, иначе результат изменится.
</details>

## 4. Скользящая сумма: строки или даты?

Посчитайте сумму за текущий день и два предыдущих **календарных дня**, включая дни без продаж как нулевые. Верните только даты, когда продажи были.

```sql
WITH sales(day, amount) AS (
  VALUES (date '2026-01-01', 10), (date '2026-01-01', 5),
         (date '2026-01-03', 20), (date '2026-01-05', 7)
)
SELECT * FROM sales;
```

**Ожидается:** 1 января - 15, 3 января - 35, 5 января - 27. Почему ROWS 2 PRECEDING без заполнения календаря даёт другой смысл?

<details><summary>Решение и объяснение</summary>

```sql
WITH sales(day, amount) AS (
  VALUES (date '2026-01-01', 10), (date '2026-01-01', 5),
         (date '2026-01-03', 20), (date '2026-01-05', 7)
), daily AS (
  SELECT day, sum(amount) AS amount FROM sales GROUP BY day
), calendar AS (
  SELECT g::date AS day FROM generate_series(
    timestamp '2026-01-01', timestamp '2026-01-05', interval '1 day'
  ) AS g
), rolling AS (
  SELECT c.day, d.amount IS NOT NULL AS had_sale,
         sum(coalesce(d.amount, 0)) OVER (
           ORDER BY c.day ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
         ) AS total
  FROM calendar c LEFT JOIN daily d USING (day)
)
SELECT day, total FROM rolling WHERE had_sale ORDER BY day;
```

ROWS считает предыдущие строки, а не дни. Календарь создаёт отсутствующие даты, группировка до окна объединяет продажи за день. Фильтр had_sale применяется после окна, иначе нулевые дни снова пропадут.
</details>

## 5. Идемпотентное создание платежа

Клиент повторил запрос после timeout. Создайте схему, в которой один idempotency key не создаст два платежа, а повтор с другим содержимым будет обнаружен. Это учебная модель без реального платёжного API.

```sql
CREATE TABLE challenge_payments (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  request_key text NOT NULL UNIQUE,
  amount integer NOT NULL CHECK (amount > 0)
);
```

**Требование:** для ключа k1 и суммы 500 при повторе в таблице ровно одна запись. При повторе k1 с суммой 700 нельзя молча считать его успешным тем же запросом. Что делать при конкурентных запросах?

<details><summary>Вариант решения и объяснение</summary>

```sql
INSERT INTO challenge_payments(request_key, amount)
VALUES ('k1', 500)
ON CONFLICT (request_key) DO NOTHING
RETURNING id, request_key, amount;

-- Если RETURNING не вернул строку, отдельным оператором прочитайте запись:
SELECT id, request_key, amount
FROM challenge_payments WHERE request_key = 'k1';
```

В READ COMMITTED уникальный индекс разрешает конкурентный конфликт, а следующий SELECT получает новый снимок. Приложение сравнивает сохранённые параметры с запросом: совпадение - возвращает сохранённый результат; расхождение - ошибка конфликта. Если запись может удаляться или изменяться параллельно, это нужно отдельно защитить; в REPEATABLE READ или SERIALIZABLE следует учитывать ошибки сериализации и повторять всю транзакцию. Для реальных платежей дополнительно нужны состояние обработки, атомарность бизнес-операции и согласование с внешним провайдером. Уникальный ключ сам по себе не гарантирует exactly-once для внешних вызовов.
</details>

## 6. Частичный уникальный индекс и мягкое удаление

Для каждого пользователя может быть только одна активная подписка, но история отменённых подписок сохраняется.

```sql
CREATE TABLE challenge_subscriptions (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id bigint NOT NULL,
  cancelled_at timestamptz
);
```

**Требование:** две отменённые записи для пользователя допустимы; две строки с cancelled_at IS NULL - нет. Покажите, как база защитит правило при конкурентных вставках.

<details><summary>Решение и объяснение</summary>

```sql
CREATE UNIQUE INDEX challenge_one_active_subscription
ON challenge_subscriptions(user_id)
WHERE cancelled_at IS NULL;

INSERT INTO challenge_subscriptions(user_id, cancelled_at)
VALUES (1, now()), (1, now()), (1, NULL);
-- Следующая вставка должна завершиться unique_violation:
INSERT INTO challenge_subscriptions(user_id, cancelled_at) VALUES (1, NULL);
```

Частичный уникальный индекс проверяет уникальность только среди активных строк, в том числе при конкурентных вставках. Ожидаемую ошибку выполняйте отдельно; если она произошла внутри BEGIN, до продолжения нужен ROLLBACK или откат к savepoint. UNIQUE(user_id, cancelled_at) с обычной семантикой NULL не обеспечит данное правило.
</details>

## Дополнительный челлендж: найдите причину медленного запроса

В [продвинутом практикуме](ADVANCED.md) создайте lab_events. Для tenant_id=2 выберите 10 последних событий типа error. Запишите EXPLAIN (ANALYZE, BUFFERS) до и после подходящего индекса и ответьте:

1. Совпали ли оценка и фактическое число строк на ключевом узле?
2. Где выполняется сортировка и как влияет LIMIT?
3. Что означают shared hit и shared read, и почему read не гарантирует физический I/O?
4. Как изменится стоимость INSERT и объём индекса?
5. Повторяется ли вывод на других объёмах и распределениях?

Планы зависят от версии, статистики, кэша и конфигурации. Не отключайте Seq Scan только ради демонстрации и не сравнивайте выводы по одному запуску.

После упражнений удалите **только свои учебные таблицы** challenge_payments и challenge_subscriptions, если данные больше не нужны. Упражнения рассчитаны на изолированную базу; не запускайте их на production.
