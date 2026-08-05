# ER-диаграмма базы данных — Hotel Booking System (HBS)

## Назначение документа

Документ описывает логическую модель данных системы бронирования отелей на уровне MVP. Диаграмма построена на основе решений, зафиксированных в ТЗ и ЧТЗ: разделение физического статуса номера (`rooms.physical_status`) и календаря занятости (`inventory_calendar`), Enum-статусы бронирования, аудит через `booking_events`, отдельная таблица платежей.

Нотация: Mermaid ER Diagram (совместима с нативным рендерингом GitHub — открывается прямо в репозитории без плагинов).

---

## Диаграмма

```mermaid
erDiagram
    GUESTS ||--o{ BOOKINGS : "оформляет"
    ROOM_TYPES ||--o{ ROOMS : "определяет тип"
    ROOMS ||--o{ BOOKINGS : "бронируется в рамках"
    ROOMS ||--o{ INVENTORY_CALENDAR : "имеет календарь"
    BOOKINGS ||--o{ INVENTORY_CALENDAR : "блокирует даты"
    BOOKINGS ||--o{ BOOKING_EVENTS : "имеет историю статусов"
    BOOKINGS ||--o{ PAYMENTS : "оплачивается через"
    USERS ||--o{ BOOKING_EVENTS : "инициирует изменение"
    USERS ||--o{ ROOMS : "обновляет физ. статус"

    GUESTS {
        bigint id PK
        varchar full_name
        varchar email UK "уникален, используется для связи с бронью"
        varchar phone
        timestamp created_at
    }

    ROOM_TYPES {
        bigint id PK
        varchar name "Standard, Deluxe, Suite"
        int capacity "макс. число гостей"
        decimal base_price "базовая цена за ночь"
        text description
    }

    ROOMS {
        bigint id PK
        varchar room_number UK
        bigint room_type_id FK
        int floor
        varchar physical_status "AVAILABLE, DIRTY, MAINTENANCE"
        bigint updated_by FK "кто последний менял статус"
        timestamp updated_at
    }

    INVENTORY_CALENDAR {
        bigint id PK
        bigint room_id FK
        date calendar_date
        boolean is_available "false = занято/заблокировано"
        decimal price_override "цена на конкретную дату, опционально"
        bigint booking_id FK "какая бронь заблокировала дату, nullable"
    }

    BOOKINGS {
        bigint id PK
        bigint guest_id FK
        bigint room_id FK
        date check_in_date
        date check_out_date
        varchar status "PENDING, PENDING_APPROVAL, CONFIRMED, CHECKED_IN, CHECKED_OUT, CANCELLED, EXPIRED, REJECTED"
        int guests_count
        decimal total_price
        text special_requests
        timestamp created_at
        timestamp expires_at "created_at + 15 мин, для Scheduler'а"
    }

    BOOKING_EVENTS {
        bigint id PK
        bigint booking_id FK
        varchar old_status
        varchar new_status
        bigint changed_by FK "nullable — NULL, если изменение от Scheduler'а"
        timestamp changed_at
        text comment
    }

    PAYMENTS {
        bigint id PK
        bigint booking_id FK
        decimal amount
        varchar status "PENDING, COMPLETED, REFUNDED, FAILED"
        varchar payment_method
        varchar gateway_transaction_id UK
        timestamp created_at
        timestamp updated_at
    }

    USERS {
        bigint id PK
        varchar full_name
        varchar email UK
        varchar role "ADMIN, MANAGER, HOUSEKEEPER"
        timestamp created_at
    }
```

---

## Ключевые проектные решения, отражённые в модели

1. **Разделение физического и логического состояния номера.**
   `rooms.physical_status` — не связан напрямую с бронированием, меняется вручную (горничной/администратором) и хранит служебную информацию о состоянии номера. Фактическая занятость дат — исключительно в `inventory_calendar.is_available`. Это позволяет избежать конфликта: номер физически чист, но занят по календарю, и наоборот.

2. **`inventory_calendar` как источник истины по занятости.**
   Гранулярность — «номер × дата». Поле `booking_id` (nullable FK) указывает, какая бронь заблокировала конкретный день — это упрощает откат при отмене/истечении брони: система снимает блокировку только по тем датам, где `booking_id` совпадает.

3. **Аудит через `booking_events`.**
   Любое изменение `bookings.status` логируется отдельной строкой с указанием старого/нового статуса и инициатора (`changed_by`). Поле nullable, так как часть переходов (например, PENDING → EXPIRED) инициирует Scheduler, а не пользователь.

4. **Оплата вынесена в отдельную сущность.**
   Одна бронь может иметь несколько попыток оплаты (FAILED → повторная попытка), поэтому связь `BOOKINGS ||--o{ PAYMENTS` — один-ко-многим, а не один-к-одному.

5. **`USERS` — сущность для сотрудников отеля.**
   Нужна, чтобы предметно связать `booking_events.changed_by` и `rooms.updated_by` с конкретным сотрудником и его ролью (ADMIN / MANAGER / HOUSEKEEPER), а не хранить это как анонимный флаг.

## Что сознательно не включено в MVP-модель

- Сущность `hotels` / `properties` — MVP рассчитан на одну гостиницу до 100 номеров; мультиотельность вынесена в раздел "Roadmap" как будущее расширение до микросервисной архитектуры.
- Тарифные планы (rate plans) как отдельная сущность — в MVP цена хранится на уровне `room_types.base_price` с точечным переопределением через `inventory_calendar.price_override`.
- Скидки/промокоды — вне границ MVP (см. Vision_and_Scope.md, раздел "что не входит").
