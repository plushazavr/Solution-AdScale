# Проектирование взаимодействия

## Внутреннее взаимодействие

| Интеграция                               | Протокол       |
| ---------------------------------------- | -------------- |
| DSP-партнёр → B2B Gateway                | HTTPS/OpenRTB  |
| Пользователь → B2C Gateway               | HTTPS/REST     |
| Рекламодатель → Dashboard Gateway        | HTTPS/REST     |
| B2B Gateway → Ad Server                  | gRPC           |
| B2C Gateway → Delivery                   | gRPC           |
| Dashboard Gateway → Advertiser Dashboard | HTTPS/REST     |
| Ad Server → Bidding                      | gRPC           |
| Bidding → Delivery                       | gRPC           |
| Bidding → Redis                          | Redis protocol |
| Delivery → Redis                         | Redis protocol |
| Advertiser Dashboard → Campaign          | HTTPS/REST     |
| Advertiser Dashboard → Payment           | HTTPS/REST     |
| Payment → Платёжный шлюз                 | HTTPS/REST     |
| Campaign → Kafka                         | Kafka          |
| Payment/Financial → Kafka                | Kafka          |
| Delivery → Kafka                         | Kafka          |
| Statistics ← Kafka                       | Kafka          |
| Analytics ← Kafka                        | Kafka          |
| Analytics → reportDb                     | SQL            |

## Внешняя интеграция с DSP

Протокол HTTPS + OpenRTB. Синхронный request/response — DSP присылает bid-request, AdScale отвечает bid-response.
Почему: OpenRTB — отраслевой стандарт RTB, который навязан партнёром. HTTPS — обязателен для безопасности.

## Потоковая обработка событий

Kafka выбрана для событий (клики, показы, платежи, изменения кампаний и балансов), потому что:
1. события читают несколько сервисов сразу (Statistics, Financial, Analytics)
2. если сервис упал, он дочитает события с того места, где остановился (replay)3. Kafka держит большой поток сообщений (требование  15 000 RPS в пике на аукцион)
4. события хранятся — можно разобрать инцидент или провести аудит

RabbitMQ и Artemis не подходят для этого потока: они не умеют перечитывать события заново и не держат такую нагрузку. Они хороши для очередей задач (СМС, письма, пуши), но таких задач у AdScale нет.

## Почему gRPC для Bidding, а REST для Campaign

1. Bidding gRPC. Быстрее REST. Строгий контракт, обе стороны заранее знают формат, меньше ошибок. Есть таймауты — если ответ не пришёл за 80 мс, вызов прерывается.

2. Campaign REST. Не требуется скорость, т.к. рекламодатель создаёт кампанию не 15 000 раз в секунду.Легко поддерживается через браузер

## API Gateway

Для hot path (B2B и B2C) необходимо выбрать Envoy, т.к. он самый быстрый, поддерживает gRPC и HTTP, бесплатный. Если команда не готова разбираться с ним, то NGINX + Lua как более простой вариант. Spring Cloud Gateway не подходит: он Java, а стек AdScale другой. Облачный API Gateway не подходит: платный при 50k RPS и завязан на облако, а у AdScale свой кластер.