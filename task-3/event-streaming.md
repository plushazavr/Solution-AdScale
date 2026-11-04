# Потоковая обработка данных
 
## Топики Kafka
| Топик             | Продюсер             | Консьюмеры (сервисы)                                     | Событие / Avro-схема                                   | Хранение |
| --------------- | -------------------- | -------------------------------------------------------- | ------------------------------------------------------ | -------- |
| campaignTopic   | Campaign Service     | Redis                                                    | `CampaignChangedData`                                  | 7 дней   |
| clickTopic      | Delivery Service     | Statistics Service, Financial Service, Analytics Service | `ClickBusinessData`                                    | 10 дней  |
| impressionTopic | Delivery Service     | Statistics Service, Financial Service, Analytics Service | `ImpressionBusinessData` (см. Avro-схему ниже)         | 10 дней  |
| paymentTopic    | Payment Service      | Financial Service, Statistics Service                    | `PaymentBusinessData`                                  | 30 дней  |
| balanceTopic    | Financial Service    | Redis                                                    | `BalanceChangedData`                                   | 30 дней  |
| reqReportTopic  | Advertiser Dashboard | Analytics Service                                        | Заказы отчётов                                         | 7 дней   |
| respReportTopic | Analytics Service    | Advertiser Dashboard                                     | Готовность отчётов                                     | 7 дней   |
 
## Группы потребителей
 
| Группа | Читает | Пишет |
|---|---|---|
| `group-stats` | clickTopic | ClickHouse (Statistics) |
| `group-financial` | clickTopic, paymentTopic | PostgreSQL (Financial) |
| `group-cache-campaigns` | campaignTopic, balanceTopic | Redis |
| `group-analytics` | reqReportTopic, clickTopic | PostgreSQL (Analytics) |
| `group-report-notify` | respReportTopic | Advertiser Dashboard |
 
## Схема событий
Avro + Schema Registry. At-least-once для всех топиков. Идемпотентность по `event_id`.
 
### Типизированная Avro-схема (impression-event)
 
```json
{
  "namespace": "namespacecom.adscale.events",
  "type": "record",
  "name": "ImpressionEvent",
  "doc": "Событие подтверждения показа рекламного баннера",
  "fields": [
    {
      "name": "event_id",
      "type": "string",
      "doc": "UUID, уникальный идентификатор события (ключ идемпотентности)"
    },
    {
      "name": "event_type",
      "type": "string",
      "default": "impression",
      "doc": "Тип события"
    },
    {
      "name": "timestamp",
      "type": "long",
      "doc": "timestamp"
    },
    {
      "name": "correlation_id",
      "type": "string",
      "doc": "Сквозной requestId из bid-request"
    },
    {
      "name": "business_data",
      "type": {
        "type": "record",
        "name": "ImpressionBusinessData",
        "doc": "Полезная нагрузка события показа",
        "fields": [
          {
            "name": "requestId",
            "type": "string",
            "doc": "Идентификатор bid-request (коррелирует с аукционом)"
          },
          {
            "name": "auctionId",
            "type": "string",
            "doc": "Идентификатор аукциона для связки с резервом"
          },
          {
            "name": "advertiserId",
            "type": "string",
            "doc": "Идентификатор рекламодателя"
          },
          {
            "name": "campaignId",
            "type": "string",
            "doc": "Идентификатор кампании"
          },
          {
            "name": "bannerId",
            "type": "string",
            "doc": "Идентификатор баннера"
          },
          {
            "name": "bidPrice",
            "type": "double",
            "doc": "Ставка по аукциону"
          },
          {
            "name": "winPrice",
            "type": "double",
            "doc": "Фактическая цена победителя"
          },
          {
            "name": "dspPartnerId",
            "type": "string",
            "doc": "Идентификатор DSP-партнёра"
          },
          {
            "name": "placeId",
            "type": "string",
            "doc": "Идентификатор размещения баннера"
          },
          {
            "name": "deviceType",
            "type": ["null", "string"],
            "default": null,
            "enum": ["mobile", "desktop", "tablet"],
            "doc": "Тип устройства"
          },
          {
            "name": "geo",
            "type": [
              "null",
              {
                "type": "record",
                "name": "Geo",
                "fields": [
                  { "name": "country", "type": "string" },
                  { "name": "region", "type": ["null", "string"] },
                  { "name": "city", "type": ["null", "string"] }
                ]
              }
            ],
            "default": null,
            "doc": "Геолокация"
          },
          {
            "name": "reservationStatus",
            "type": "string",
            "default": "confirmed",
            "enum": ["confirmed", "rejected"],
            "doc": "Статус резерва: confirmed = показ прошёл, rejected = таймаут"
          }
        ]
      }
    }
  ]
}
```
 ### Заполненный пример события (impression)
 
Событие, записанное в Kafka-топик `impressionTopic` (консьюмеры: `group-stats`, `group-financial`, `group-analytics`):
 
```json
{
  "event_id": "a3f8b2c1-4e7d-4a9b-8c1f-2d6e5b0a9f3c",
  "event_type": "impression",
  "timestamp": 1698765432100,
  "correlation_id": "req-7f3a8b2c-1d4e-4a9b-8c1f-2d6e5b0a9f3c",
  "business_data": {
    "requestId":        "req-7f3a8b2c-1d4e-4a9b-8c1f-2d6e5b0a9f3c",
    "auctionId":        "auc-42",
    "advertiserId":     "adv-10042",
    "campaignId":       "cmp-season-sale-2026",
    "bannerId":       "cre-banner-300x250-v3",
    "bidPrice":         2.50,
    "winPrice":         1.85,
    "dspPartnerId":     "dsp-mega-network",
    "placeId":      "site-main-header-728x90",
    "deviceType":       "mobile",
    "geo": {
      "country": "RU",
      "region":  "MOW",
      "city":    "Moscow"
    },
    "confirmedAt":      1698765432145,
    "reservationStatus": "confirmed"
  }
}
```