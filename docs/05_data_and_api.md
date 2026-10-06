# 5. Модель данных и интеграция с МИС

[← К оглавлению](../README.md)

## Логическая модель данных

```mermaid
erDiagram
    BRANCH ||--o{ DOCTOR_BRANCH : "работают"
    DOCTOR ||--o{ DOCTOR_BRANCH : "принимает в"
    SPECIALTY ||--o{ DOCTOR : "имеет"
    DOCTOR ||--o{ SLOT : "расписание"
    BRANCH ||--o{ SLOT : "проходит в"
    SLOT ||--o| APPOINTMENT : "занят"
    PATIENT ||--o{ APPOINTMENT : "записан"
    APPOINTMENT ||--o{ NOTIFICATION : "уведомления"

    BRANCH {
        int branch_id PK
        string name
        string address
        string phone
    }
    SPECIALTY {
        int specialty_id PK
        string name
    }
    DOCTOR {
        int doctor_id PK
        int specialty_id FK
        string full_name
        int experience_years
        string photo_url
    }
    DOCTOR_BRANCH {
        int doctor_id FK
        int branch_id FK
    }
    SLOT {
        int slot_id PK
        int doctor_id FK
        int branch_id FK
        datetime start_at
        int duration_min
        bool is_reserved
    }
    PATIENT {
        int patient_id PK
        string phone
        string full_name
        date birth_date
    }
    APPOINTMENT {
        int appointment_id PK
        int slot_id FK
        int patient_id FK
        string status
        string source
        datetime created_at
        string comment
    }
    NOTIFICATION {
        int notification_id PK
        int appointment_id FK
        string type
        datetime sent_at
        string delivery_status
    }
```

**Статусы записи (`APPOINTMENT.status`):**

```mermaid
stateDiagram-v2
    [*] --> Подтверждена: запись создана
    Подтверждена --> Подтверждена: перенос на другой слот
    Подтверждена --> Отменена_пациентом: отмена ≥ 2 ч до приёма
    Подтверждена --> Отменена_клиникой: врач заболел / изменилось расписание
    Подтверждена --> Состоялась: пациент пришёл
    Подтверждена --> Неявка: пациент не пришёл
    Отменена_пациентом --> [*]
    Отменена_клиникой --> [*]
    Состоялась --> [*]
    Неявка --> [*]
```

При переносе новая запись не создаётся: у существующей меняется слот, статус остаётся «Подтверждена». История переносов пишется в журнал (NFR-07). Так у записи сохраняется комментарий пациента, а в отчётах один визит не считается дважды.

## Интеграция с МИС (REST API)

Своё расписание сервис записи не ведёт, главный источник данных о слотах — МИС. Слоты кэшируются не дольше минуты (FR-02), а перед созданием записи слот ещё раз проверяется в МИС (FR-04). Ниже методы, которые нужны от API МИС; список нужно согласовать с подрядчиком.

| Метод | Endpoint | Назначение | Требование |
|---|---|---|---|
| GET | `/branches` | Список филиалов | FR-01 |
| GET | `/doctors?branch_id=&specialty_id=` | Врачи по фильтру | FR-01 |
| GET | `/doctors/{id}/slots?date_from=&date_to=` | Свободные слоты врача | FR-02 |
| GET | `/patients?phone=` | Поиск пациента по телефону | FR-05 |
| POST | `/patients` | Создание карты пациента | FR-05 |
| POST | `/appointments` | Создание записи | FR-04, FR-10 |
| PATCH | `/appointments/{id}` | Перенос записи на другой слот | FR-08 |
| POST | `/appointments/{id}/cancel` | Отмена записи. Запись не удаляется, а получает статус «Отменена пациентом» | FR-08 |

### Пример: создание записи

**Запрос**

```http
POST /api/v1/appointments
Content-Type: application/json
```

```json
{
  "slot_id": 58213,
  "patient_id": 10442,
  "source": "online",
  "comment": "Первичный приём"
}
```

**Ответ 201 Created**

```json
{
  "appointment_id": 900125,
  "status": "confirmed",
  "doctor": { "doctor_id": 17, "full_name": "Иванова Анна Андреевна" },
  "branch": { "branch_id": 2, "address": "ул. Примерная, 10" },
  "start_at": "2026-10-15T10:30:00+03:00",
  "duration_min": 20
}
```

### Обработка ошибок

| Код | Ситуация | Что показать пациенту |
|---|---|---|
| 400 | Некорректные данные запроса | «Проверьте введённые данные» |
| 404 | Слот или пациент не найден | «Время больше недоступно, выберите другое» |
| 409 | Слот уже занят (конфликт) | «Это время уже заняли» + ближайшие слоты (исключение 14а в UC-01) |
| 422 | Нарушено бизнес-правило (например, лимит в 3 записи) | Текст правила |
| 503 | МИС недоступна | «Запись временно недоступна» + телефон филиала (NFR-06) |
