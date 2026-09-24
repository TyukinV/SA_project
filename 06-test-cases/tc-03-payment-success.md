# TC-03: Успешная оплата

| **Атрибут** | **Значение** |
| :--- | :--- |
| **ID** | TC-03 |
| **Название** | Успешная оплата |
| **Связанный UC** | [UC-03](../02-functional-requirements/use-case-specifications.md#uc-03-оплатить-бронь) |
| **Связанные AC** | AC-03.1, AC-03.2 |
| **Связанные FR** | FR-03 |
| **Связанные NFR** | NFR-02 (ответ шлюза < 3 сек, timeout 5 сек), NFR-04 (безопасность) |
| **Связанный документ** | [payment-gateway-integration.md](../04-api-and-integrations/payment-gateway-integration.md) |
| **Тип** | Интеграционный, позитивный + негативный |
| **Автор** | Тюкин Вадим |
| **Версия** | 1.0 |
| **Дата** | 12.08.2026 |

---

## 1. Назначение

Проверить полный цикл онлайн-оплаты бронирования: **создание платёжной сессии → оплата на стороне шлюза → получение вебхука → подтверждение брони**.

**Цель теста:** убедиться, что система корректно интегрирована с внешним платёжным шлюзом, безопасно обрабатывает асинхронные вебхуки, обновляет статусы брони и платежа в течение 5 секунд (AC-03.1) и уведомляет гостя по email в течение 1 минуты (AC-03.2).

**Критичность:** без успешной оплаты бронь остаётся в `PENDING` и истекает через 15 минут. Это ключевой шаг монетизации.

---

## 2. Предусловия

| Условие | Значение |
| :--- | :--- |
| Бронь | №123 создана со статусом `PENDING` |
| Номер | №101 заблокирован в `inventory_calendar` на 10–13 августа |
| `expires_at` | `now() + 15 минут` |
| Платёжный шлюз | Доступен, sandbox-режим |
| HMAC-секрет | Настроен, используется для проверки вебхуков |
| Email-провайдер | Доступен (SMTP) |
| Backend | Доступен, БД подключена |

---

## 3. Сценарии

### Сценарий 1: Создание платёжной сессии

```gherkin
Функция: Создание платежа
  Сценарий: HBS создаёт платёжную сессию во внешнем шлюзе
    Дано бронь №123 существует со статусом "PENDING"
    И сумма к оплате = 1500000 копеек (15000.00 RUB)
    Когда фронтенд отправляет POST /api/bookings/123/payment-link
    Тогда backend отправляет в шлюз POST /v1/payments:
      | Поле        | Значение                                        |
      | amount      | 1500000                                         |
      | currency    | RUB                                             |
      | description | "Бронирование отеля HBS-123 (заезд 2026-08-10)" |
      | return_url  | "https://hotel-booking.example.com/payment/result" |
      | webhook_url | "https://hotel-booking.example.com/webhook/payment" |
      | metadata    | { "booking_id": "123" }                         |
    И передаёт заголовок Idempotency-Key = booking_id
    Когда шлюз отвечает успешно
    Тогда backend получает:
      | Поле           | Значение                                  |
      | transaction_id | "tr_abc123xyz"                            |
      | payment_url    | "https://payment-gateway.example.com/pay/tr_abc123xyz" |
      | status         | "pending"                                 |
    И сохраняет платёж в БД:
      | Поле                   | Значение    |
      | booking_id             | 123         |
      | gateway_transaction_id | tr_abc123xyz|
      | amount                 | 1500000     |
      | status                 | PENDING     |
    И возвращает фронтенду 200 OK с payment_url
    И фронтенд перенаправляет гостя на страницу шлюза
```

---

### Сценарий 2: Идемпотентность при создании платежа

```gherkin
Функция: Идемпотентность платёжной сессии
  Сценарий: Повторный вызов payment-link возвращает ту же сессию
    Дано для брони №123 уже создана платёжная сессия
    И она активна (не истекла, не завершена)
    Когда фронтенд повторно отправляет POST /api/bookings/123/payment-link
    Тогда backend НЕ создаёт новую сессию в шлюзе
    И возвращает ту же payment_url = "https://payment-gateway.example.com/pay/tr_abc123xyz"
    И в таблице payments остаётся ровно одна активная запись для брони №123
```

---

### Сценарий 3: Успешная оплата через вебхук

```gherkin
Функция: Обработка вебхука
  Сценарий: Гость успешно оплачивает бронь
    Дано гость ввёл данные карты на стороне платёжного шлюза
    И шлюз успешно обработал транзакцию
    Когда шлюз отправляет POST /webhook/payment:
      | Поле           | Значение           |
      | transaction_id | tr_abc123xyz       |
      | status         | succeeded          |
      | amount         | 1500000            |
      | currency       | RUB                |
      | paid_at        | 2026-08-06T10:17:45Z |
      | metadata       | { "booking_id": "123" } |
      | signature      | sha256=valid_hash  |
      | timestamp      | 1638795456         |
    Тогда backend проверяет подпись HMAC-SHA256
    И проверяет актуальность timestamp (не старше 5 минут)
    И находит платёж по transaction_id = "tr_abc123xyz"
    И обновляет payments:
      | Поле   | Было    | Стало     |
      | status | PENDING | COMPLETED |
    И переводит бронь в статус "CONFIRMED" в течение 5 секунд (AC-03.1)
    И сохраняет блокировку календаря (is_available = false, booking_id = 123)
    И записывает событие в booking_events:
      | old_status | new_status | changed_by | comment         |
      | PENDING    | CONFIRMED  | system     | "Webhook tr_..."|
    И отправляет гостю email с деталями заезда в течение 1 минуты (AC-03.2)
    И возвращает шлюзу 200 OK
```

---

### Сценарий 4: Возврат гостя на сайт после оплаты

```gherkin
Функция: Возврат гостя
  Сценарий: Гость возвращается на сайт после оплаты
    Дано гость завершил оплату на стороне шлюза
    Когда шлюз перенаправляет гостя на return_url
    И фронтенд отправляет GET /api/bookings/123/status
    Тогда backend возвращает:
      | Поле   | Значение  |
      | status | CONFIRMED |
    И гость видит сообщение: "Бронь подтверждена! Детали отправлены на email."
```

---

### Сценарий 5: Гонка вебхука и возврата гостя

```gherkin
Функция: Синхронизация состояния
  Сценарий: Вебхук приходит после возврата гостя
    Дано гость вернулся на сайт раньше, чем пришёл вебхук
    Когда гость запрашивает GET /api/bookings/123/status
    Тогда backend возвращает status = "PENDING"
    И фронтенд показывает: "Ожидаем подтверждение оплаты..."
    Когда приходит вебхук со статусом "succeeded"
    Тогда backend обновляет бронь на "CONFIRMED"
    И фронтенд при следующем polling (каждые 3 сек) показывает "Подтверждено"
```

---

### Сценарий 6: Повторный вебхук (идемпотентность)

```gherkin
Функция: Идемпотентность вебхука
  Сценарий: Шлюз повторяет доставку вебхука
    Дано вебхук с transaction_id = "tr_abc123xyz" уже обработан
    И платёж имеет статус "COMPLETED"
    И бронь имеет статус "CONFIRMED"
    Когда шлюз повторно отправляет тот же вебхук
    Тогда backend находит платёж по transaction_id
    И НЕ меняет статус платежа
    И НЕ меняет статус брони
    И НЕ создаёт дубликат события в booking_events
    И возвращает 200 OK (идемпотентность)
```

---

### Сценарий 7: Невалидная подпись вебхука

```gherkin
Функция: Защита от подделки
  Сценарий: Вебхук с невалидной подписью
    Дано backend получает POST /webhook/payment
    И signature = "sha256=invalid_hash"
    Когда backend проверяет подпись
    Тогда подпись НЕ валидна
    И backend логирует инцидент (warning: invalid_signature)
    И возвращает 200 OK (не раскрывает информацию злоумышленнику)
    И НЕ обновляет статус платежа
    И НЕ меняет статус брони
```

---

### Сценарий 8: Устаревший timestamp (защита от replay)

```gherkin
Функция: Защита от replay-атак
  Сценарий: Вебхук с устаревшим timestamp
    Дано backend получает вебхук с timestamp = now() - 10 минут
    И подпись формально валидна
    Когда backend проверяет timestamp
    Тогда timestamp старше 5 минут
    И backend отклоняет вебхук
    И логирует инцидент (warning: stale_webhook)
    И возвращает 200 OK
    И НЕ обновляет статус платежа
```

---

### Сценарий 9: Временная ошибка БД при обработке вебхука

```gherkin
Функция: Отказоустойчивость
  Сценарий: БД недоступна во время обработки вебхука
    Дано backend получает валидный вебхук
    И при попытке обновить payments происходит ошибка соединения с БД
    Когда backend обрабатывает вебхук
    Тогда backend возвращает шлюзу 503 Service Unavailable
    И шлюз повторит доставку вебхука через некоторое время
    И при повторной доставке (когда БД снова доступна) вебхук обрабатывается успешно
```

---

### Сценарий 10: Fallback-опрос при потере вебхука

```gherkin
Функция: Резервный канал получения статуса
  Сценарий: Вебхук не пришёл, система сама опрашивает шлюз
    Дано платёж создан 16 минут назад
    И статус платежа остаётся "PENDING"
    И вебхук от шлюза не приходил
    Когда Scheduler запускает задачу "Fallback Polling"
    Тогда backend отправляет GET /v1/payments/tr_abc123xyz
    И шлюз отвечает:
      | Поле           | Значение           |
      | transaction_id | tr_abc123xyz       |
      | status         | succeeded          |
      | paid_at        | 2026-08-06T10:17:45Z |
    Тогда backend обновляет платёж на "COMPLETED"
    И переводит бронь в "CONFIRMED"
    И отправляет гостю email
```

---

## 4. Постусловия

| Состояние | Значение |
| :--- | :--- |
| `payments` | Запись со статусом `COMPLETED`, `gateway_transaction_id` заполнен |
| `bookings` | Статус `CONFIRMED` |
| `inventory_calendar` | Блокировка сохранена (`is_available = false`, `booking_id = 123`) |
| `booking_events` | Запись `PENDING → CONFIRMED`, `changed_by = system` |
| `notifications` | Запись со статусом `SENT`, канал `email` |
| Гость | Видит статус «Подтверждено» на сайте, получил email |
| Scheduler | Не находит бронь для отмены (status != `PENDING`) |

---

## 5. Тестовые данные

### Параметры платежа

| Параметр | Значение |
| :--- | :--- |
| booking_id | 123 |
| amount | 1500000 (15000.00 RUB) |
| currency | RUB |
| transaction_id | tr_abc123xyz |
| payment_url | https://payment-gateway.example.com/pay/tr_abc123xyz |
| Idempotency-Key | 550e8400-e29b-41d4-a716-446655440000 |

### Валидный вебхук

```json
{
  "transaction_id": "tr_abc123xyz",
  "status": "succeeded",
  "amount": 1500000,
  "currency": "RUB",
  "paid_at": "2026-08-06T10:17:45Z",
  "metadata": { "booking_id": "123" },
  "signature": "sha256=valid_hash",
  "timestamp": 1638795456
}
```

### Невалидный вебхук (подпись)

```json
{
  "transaction_id": "tr_abc123xyz",
  "status": "succeeded",
  "amount": 1500000,
  "currency": "RUB",
  "metadata": { "booking_id": "123" },
  "signature": "sha256=invalid_hash",
  "timestamp": 1638795456
}
```

### Ожидаемые записи в БД

**`payments`**

```
payment_id             = 999
booking_id             = 123
amount                 = 1500000
status                 = COMPLETED
payment_method         = "card"
gateway_transaction_id = tr_abc123xyz
created_at             = 2026-08-06T10:15:00Z
updated_at             = 2026-08-06T10:17:46Z
```

**`bookings`**

```
booking_id = 123
status     = CONFIRMED
```

**`booking_events`**

```
| booking_id | old_status | new_status | changed_by | comment                    |
|------------|------------|------------|------------|----------------------------|
| 123        | PENDING    | CONFIRMED  | system     | Webhook tr_abc123xyz       |
```

---

## 6. Метрики для мониторинга

| Метрика | Описание | Целевое значение |
| :--- | :--- | :--- |
| `payment_webhook_received_total` | Количество полученных вебхуков | — |
| `payment_webhook_delay_seconds` | Задержка между оплатой и обработкой | p95 < 5 сек |
| `payment_processing_duration_seconds` | Время обработки вебхука | p95 < 1 сек |
| `webhook_signature_invalid_total` | Невалидные подписи | **0** |
| `webhook_replay_attempts_total` | Replay-атаки | **0** |
| `payment_fallback_polling_total` | Fallback-опросы | — (информативно) |

**Алерт:** если `payment_webhook_delay_seconds > 60` — расследовать (возможен сбой доставки вебхуков).

---

## 7. Критерии приёмки (Definition of Done)

- [ ] Все 10 сценариев проходят на тестовом стенде со sandbox-шлюзом.
- [ ] AC-03.1 подтверждён нагрузочным тестом: 100 вебхуков → все обрабатываются < 5 сек.
- [ ] AC-03.2 подтверждён: email отправлен < 1 мин после вебхука.
- [ ] Проверено, что повторный вызов `payment-link` возвращает ту же `payment_url`.
- [ ] Проверено, что невалидная подпись не меняет статусы.
- [ ] Проверено, что устаревший `timestamp` (> 5 мин) отклоняется.
- [ ] Проверено, что fallback-опрос подхватывает «зависшие» платежи.
- [ ] Логи содержат `payment_completed` с полями `booking_id`, `transaction_id`, `amount`.
- [ ] Покрытие автотестами ≥ 85% для кода обработки вебхуков.

---

## 8. Связанные артефакты

| Артефакт | Ссылка |
| :--- | :--- |
| Use Case UC-03 | [use-case-specifications.md#uc-03](../02-functional-requirements/use-case-specifications.md#uc-03-оплатить-бронь) |
| Acceptance Criteria AC-03.1, AC-03.2 | [srs.md, разд. 7](../02-functional-requirements/srs.md) |
| NFR-02, NFR-04 | [srs.md, разд. 4](../02-functional-requirements/srs.md) |
| Интеграция с платёжным шлюзом | [payment-gateway-integration.md](../04-api-and-integrations/payment-gateway-integration.md) |
| OpenAPI `POST /bookings/{id}/payment-link` | [openapi.yaml](../04-api-and-integrations/openapi.yaml) |
| Sequence Diagram (оплата) | [sequence-payment-webhook.png](../03-diagrams/uml/sequence-payment-webhook.png) |
| ER-диаграмма | [er-diagram.md](../03-diagrams/erd/er-diagram.md) |
| BPMN | [booking-and-payment.svg](../03-diagrams/bpmn/booking-and-payment.svg) |

---

## 9. Примечания

- Сценарии **1–4** — позитивные, покрывают основной поток UC-03.
- Сценарий **5** — граничный случай: гонка вебхука и возврата гостя.
- Сценарии **6–8** — безопасность: идемпотентность, защита от подделки, replay-защита.
- Сценарий **9** — отказоустойчивость при сбое БД.
- Сценарий **10** — fallback-опрос, ключевой для NFR-06.
- Отказ платежа (недостаточно средств) не входит в этот тест — он покрывается отдельно в негативных ветках TC-05 при расширении.
- Таймаут `PENDING`-брони проверяется в [TC-04](./tc-04-payment-timeout.md).
