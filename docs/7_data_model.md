## Модель данных

1. Основные сущности

| Сущность | Описание |
|---|---|
| USER | Клиент Банка N |
| ACCOUNT | Счёт или карта, с которого выполняется списание |
| TRANSFER | Операция перевода по номеру телефона |
| TRANSFER_STATUS_HISTORY | История изменения статусов перевода |
| TRANSFER_LIMIT | Лимит переводов пользователя за период |

2. Атрибуты сущностей

USER

| Атрибут | Тип | Ключ / ограничение | Описание |
| --- | --- | --- | --- |
| user_id | UUID | PK, NOT NULL | Уникальный идентификатор клиента |
| full_name | VARCHAR(255) | NOT NULL | ФИО клиента |
| phone | VARCHAR(20) | NOT NULL, UNIQUE | Номер телефона клиента |
| status | VARCHAR(20) | NOT NULL | Статус клиента: ACTIVE, BLOCKED |
| created_at | TIMESTAMP | NOT NULL | Дата регистрации клиента |

ACCOUNT

| Атрибут           | Тип           | Ключ / ограничение          | Описание                      |
| ----------------- | ------------- | --------------------------- | ----------------------------- |
| account_id        | UUID          | PK, NOT NULL                | Идентификатор счёта или карты |
| user_id           | UUID          | FK → USER.user_id, NOT NULL | Владелец счёта                |
| account_type      | VARCHAR(20)   | NOT NULL                    | CARD, ACCOUNT, SAVINGS        |
| currency          | VARCHAR(3)    | NOT NULL                    | Валюта: RUB                   |
| available_balance | DECIMAL(18,2) | NOT NULL                    | Доступный остаток             |
| status            | VARCHAR(20)   | NOT NULL                    | ACTIVE, BLOCKED, CLOSED       |
| created_at        | TIMESTAMP     | NOT NULL                    | Дата открытия                 |

TRANSFER

| Атрибут               | Тип           | Ключ / ограничение                | Описание                                    |
| --------------------- | ------------- | --------------------------------- | ------------------------------------------- |
| transfer_id           | UUID          | PK, NOT NULL                      | Идентификатор перевода                      |
| sender_user_id        | UUID          | FK → USER.user_id, NOT NULL       | Отправитель                                 |
| source_account_id     | UUID          | FK → ACCOUNT.account_id, NOT NULL | Счёт списания                               |
| recipient_phone       | VARCHAR(20)   | NOT NULL                          | Телефон получателя                          |
| recipient_user_id     | UUID          | FK → USER.user_id, NULL           | Заполняется, если получатель — клиент банка |
| recipient_bank_id     | VARCHAR(20)   | NULL                              | Заполняется для перевода через СБП          |
| transfer_type         | VARCHAR(20)   | NOT NULL                          | INTERNAL или SBP                            |
| amount                | DECIMAL(18,2) | NOT NULL                          | Сумма перевода                              |
| commission            | DECIMAL(18,2) | NOT NULL                          | Комиссия                                    |
| currency              | VARCHAR(3)    | NOT NULL                          | Валюта                                      |
| comment               | VARCHAR(255)  | NULL                              | Комментарий                                 |
| status                | VARCHAR(20)   | NOT NULL                          | Текущий статус                              |
| idempotency_key       | UUID          | NOT NULL, UNIQUE                  | Ключ защиты от повторного запроса           |
| external_operation_id | VARCHAR(100)  | NULL                              | Идентификатор операции в СБП                |
| created_at            | TIMESTAMP     | NOT NULL                          | Дата создания                               |
| completed_at          | TIMESTAMP     | NULL                              | Дата завершения                             |

TRANSFER_STATUS_HISTORY

| Атрибут           | Тип          | Ключ / ограничение                  | Описание                                       |
| ----------------- | ------------ | ----------------------------------- | ---------------------------------------------- |
| status_history_id | UUID         | PK, NOT NULL                        | Идентификатор записи истории                   |
| transfer_id       | UUID         | FK → TRANSFER.transfer_id, NOT NULL | Перевод, к которому относится запись           |
| status            | VARCHAR(20)  | NOT NULL                            | Статус на момент записи                        |
| status_reason     | VARCHAR(255) | NULL                                | Причина изменения статуса                      |
| changed_at        | TIMESTAMP    | NOT NULL                            | Когда изменился статус                         |
| changed_by        | VARCHAR(50)  | NOT NULL                            | Кто или что изменило статус: SYSTEM, USER, SBP |

TRANSFER_LIMIT

| Атрибут       | Тип           | Ключ / ограничение          | Описание                             |
| ------------- | ------------- | --------------------------- | ------------------------------------ |
| limit_id      | UUID          | PK, NOT NULL                | Идентификатор лимита                 |
| user_id       | UUID          | FK → USER.user_id, NOT NULL | Клиент, для которого действует лимит |
| transfer_type | VARCHAR(20)   | NOT NULL                    | INTERNAL или SBP                     |
| period_type   | VARCHAR(20)   | NOT NULL                    | DAILY, MONTHLY                       |
| period_start  | DATE          | NOT NULL                    | Начало периода                       |
| period_end    | DATE          | NOT NULL                    | Конец периода                        |
| limit_amount  | DECIMAL(18,2) | NOT NULL                    | Максимальная сумма за период         |
| used_amount   | DECIMAL(18,2) | NOT NULL                    | Уже использованная сумма             |

IDEMPOTENCY_KEY

| Атрибут         | Тип          | Ключ / ограничение              | Описание                   |
| --------------- | ------------ | ------------------------------- | -------------------------- |
| idempotency_key | UUID         | PK, NOT NULL                    | Уникальный ключ запроса    |
| user_id         | UUID         | FK → USER.user_id, NOT NULL     | Клиент, отправивший запрос |
| request_hash    | VARCHAR(255) | NOT NULL                        | Хэш тела запроса           |
| transfer_id     | UUID         | FK → TRANSFER.transfer_id, NULL | Связанный перевод          |
| created_at      | TIMESTAMP    | NOT NULL                        | Когда получен запрос       |


3. Связи

USER 1 --- N ACCOUNT
USER 1 --- N TRANSFER
USER 1 --- N TRANSFER как получатель
ACCOUNT 1 --- N TRANSFER
TRANSFER 1 --- N TRANSFER_STATUS_HISTORY
USER 1 --- N TRANSFER_LIMIT
TRANSFER 1 --- 0...1 IDEMPOTENCY_KEY
USER 1 --- N IDEMPOTENCY_KEY

