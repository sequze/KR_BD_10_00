# 1
До изменений:
- Используется Seq Scan
- Не помогают индексы:
```sql
CREATE INDEX idx_exam_events_status ON exam_events (status);
CREATE INDEX idx_exam_events_amount_hash ON exam_events USING hash (amount);
```
Не помогают эти индексы, тк они покрывают другие столбцы, планировщик выбирает такой план потому что нет других индексов
Индекс:
```sql
CREATE INDEX idx_exam_events ON exam_events (user_id, created_at);
```
Теперь в плане используется Index Scan по обоим столбцам
Выполнять ANALYZE не нужно, планировщик будет учитывать индекс автоматически

# 2

```sql
SELECT u.id, u.country, o.amount, o.created_at
FROM exam_users u
JOIN exam_orders o ON o.user_id = u.id
WHERE u.country = 'JP'
  AND o.created_at >= TIMESTAMP '2025-03-01 00:00:00'
  AND o.created_at < TIMESTAMP '2025-03-08 00:00:00';
```

Используется Hash Join. Он выбран тк мы соединяем по exam_users.id у которого есть индекс
Не помогает этот индекс:
```sql
CREATE INDEX idx_exam_users_name ON exam_users (name);
```
Индекс по created_at помогает, внутрти используется Bitmap Index Scan по условиям с created_at
```sql
CREATE INDEX idx_exam_orders_created_at ON exam_orders (created_at);
```
Для ускорения запроса можно добавить индекс по exam_users.country:
```sql
CREATE INDEX exam_users_country_idx ON exam_users (country);
```
cost упал с 558 до 333
Улучшился за счет Bitmap Index Scan по новому индексу
shared hit - данные уже были в shared buffers (кэш), shared read - чтение с диска

# 3

После UPDATE:

Было: xmin = 814, ctid = (0,4), qty = 15
Стало: xmin = 816, ctid = (0,5), qty = 20


- появился новый tuple (новый ctid)
- xmin стал равен XID транзакции обновления (816) 
- старая версия строки больше не видна (у неё выставлен xmax, но в SELECT она не показывается)

После DELETE строка с id = 2 исчезла из SELECT, потому что ей выставили xmax (она помечена как удалённая) и она стала невидимой для текущих транзакций. Физически строка ещё может лежать в таблице до VACUUM.
Сравнение:

- VACUUM:
  - очищает мёртвые версии
  - не сжимает файл
- autovacuum:
  - делает VACUUM автоматически

- VACUUM FULL:
  - переписывает таблицу
  - реально уменьшает размер на диске

Vacuum Full - полная блокировка таблицы

# 4

1. В обоих сессиях транзакция 2 блокируется до выполнения транзакции 1.
2. FOR SHARE - блокирует строки на изменение, но сам использует только для чтения. FOR UPDATE -  
3. Потому что он использует другой блокировку ACCESS SHARE, которая конкурирует только с ACCESS EXCLUSIVE.
4. Когда нужно консистентно изменить строки, для которых мы делаем SELECT

