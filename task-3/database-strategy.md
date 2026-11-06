# Database per Service
  
| Сервис                         | Тип базы данных                                                                           | Обоснование                                                                                                            |
| ------------------------------ | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Сервис ставок (Bidding)        | Нет своей БД. Использует только кеш Redis и локальный кеш кампаний, баланса и победителей | Hot path, SLA менее 80 мс. Своих постоянных данных нет                                                                 |
| Сервис кампаний (Campaign)     | PostgreSQL                                                                                | Транзакционные CRUD-операции: кампании, бюджеты, ставки, таргетинг. Нужен ACID                                         |
| Сервис статистики (Statistics) | ClickHouse                                                                                | Высокий поток на запись событий                                                                                        |
| Сервис аналитики (Analytics)   | PostgreSQL / Object Storage                                                               | Отчёты большие по объему памяти, поэтому Object Storage для тяжёлых файлов. PostgreSQL - для метаданных к этим отчетам |
| Сервис финансов (Financial)    | PostgreSQL                                                                                | Требуется ACID, соблюдение идемпотентности                                                                             |
  
## Financial Service: SQL-транзакция и Kafka offset

При обработке события показа (или списания) из Kafka Financial Service выполняет два связанных действия:

1. Обновляет баланс (возврат резерва или списание).
2. Фиксирует факт обработки (запись в таблицу processed_events или обновление статуса резерва).

Эти действия обязаны быть в одной SQL-транзакции.

### Фиксация Kafka offset после SQL commit

Консьюмер получает событие из Kafka, выполняет SQL-операции (баланс, `Transaction`, `processed_events`), делает SQL commit, и только потом фиксирует offset в Kafka.

Порядок SQL commit → offset commit обеспечивает safe redelivery:
- если сервис упал до commit offset, то при перезапуске событие перечитается заново
- если упал после commit SQL, то данные уже надёжно записаны, повторная обработка защитится unique constraint на `event_id`.

#### Сценарий: нормальная обработка

1. Получено событие из Kafka
2. Выполнены SQL-операции: изменение баланса, запись в `Transaction`, запись в `processed_events`
3. SQL commit
4. Offset commit
5. Результат: деньги изменены, транзакция создана, offset зафиксирован

#### Сценарий: дубль события

1. Событие пришло повторно (ретрай продюсера или повторная доставка)
2. INSERT в `Transaction(event_id)` вызывает `ON CONFLICT DO NOTHING`
3. Финансовая транзакция не создаётся
4. offset фиксируется корректно
5. Результат: баланс не изменился, повторная обработка не повредила данные

#### Сценарий: падение после commit SQL, до commit offset

1. Сервис упал после фиксации транзакции, но offset ещё не записан
2. При перезапуске событие перечитывается заново
3. INSERT в `Transaction(event_id)` игнорируется (`ON CONFLICT DO NOTHING`)
4. Баланс не изменяется
5. Offset фиксируется, обработка завершается
6. Результат: safe redelivery — дубль не создал двойное списание

## Transactional Outbox

При создании резерва Bidding Service должен ответить DSP в течение SLA < 80 мс. Событие `reserve_pending` нужно надёжно записать в Kafka для восстановления резервов.

Если Bidding Service отправляет bid-response DSP, но падает до того, как `reserve_pending` попадает в Kafka, то при восстановлении Redis из Kafka резерв будет потерян, а деньги заморожены в Redis, но не учтены в журнале. Outbox устраняет это. Событие записывается в таблицу `outbox` в той же SQL-транзакции, что и резерв. После commit транзакции Outbox-Consumer доставляет событие в Kafka.

### Таблица outbox_events

| Поле | Тип | Назначение |
| --- | --- | --- |
| `id` | uuid | PK |
| `aggregate_id` | text | requestId (аукцион) |
| `event_type` | text | reserve_pending / reserve_confirmed / reserve_released |
| `payload` | jsonb | Событие для `balanceTopic` |
| `status` | text | pending / published / failed |
| `created_at` | timestamp | Время создания |
| `published_at` | timestamp | Время доставки в Kafka |

### Запись и доставка

Financial Service при создании резерва выполняет две операции в одной SQL-транзакции: INSERT в `reservations` (status=`pending`) + INSERT в `outbox_events` (event_type=`reserve_pending`). Commit транзакции фиксирует оба действия.

Outbox consumer отдельный процесс в Financial Service, который каждые 200-300мс (т.к. это не hot path, то не влияет на общий SLA 80мс, и обеспечит баланс нагрузки на PostgreSQL) опрашивает `outbox_events` по статусу `pending`, батчами по 100 событий отправляет их в `balanceTopic` (продюсер с `acks=all`), и при успешной доставке обновляет статус на `published`. Если Outbox consumer упал после отправки и до обновления статуса, то событие будет доставлено повторно. At-least-once гарантия Kafka + идемпотентность консьюмеров (по `event_id`) защищают от дублей.

### Нормальный сценарий

1. DSP → bid-request → Bidding Service
2. Bidding Service резервирует баланс в Redis (Lua-скрипт)
3. Bidding Service → Financial Service (gRPC): `createReservation`
4. Financial Service: SQL-транзакция (reservations + outbox_events) → commit
5. Bidding Service отправляет bid-response DSP
6. Outbox consumer читает событие, отправляет в `balanceTopic`
7. Redis-консьюмер (`group-cache-campaigns`) обновляет `reserve:{requestId}`

### Частичный успех

Redis-резерв выделен, bid-response отправлен DSP, но Financial Service упал до commit SQL.

1. Redis: баланс уменьшен, `reserve:{requestId}` создан
2. DSP получил ставку
3. Financial Service упал до commit SQL, reservations не записан, outbox пуст
4. TTL резерва истёк. Financial Service возвращает баланс через `reserve_released`
5. Событие `reserve_released` записывается в Outbox → Kafka

Частичный успех обрабатывается TTL резерва в Redis. Поскольку `reserve_pending` не попало в Outbox, Outbox consumer ничего не отправит. Таймер обнаружит резерв без записи в БД и вернёт баланс.

### Outbox consumer упал после publish

1. Outbox consumer отправил событие в Kafka
2. Перед обновлением статуса на `published` процесс упал
3. При перезапуске событие прочитается повторно
4. Kafka-продюсер отправит дубль в `balanceTopic`
5. Консьюмер Redis получает дубль, но игнорирует по `event_id`
6. Результат: at-least-once доставка, идемпотентность защищает от дублей

## Таблица blocked_balances

При рассогласовании Redis и PostgreSQL (например, RedisReplayer обнаружил резерв в Redis, которого нет в БД) разница блокируется в таблице `blocked_balances` Financial PostgreSQL.

| Поле | Тип | Назначение |
| --- | --- | --- |
| `advertiser_id` | text | Рекламодатель |
| `amount` | numeric | Заблокированная сумма |
| `reason` | text | reserve_pending_drift / sync_in_progress |
| `created_at` | timestamp | Момент блокировки |
| `released_at` | timestamp | Момент разблокировки |

Пока `blocked_balance > 0`, Financial Service не подтверждает резервы для новых аукционов рекламодателя. Блокировка снимается после завершения восстановления согласованности.
