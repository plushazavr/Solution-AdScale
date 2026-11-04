# Проектирование сервиса ставок (Bidding Service)

## Границы сервиса

Отвечает за приём запросов от Ad Server, взвешивание ставок, применение бизнес-правил, выбор победителя, запись победителя в кеш.

## API

1. healthcheck
2. Провести аукцион.
**In-параметры:** requestId, userId, candidates\[] (campaignId, ставка, таргетинг), context (geo, device)
**Out-параметры:** winner (campaignId, creativeId, price), reason (если нет победителя).

## Модель данных

### Модель "кампания"

```javascript
Campaign {
  campaignId //идентификатор кампании
  advertiserId //идентификатор рекламодателя
  bid          //ставка
  targeting    //таргетинг (geo, device)
  ...
}
```

### Модель "баланс"

```javascript
Balance {
  advertiserId //идентификатор рекламодателя
  availableAmount //доступный бюджет
  ...
}
```

### Модель «победитель»

```javascript
Winner {
  requestId //идентификатор запроса
  campaignId //идентификатор кампании
  price //цена
  advertiserId //идентификатор рекламодателя
  ...
}
```

### Модель "резервирование баланса"

Резерв не снимается автоматически по TTL. Освобождение привязано к событию показа или отсутствия показа из Kafka 

```javascript
Reservation {
  requestId // ключ резерва (коррелирует с событием показа)
  advertiserId // идентификатор рекламодателя
  amount // зарезервированная сумма
  auctionId // идентификатор аукциона для связки с событием
  createdAt // момент резервирования
  status // pending, confirmed, released
}
```

### Модель "транзакция" 
Используется для факта списания денежных средств с баланса

```javascript
Transaction {
  transactionId // PK, ключ списания
  advertiserId //идентификатор рекламодателя
  amount // денежное изменение (−списание / +возврат)
  eventId // UNIQUE — ключ идемпотентности
  createdAt
}
```
