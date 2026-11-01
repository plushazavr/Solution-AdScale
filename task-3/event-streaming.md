# Потоковая обработка данных

## Топики Kafka
| Топик           | Продюсер             | Консьюмеры (сервисы)                                     | Событие                          | Хранение |
| --------------- | -------------------- | -------------------------------------------------------- | -------------------------------- | -------- |
| campaignTopic   | Campaign Service     | Redis                                                    | CRUD кампаний, изменения бюджета | 7 дней   |
| clickTopic      | Delivery Service     | Statistics Service, Financial Service, Analytics Service | Клики, показы                    | 10 дней  |
| paymentTopic    | Payment Service      | Financial Service, Statistics Service                    | Факты успешных платежей          | 30 дней  |
| balanceTopic    | Financial Service    | Redis                                                    | Изменения баланса                | 30 дней  |
| reqReportTopic  | Advertiser Dashboard | Analytics Service                                        | Заказы отчётов                   | 7 дней   |
| respReportTopic | Analytics Service    | Advertiser Dashboard                                     | Готовность отчётов               | 7 дней   |

## Схема событий
Avro + Schema Registry. Поля: event_id, event_type, timestamp, correlation_id, business_data.
At-least-once для всех топиков.
Идемпотентность по event_id.


## Политика хранения
retention.ms по времени + retention.bytes как страховка от разрастания.
