# PostgreSQL: продвинутые лабораторные

Продолжение [основного курса](README.md). Здесь вы воспроизведёте проблемы планировщика и конкурентного доступа, а затем сравните решения. Примеры рассчитаны на PostgreSQL 16 и отдельную учебную базу `shop`. Не выполняйте их в production: создание индексов и генерация данных потребляют ресурсы и могут блокировать операции.

**Перед началом:** выполните раздел 0 основного курса. Откройте psql: `docker compose exec db psql -U student -d shop`. Для последовательных лабораторных работ используйте одну сессию, где указано иное. В многооконных примерах откройте два отдельных терминала и psql в каждом. Если включён `\set ON_ERROR_STOP on`, ожидаемые ошибки в интерактивном psql показываются в сессии; при выполнении из файла psql прекратит обработку, поэтому не запускайте многооконные блоки как один скрипт.

## Маршрут

1. [Селективность, статистика и планы](#1-селективность-статистика-и-планы)
2. [Составной, частичный и покрывающий индекс](#2-составной-частичный-и-покрывающий-индекс)
3. [Пагинация без OFFSET](#3-пагинация-без-offset)
4. [Конкурентное обновление и атомарность](#4-конкурентное-обновление-и-атомарность)
5. [Deadlock и порядок блокировок](#5-deadlock-и-порядок-блокировок)
6. [Очередь с SKIP LOCKED](#6-очередь-с-skip-locked)
7. [VACUUM, длинные транзакции и наблюдение](#7-vacuum-длинные-транзакции-и-наблюдение)
8. [Ошибки SQL с готовыми проверками](#8-ошибки-sql-с-готовыми-проверками)
9. [Разбор инцидента и итог](#9-разбор-инцидента-и-итог)

## 1. Селективность, статистика и планы

Планировщик оценивает число строк и стоимость возможных путей. Он не выполняет все варианты для сравнения. Статистика - выборка и сводка распределения, а не точная карта данных. Если значения коррелируют, оценка для комбинации условий может заметно отличаться от реальности.

### Лабораторная: перекос значений

В отдельной учебной базе создайте таблицу с предсказуемым распределением. Имя должно быть свободно; повторный запуск CREATE завершится ошибкой.

```sql
CREATE TABLE lab_events (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id integer NOT NULL,
  kind text NOT NULL,
  created_at timestamptz NOT NULL
);
INSERT INTO lab_events (tenant_id, kind, created_at)
SELECT CASE WHEN g <= 90000 THEN 1 ELSE 2 END,
       CASE WHEN g % 10 = 0 THEN 'error' ELSE 'ok' END,
       timestamptz '2026-01-01 00:00+00' + g * interval '1 second'
FROM generate_series(1, 100000) AS g;
ANALYZE lab_events;
SELECT tenant_id, count(*) FROM lab_events GROUP BY tenant_id ORDER BY tenant_id;
```

**Проверка:** tenant 1 содержит 90 000 строк, tenant 2 - 10 000. Теперь сравните:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM lab_events WHERE tenant_id = 1;
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM lab_events WHERE tenant_id = 2;
```

На каждом плане найдите `rows` оценки и `actual rows`, число loops, тип scan и сведения о буферах. EXPLAIN ANALYZE выполняет запрос; не применяйте его к изменяющим данным операторам без понимания последствий. Buffers показывают обращения PostgreSQL к shared/local/temp буферам; shared read не гарантирует физическое чтение именно с диска: данные могут быть в кэше ОС.

**Задание:** создайте индекс `CREATE INDEX lab_events_tenant_idx ON lab_events (tenant_id);`, повторите оба плана и объясните различия. Индекс не обязан победить: решение зависит от стоимости, видимости страниц, кэша и настроек. Не делайте вывод из одного запуска и не отключайте Seq Scan ради демонстрации.

### Коррелированные столбцы

При необходимости расширенная статистика помогает некоторым оценкам комбинаций. Например, при наличии зависимости между колонками рассмотрите `CREATE STATISTICS ... (dependencies, mcv) ON col1, col2 FROM table; ANALYZE table;`. Она не создаёт индекс и не гарантирует точную оценку любого выражения или соединения. Сначала подтвердите ошибку оценки на конкретном запросе.

## 2. Составной, частичный и покрывающий индекс

Условия, сортировка и доля подходящих строк определяют пользу индекса. У B-tree порядок колонок влияет на диапазон просмотра; индекс также добавляет цену INSERT/UPDATE/DELETE и занимает место. Условия частичного индекса должны быть доказуемы планировщиком для конкретного запроса.

```sql
CREATE INDEX lab_events_recent_errors_idx
  ON lab_events (tenant_id, created_at DESC, id DESC)
  WHERE kind = 'error';
ANALYZE lab_events;
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, created_at
FROM lab_events
WHERE tenant_id = 2 AND kind = 'error'
ORDER BY created_at DESC, id DESC
LIMIT 10;
```

**Проверка результата:** 10 строк, первый `id = 100000`, последний `id = 99910`; даты упорядочены по убыванию. План зависит от среды. Сравните с запросом `kind = 'ok'`: частичный индекс ошибок не покрывает нужный набор.

`INCLUDE` позволяет хранить дополнительные значения в B-tree для возможного index-only scan, но увеличивает индекс и стоимость записи. Даже если все колонки доступны в индексе, PostgreSQL может обращаться к heap для проверки видимости; смотрите `Heap Fetches`. Применение индекса не является самоцелью: сравнивайте время, буферы и стоимость изменений.

## 3. Пагинация без OFFSET

Большой OFFSET требует пропустить предыдущие строки и при изменениях между запросами может сдвигать страницы. Keyset pagination передаёт последнюю пару ключей. Нужен полный стабильный порядок с уникальным tie-breaker.

```sql
-- Первая страница ошибок tenant 2
SELECT id, created_at FROM lab_events
WHERE tenant_id = 2 AND kind = 'error'
ORDER BY created_at DESC, id DESC
LIMIT 3;

-- Следующая страница: значения последней строки первой страницы
SELECT id, created_at FROM lab_events
WHERE tenant_id = 2 AND kind = 'error'
  AND (created_at, id) < (timestamptz '2026-01-02 03:46:20+00', 99980)
ORDER BY created_at DESC, id DESC
LIMIT 3;
```

При созданном наборе первая страница - IDs 100000, 99990, 99980; вторая - 99970, 99960, 99950. Для другого набора используйте фактический курсор. Keyset не фиксирует снимок между отдельными запросами: обновления сортировочных полей и новые записи влияют на результат. Если требуется единый снимок, продумайте транзакцию, срок её жизни и влияние на vacuum.

## 4. Конкурентное обновление и атомарность

Проверка баланса отдельным SELECT и последующий UPDATE в приложении могут работать на устаревшем значении. Выражение в одном UPDATE выполняет проверку и изменение как одну операцию над строкой.

```sql
CREATE TABLE lab_inventory (
  sku text PRIMARY KEY,
  quantity integer NOT NULL CHECK (quantity >= 0)
);
INSERT INTO lab_inventory VALUES ('book', 1);
UPDATE lab_inventory
SET quantity = quantity - 1
WHERE sku = 'book' AND quantity >= 1
RETURNING quantity;
UPDATE lab_inventory
SET quantity = quantity - 1
WHERE sku = 'book' AND quantity >= 1
RETURNING quantity;
```

**Проверка:** первый UPDATE возвращает 0, второй - 0 строк. Приложение должно проверять число возвращённых строк. Для инвариантов, охватывающих несколько строк, одной такой операции может быть недостаточно: рассмотрите ограничения, блокировки или Serializable с повтором всей транзакции. Нулевое число строк может означать и отсутствующий SKU - различайте состояния по требованиям API.

## 5. Deadlock и порядок блокировок

Сделайте упражнение только в учебной базе. Откройте **две независимые psql-сессии**. Для ограничения ожидания используйте `SET LOCAL statement_timeout = '10s'` в каждой транзакции. Подготовьте данные один раз:

```sql
CREATE TABLE lab_locks (id integer PRIMARY KEY, value integer NOT NULL);
INSERT INTO lab_locks VALUES (1, 0), (2, 0);
```

Последовательность:

| Шаг | Сессия A | Сессия B |
|---|---|---|
| 1 | BEGIN; SET LOCAL statement_timeout='10s'; UPDATE lab_locks SET value=value+1 WHERE id=1; | |
| 2 | | BEGIN; SET LOCAL statement_timeout='10s'; UPDATE lab_locks SET value=value+1 WHERE id=2; |
| 3 | UPDATE lab_locks SET value=value+1 WHERE id=2; - ожидает | |
| 4 | | UPDATE lab_locks SET value=value+1 WHERE id=1; |

Для шага 4 отправьте запрос B, пока A ждёт. PostgreSQL обнаруживает цикл и прерывает одну транзакцию с SQLSTATE 40P01; другая может продолжить работу. Какой участник станет жертвой, не определено. Если вместо deadlock первым сработал timeout, завершите обе транзакции и повторите шаги быстрее. После наблюдения выполните `ROLLBACK;` в обеих сессиях, в том числе в успешно продолжившейся.

**Вывод:** единый порядок захвата строк уменьшает риск цикла, но не исключает всех deadlock. Приложение должно корректно повторять **всю** безопасную к повтору транзакцию, а не только последний запрос; внешние побочные эффекты требуют отдельной идемпотентности.

## 6. Очередь с SKIP LOCKED

`SKIP LOCKED` позволяет нескольким работникам выбирать разные незаблокированные задания. Он намеренно возвращает неполную картину и подходит не для любого отчёта. Блокировка снимается в конце транзакции; если держать её во время долгой внешней операции, возрастут время транзакции и нагрузка на систему.

```sql
CREATE TABLE lab_jobs (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  status text NOT NULL DEFAULT 'ready',
  payload text NOT NULL
);
INSERT INTO lab_jobs (payload) VALUES ('one'), ('two'), ('three');
```

В A:

```sql
BEGIN;
SELECT id FROM lab_jobs WHERE status = 'ready'
ORDER BY id FOR UPDATE SKIP LOCKED LIMIT 1;
```

Оставьте транзакцию открытой. В B выполните тот же блок: ожидается следующий id. Затем выполните `ROLLBACK;` в обеих сессиях. Это демонстрация блокировки, а не законченная очередь: реальной системе нужны атомарное изменение состояния, lease/восстановление после падения, повторные попытки, идемпотентный обработчик и контроль starvation. `SKIP LOCKED` не гарантирует FIFO при конкуренции.

## 7. VACUUM, длинные транзакции и наблюдение

В MVCC обновление создаёт новую версию строки. VACUUM может освобождать место для повторного использования, но старый снимок способен удерживать нужные ему версии. `n_dead_tup` в статистике приблизителен и обновляется не мгновенно. Длительная транзакция не всегда держит старый snapshot: это зависит от уровня изоляции и состояния; для демонстрации используйте Repeatable Read после первого SELECT.

```sql
CREATE TABLE lab_versions (id integer PRIMARY KEY, value integer NOT NULL);
INSERT INTO lab_versions VALUES (1, 0);
```

В A: `BEGIN ISOLATION LEVEL REPEATABLE READ; SELECT * FROM lab_versions;`. В B несколько раз выполните `UPDATE lab_versions SET value=value+1 WHERE id=1;`, затем `VACUUM (ANALYZE) lab_versions;` вне блока транзакции. Повторите SELECT в A: он видит прежнюю версию. Завершите A командой COMMIT и снова выполните VACUUM в B.

Сравните приблизительные показатели `SELECT relname, n_live_tup, n_dead_tup, last_vacuum FROM pg_stat_user_tables WHERE relname='lab_versions';`. Не ожидайте точного значения и немедленного уменьшения размера файла ОС. Восстановление места зависит от страниц, снимков и работы vacuum. Не запускайте VACUUM FULL в production без оценки эксклюзивной блокировки и дискового пространства.

## 8. Ошибки SQL с готовыми проверками

### NULL и NOT IN

```sql
SELECT 1 WHERE 1 NOT IN (2, NULL);
SELECT 1 WHERE NOT EXISTS (SELECT 1 FROM (VALUES (2), (NULL)) AS x(v) WHERE x.v = 1);
```

**Проверка:** первый запрос возвращает 0 строк, второй - одну. NULL делает сравнение unknown; WHERE пропускает только true. Учитывайте и возможный NULL в проверяемом значении при выборе семантики.

### JOIN и размножение суммы

```sql
SELECT sum(o.total) AS incorrect_sum
FROM orders o JOIN order_items i ON i.order_id = o.id
WHERE o.id = 1;
SELECT total FROM orders WHERE id = 1;
```

**Проверка на исходных данных:** 2400 и 1200: у заказа две позиции. `SUM(DISTINCT o.total)` не является общим исправлением: разные заказы могут иметь одинаковую сумму. Сначала агрегируйте на нужном уровне или используйте EXISTS.

### Последняя строка и ничья

```sql
SELECT DISTINCT ON (customer_id) customer_id, id, created_at
FROM orders WHERE status = 'paid'
ORDER BY customer_id, created_at DESC, id DESC;
```

**Проверка:** три клиента, для customer_id=1 выбран id=2. Дополнительный id делает выбор детерминированным при равных датах. `DISTINCT ON` - расширение PostgreSQL; для переносимого SQL используйте `row_number()` в подзапросе.

## 9. Разбор инцидента и итог

**Сценарий:** после релиза выросла задержка запросов, а число подключений приблизилось к пределу. Не начинайте с увеличения `max_connections`: больше соединений может увеличить конкуренцию за память и CPU. Сначала определите влияние и период, проверьте пул соединений, долгие и ожидающие транзакции, блокирующие сессии, планы и недавние изменения.

```sql
SELECT state, wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state, wait_event_type, wait_event
ORDER BY count(*) DESC;
```

Видимость других сессий зависит от прав; имена и текст запросов могут быть чувствительными. Проверяйте изменение метрик во времени, а не только один снимок. Завершение чужих сессий, изменение лимитов и параметров - отдельные эксплуатационные действия с последствиями.

**Итоговое задание:** подготовьте короткий runbook с симптомом, тремя гипотезами, проверками, безопасным уменьшением влияния и проверкой восстановления. Приложите план запроса до и после изменения, результаты нагрузочного эксперимента в отдельной среде и объяснение границ выводов. После практики закройте обе psql-сессии и удалите только созданные таблицы `lab_events`, `lab_inventory`, `lab_locks`, `lab_jobs`, `lab_versions`, если они больше не нужны; в них находятся учебные данные.

## Документация PostgreSQL 16

- [EXPLAIN и BUFFERS](https://www.postgresql.org/docs/16/using-explain.html), [статистика планировщика](https://www.postgresql.org/docs/16/planner-stats.html)
- [Индексы](https://www.postgresql.org/docs/16/indexes.html), [частичные индексы](https://www.postgresql.org/docs/16/indexes-partial.html), [index-only scans](https://www.postgresql.org/docs/16/indexes-index-only-scans.html)
- [Изоляция транзакций](https://www.postgresql.org/docs/16/transaction-iso.html), [блокировки и deadlocks](https://www.postgresql.org/docs/16/explicit-locking.html), [SELECT и SKIP LOCKED](https://www.postgresql.org/docs/16/sql-select.html)
- [VACUUM](https://www.postgresql.org/docs/16/routine-vacuuming.html), [статистика активности](https://www.postgresql.org/docs/16/monitoring-stats.html)
