# Проектирование взаимодействия
 
## Внутреннее взаимодействие
 
| Интеграция | Протокол |
|---|---|
| DSP-партнёр → B2B Gateway | HTTPS/OpenRTB |
| Пользователь → B2C Gateway | HTTPS/REST |
| Рекламодатель → Dashboard Gateway | HTTPS/REST |
| B2B Gateway → Ad Server | gRPC |
| B2C Gateway → Delivery | gRPC |
| Dashboard Gateway → Advertiser Dashboard | HTTPS/REST |
| Ad Server → Bidding | gRPC |
| Bidding → Delivery | gRPC |
| Bidding → Redis | Redis protocol |
| Delivery → Redis | Redis protocol |
| Advertiser Dashboard → Campaign | HTTPS/REST |
| Advertiser Dashboard → Payment | HTTPS/REST |
| Payment → Платёжный шлюз | HTTPS/REST |
| Campaign → Kafka | Kafka |
| Payment/Financial → Kafka | Kafka |
| Delivery → Kafka | Kafka |
| Statistics ← Kafka | Kafka |
| Analytics ← Kafka | Kafka |
| Analytics → reportDb | SQL |
 
## Внешняя интеграция с DSP
 
Протокол HTTPS + OpenRTB. Синхронный request/response — DSP присылает bid-request, AdScale отвечает bid-response.
 
Почему: OpenRTB — отраслевой стандарт RTB, который навязан партнёром. HTTPS — обязателен для безопасности.
 
## Потоковая обработка событий
 
Kafka выбрана для событий (клики, показы, платежи, изменения кампаний и балансов), потому что:
1. события читают несколько сервисов сразу (Statistics, Financial, Analytics)
2. если сервис упал, он дочитает события с того места, где остановился (replay)
3. Kafka держит большой поток сообщений (требование 15 000 RPS в пике на аукцион)
4. события хранятся — можно разобрать инцидент или провести аудит
 
RabbitMQ и Artemis не подходят для этого потока: они не умеют перечитывать события заново и не держат такую нагрузку. Они хороши для очередей задач (СМС, письма, пуши), но таких задач у AdScale нет.
 
## Почему gRPC для Bidding, а REST для Campaign
 
1. **Bidding - gRPC.** Быстрее REST. Строгий контракт, обе стороны заранее знают формат, меньше ошибок. Есть таймауты - если ответ не пришёл за 80 мс, вызов прерывается
2. **Campaign -REST.** Не требуется скорость, т.к. рекламодатель создаёт кампанию не 15 000 раз в секунду. Легко поддерживается через браузер
 
## Sequence основного потока RTB — бюджет 80 мс P95 (проектная оценка hot path — 20 мс)
 
Последовательность вызовов в рамках одного аукционного запроса — в отдельном файле `task-2/diagrams/rtb-sequence.puml`. Общий бюджет — 80 мс P95 на всю цепочку от DSP до ответа. Ретраи (100–200–400 мс) на этом пути отсутствуют — они применяются только к НЕ hot-path
 
### Распределение бюджета 80 мс
 
| Шаг | Операция | Бюджет |
|---|---|---|
| 1 | DSP → B2B Gateway (принятие запроса, TLS handshake в кэше) | < 2 мс |
| 2 | B2B Gateway → Ad Server (gRPC) | < 2 мс |
| 3 | Ad Server → Bidding (gRPC) | < 2 мс |
| 4 | Bidding → Redis (данные кампании, кеш) | < 1 мс |
| 5 | Bidding → Redis (резервирование баланса) | < 2 мс |
| 6 | Bidding → Redis (запись победителя) | < 1 мс |
| 7 | **Bidding → Auction Engine (расчёт весов, win-rate, определение победителя)** | **< 5 мс** |
| 8 | Bidding → Ad Server (победитель+цена) | < 5 мс |
| 9 | Ad Server → B2B Gateway | < 2 мс |
| 10 | B2B Gateway → DSP | < 2 мс |
| **Сумма шагов** | **Суммарный hot path (P95)** | **≤ 24 мс** |
| **Запас** | **Очереди, сериализация, межсерверная задержка, P95-разброс** | **до 80 мс** |
 
**Примечание:** Сумма шагов по hot path примерно 24 мс. Проекционная оценка внутреннего hot path 20 мс P95. 20 мс это целевой показатель при отсутствии сетевых задержек и с кэшированным TLS. Запас до SLA 80 мс  на очередь, сериализацию и P95-разброс при реальной нагрузке 18 000 RPS.