# РОЗДІЛ 2. ПРОЄКТУВАННЯ АРХІТЕКТУРИ СИСТЕМИ

## 2.1 Контекстне та контейнерне моделювання системи (C4 Model)

### <a id="diagram-level1"></a>Рівень 1. System Context Diagram

```mermaid
C4Context
    title FinJar — System Context
    Person(user, "Individual User", "Облік особистих гаманців і баночок")
    Person(family, "Family Member", "Учасник спільного бюджету (тариф Pro)")
    Person(accountant, "Business Accountant", "Облік підприємства (тариф Business)")
    System(finjar, "FinJar", "Мультивалютний облік коштів, розподіл доходу, LLM-асистент")
    System_Ext(fx, "FX Rates Provider", "Курси валют")
    System_Ext(bank, "Bank Webhook Source", "Події банківських транзакцій")
    System_Ext(llm, "LLM Provider", "Генерація відповідей, tool calling")
    System_Ext(pay, "Payment Provider", "Оплата підписок")
    System_Ext(mail, "Email Service", "Транзакційні листи")
    Rel(user, finjar, "Використовує", "HTTPS")
    Rel(family, finjar, "Використовує", "HTTPS")
    Rel(accountant, finjar, "Використовує", "HTTPS")
    Rel(finjar, fx, "Отримує курси", "REST / HTTPS")
    Rel(bank, finjar, "Надсилає події", "HTTPS webhook, HMAC-SHA256")
    Rel(finjar, llm, "Надсилає агреговані запити", "REST / HTTPS")
    Rel(pay, finjar, "Події підписки", "HTTPS webhook")
    Rel(finjar, mail, "Надсилає листи", "HTTPS API")
```

Система FinJar взаємодіє з трьома ролями користувачів та п'ятьма зовнішніми системами. Вхідні webhook-події від банку та платіжного провайдера автентифікуються підписом HMAC-SHA256 відповідно до [SRS-NFR-SEC-04](srs.md#srs-nfr-sec-04). Обмін з LLM Provider містить лише агреговані знеособлені дані ([SRS-FUN-13](srs.md#srs-fun-13)).

### <a id="diagram-level2"></a>Рівень 2. Container Diagram

```mermaid
C4Container
    title FinJar — Container Diagram
    Person(user, "User", "Individual / Family / Business")
    System_Boundary(fj, "FinJar") {
        Container(web, "Web App", "Next.js, TypeScript, Tailwind CSS", "SSR/SPA-клієнт, англомовний UI")
        Container(proxy, "Reverse Proxy", "Nginx", "TLS termination, статичні файли, маршрутизація /api")
        Container(api, "Core API", "Django REST Framework, Gunicorn", "Модулі wallets, ledger, fx, jars, billing, assistant, reports")
        Container(worker, "Async Workers", "Celery", "Розподіл доходу, оновлення курсів, звіти, LLM-запити")
        Container(beat, "Scheduler", "Celery Beat", "Розклад оновлення курсів і Outbox-релею")
        ContainerDb(db, "Primary DB", "PostgreSQL 16", "Реєстр подвійного запису, Outbox, аудит, pgvector")
        ContainerDb(cache, "Cache", "Redis", "Кеш курсів, rate limiting, result backend")
        ContainerQueue(mq, "Message Broker", "RabbitMQ", "Черги Celery, dead-letter exchange")
        Container(otel, "Observability", "OpenTelemetry Collector, Prometheus, Grafana", "Трасування та метрики")
    }
    System_Ext(fx, "FX Rates Provider", "")
    System_Ext(llm, "LLM Provider", "")
    Rel(user, proxy, "Використовує", "HTTPS")
    Rel(proxy, web, "Проксіює сторінки", "HTTP/1.1")
    Rel(proxy, api, "Проксіює /api", "HTTP/1.1")
    Rel(web, api, "Виклики API", "REST / HTTPS, JWT")
    Rel(api, db, "Читає/пише", "SQL / TCP 5432")
    Rel(api, cache, "Кеш, rate limit", "RESP / TCP 6379")
    Rel(beat, mq, "Публікує задачі", "AMQP 0-9-1")
    Rel(worker, mq, "Споживає задачі", "AMQP 0-9-1")
    Rel(worker, db, "Читає Outbox, пише проводки", "SQL / TCP 5432")
    Rel(worker, fx, "Отримує курси", "REST / HTTPS")
    Rel(worker, llm, "Tool calling", "REST / HTTPS")
    Rel(api, otel, "Трейси, метрики", "OTLP / gRPC")
    Rel(worker, otel, "Трейси, метрики", "OTLP / gRPC")
```

Опис контейнерів та трасування вимог:

- **Web App** реалізує інтерфейс користувача англійською мовою та звертається до Core API через REST / HTTPS з JWT ([SRS-FUN-01](srs.md#srs-fun-01)).
- **Reverse Proxy (Nginx)** завершує TLS 1.2+ ([SRS-NFR-SEC-02](srs.md#srs-nfr-sec-02)).
- **Core API** — stateless-контейнер ([SRS-NFR-OPS-01](srs.md#srs-nfr-ops-01)), що реалізує модулі гаманців, реєстру та баночок ([SRS-FUN-02](srs.md#srs-fun-02) – [SRS-FUN-12](srs.md#srs-fun-12)); rate limiting виконується через Redis ([SRS-NFR-SEC-03](srs.md#srs-nfr-sec-03)).
- **Async Workers** обробляють події розподілу доходу ([SRS-FUN-08](srs.md#srs-fun-08)) через RabbitMQ відповідно до рішення [ADR-001](adr/0001-choice-of-technology.md).
- **Primary DB** зберігає реєстр подвійного запису ([SRS-FUN-04](srs.md#srs-fun-04)) і таблицю Outbox ([SRS-NFR-REL-02](srs.md#srs-nfr-rel-02)).
- **Observability** забезпечує trace ID для кожного запиту ([SRS-NFR-OBS-01](srs.md#srs-nfr-obs-01)).

## 2.2 Компонентне моделювання критичного сервісу системи

Критичним обрано контейнер **Async Workers** у частині модуля `jars` — сервіс розподілу доходу (Allocation Service), що забезпечує ідемпотентне застосування процентних правил до подій надходження.

### <a id="diagram-level3"></a>Рівень 3. Component Diagram — Allocation Service

```mermaid
C4Component
    title Allocation Service (Celery worker, модуль jars) — Component Diagram
    ContainerQueue(mq, "RabbitMQ", "AMQP 0-9-1", "Черга allocation, DLX")
    ContainerDb(db, "PostgreSQL", "SQL", "Ledger, Outbox, ProcessedEvents")
    Container(otel, "OpenTelemetry Collector", "OTLP", "")
    Container_Boundary(svc, "Allocation Service") {
        Component(consumer, "IncomeEventConsumer", "Celery task", "Приймає подію IncomeReceived, ack після commit")
        Component(validator, "EventSchemaValidator", "Pydantic", "Валідація DTO події")
        Component(idem, "IdempotencyGuard", "Service", "Перевірка event_id у ProcessedEvents")
        Component(usecase, "AllocateIncomeUseCase", "Domain service", "Оркестрація розподілу в одній транзакції")
        Component(rules, "AllocationRuleEngine", "Domain", "Обчислення часток, округлення Decimal, залишок на останню баночку")
        Component(ledgerport, "LedgerPort", "Interface", "Абстракція запису проводок")
        Component(ledgerrepo, "DjangoLedgerRepository", "Django ORM", "Реалізація LedgerPort")
        Component(rulerepo, "RuleRepository", "Django ORM", "Читання активних правил")
        Component(audit, "AuditPublisher", "Service", "Append-only запис AuditEvent")
        Component(errors, "TaskErrorHandler", "Celery signal", "Retry з backoff, відправка в DLX")
        Component(metrics, "AllocationMetrics", "prometheus_client, OTel SDK", "allocation_latency_seconds, trace")
    }
    Rel(mq, consumer, "Доставляє подію", "AMQP")
    Rel(consumer, validator, "Валідує payload", "виклик функції")
    Rel(consumer, idem, "Перевіряє дублікат", "DI")
    Rel(consumer, usecase, "Викликає", "DI")
    Rel(usecase, rules, "Обчислює частки", "виклик домену")
    Rel(usecase, rulerepo, "Читає правила", "інтерфейс")
    Rel(usecase, ledgerport, "Записує проводки", "Dependency Inversion")
    Rel(ledgerrepo, ledgerport, "Реалізує", "")
    Rel(ledgerrepo, db, "INSERT у межах транзакції", "SQL")
    Rel(rulerepo, db, "SELECT", "SQL")
    Rel(idem, db, "INSERT ON CONFLICT DO NOTHING", "SQL")
    Rel(usecase, audit, "Публікує аудит", "DI")
    Rel(consumer, errors, "Помилки обробки", "Celery signals")
    Rel(errors, mq, "Повтор / DLX", "AMQP")
    Rel(consumer, metrics, "Латентність, трейс", "виклик")
    Rel(metrics, otel, "Експорт", "OTLP / gRPC")
```

Опис компонентів:

- **IncomeEventConsumer** — вхідний шар; підтверджує повідомлення (`acks_late`) лише після commit транзакції, що забезпечує at-least-once обробку ([SRS-NFR-REL-02](srs.md#srs-nfr-rel-02)).
- **EventSchemaValidator** — валідація DTO події до виконання бізнес-логіки; невалідні події одразу направляються в dead-letter exchange.
- **IdempotencyGuard** — вставка `event_id` у таблицю `processed_events` з унікальним обмеженням у тій самій транзакції, що й проводки; повторна подія не змінює стан ([SRS-FUN-08](srs.md#srs-fun-08)).
- **AllocateIncomeUseCase** — доменний сервіс, що залежить від абстракцій `LedgerPort` і `RuleRepository` (принцип інверсії залежностей); прямий доступ вхідного шару до БД відсутній.
- **AllocationRuleEngine** — чиста доменна логіка: обчислення часток у `Decimal` з округленням до двох знаків і віднесенням залишку округлення на останню баночку, що зберігає інваріант подвійного запису ([SRS-NFR-REL-01](srs.md#srs-nfr-rel-01)).
- **AuditPublisher** — запис події аудиту ([SRS-FUN-10](srs.md#srs-fun-10)).
- **TaskErrorHandler** — повторні спроби з експоненційним backoff (до 5 спроб) і відправка в DLX.
- **AllocationMetrics** — гістограма `allocation_latency_seconds` для верифікації [SRS-NFR-PERF-03](srs.md#srs-nfr-perf-03).

## 2.3 Документування архітектурних рішень (Architecture Decision Records)

| ID | Рішення | Статус |
|---|---|---|
| [ADR-001](adr/0001-choice-of-technology.md) | Вибір RabbitMQ як брокера повідомлень для асинхронного розподілу доходу | Approved |

## 2.4 Проєктування та моделювання баз даних

Шар збереження реалізовано в PostgreSQL 16 за принципом Data-as-Code. Грошові суми зберігаються в `NUMERIC(23,8)` (до 8 знаків для криптоактивів), курси — у `NUMERIC(22,10)`, ідентифікатори — `UUID` (v7, впорядковані за часом).

```mermaid
erDiagram
    USERS ||--o{ BUDGET_MEMBERS : "входить у"
    BUDGETS ||--o{ BUDGET_MEMBERS : "має учасників"
    BUDGETS ||--|| SUBSCRIPTIONS : "має тариф"
    BUDGETS ||--o{ LEDGER_ACCOUNTS : "містить"
    LEDGER_ACCOUNTS ||--o| WALLETS : "деталізує"
    LEDGER_ACCOUNTS ||--o| JARS : "деталізує"
    BUDGETS ||--o{ TRANSACTIONS : "реєструє"
    TRANSACTIONS ||--|{ LEDGER_ENTRIES : "складається з"
    LEDGER_ACCOUNTS ||--o{ LEDGER_ENTRIES : "отримує"
    JARS ||--o{ ALLOCATION_RULES : "ціль правила"
    BUDGETS ||--o{ OUTBOX_EVENTS : "публікує"
    BUDGETS ||--o{ AUDIT_EVENTS : "журналює"

    USERS {
        uuid id PK
        varchar(254) email UK
        varchar(128) password_hash "Argon2id"
        char(3) base_currency
        timestamptz created_at
    }
    BUDGETS {
        uuid id PK
        varchar(100) name
        char(3) base_currency
        timestamptz created_at
    }
    BUDGET_MEMBERS {
        uuid budget_id PK, FK
        uuid user_id PK, FK
        varchar(10) role "owner | member | viewer"
    }
    SUBSCRIPTIONS {
        uuid id PK
        uuid budget_id FK, UK
        varchar(10) plan "free | pro | business"
        timestamptz valid_until
    }
    LEDGER_ACCOUNTS {
        uuid id PK
        uuid budget_id FK
        varchar(10) kind "wallet | jar | income | expense | fx"
        varchar(10) currency
        boolean is_archived
    }
    WALLETS {
        uuid account_id PK, FK
        varchar(100) name
        varchar(15) type "bank_card | credit_card | crypto_wallet | cash"
        numeric credit_limit "NUMERIC(23,8) NULL"
    }
    JARS {
        uuid account_id PK, FK
        varchar(100) name
    }
    ALLOCATION_RULES {
        uuid id PK
        uuid budget_id FK
        uuid jar_account_id FK
        numeric percent "NUMERIC(5,2) CHECK 0..100"
        smallint priority
        boolean is_active
    }
    TRANSACTIONS {
        uuid id PK
        uuid budget_id FK
        varchar(15) kind
        timestamptz occurred_at
        varchar(500) description
        uuid idempotency_key UK
        uuid source_event_id "NULL, UK"
        timestamptz created_at
    }
    LEDGER_ENTRIES {
        uuid id PK
        uuid transaction_id FK
        uuid account_id FK
        char(1) direction "D | C"
        numeric amount "NUMERIC(23,8) CHECK > 0"
        varchar(10) currency
        numeric fx_rate "NUMERIC(22,10)"
        numeric amount_base "NUMERIC(23,8)"
    }
    FX_RATES {
        varchar(10) base PK
        varchar(10) quote PK
        date on_date PK
        numeric rate "NUMERIC(22,10)"
        varchar(30) source
    }
    OUTBOX_EVENTS {
        uuid id PK
        uuid budget_id FK
        varchar(50) event_type
        jsonb payload
        timestamptz created_at
        timestamptz published_at "NULL"
    }
    PROCESSED_EVENTS {
        uuid event_id PK
        timestamptz processed_at
    }
    AUDIT_EVENTS {
        bigint id PK
        uuid budget_id FK
        uuid actor_id FK
        varchar(50) action
        jsonb diff
        timestamptz created_at
    }
```

**План індексування:**

| Таблиця | Індекс | Тип | Призначення |
|---|---|---|---|
| `ledger_entries` | `(account_id, transaction_id)` | B-Tree | Обчислення балансу рахунку та виписки |
| `transactions` | `(budget_id, occurred_at DESC)` | B-Tree | Журнал транзакцій з пагінацією за курсором |
| `transactions` | `idempotency_key` UNIQUE | B-Tree | Захист від повторного створення ([SRS-FUN-04](srs.md#srs-fun-04)) |
| `transactions` | `source_event_id` UNIQUE WHERE NOT NULL | Partial B-Tree | Однократний розподіл для події надходження ([SRS-FUN-08](srs.md#srs-fun-08)) |
| `outbox_events` | `created_at` WHERE `published_at IS NULL` | Partial B-Tree | Вибірка неопублікованих подій релеєм |
| `fx_rates` | PK `(base, quote, on_date)` | B-Tree | Пошук курсу на дату |
| `audit_events` | `diff` | GIN | Пошук змін за атрибутами |
| `allocation_rules` | `(budget_id) WHERE is_active` | Partial B-Tree | Вибірка активних правил |

Інваріант подвійного запису контролюється відкладеним (DEFERRABLE INITIALLY DEFERRED) тригером-обмеженням: для кожної транзакції `SUM(amount_base WHERE direction = 'D') = SUM(amount_base WHERE direction = 'C')` ([SRS-NFR-REL-01](srs.md#srs-nfr-rel-01)). Таблиці `ledger_entries` та `audit_events` доступні лише на вставку (append-only) через права ролі БД.

## 2.5 Специфікація міжсервісних контрактів (API)

Контракт REST API описано за підходом API-First у форматі OpenAPI 3.0.3: [openapi.yaml](api/openapi.yaml). Контракт містить 10 маршрутів і 16 схем даних.

Ключові рішення контракту:

- Грошові суми передаються рядками `Money` (pattern `^-?\d{1,15}(\.\d{1,8})?$`), що відповідає `NUMERIC(23,8)` у БД та виключає похибки float у JavaScript-клієнті.
- Створення транзакцій, гаманців і зняття з баночок вимагає заголовка `Idempotency-Key` (UUID), що відповідає унікальному індексу `transactions.idempotency_key`.
- Помилки повертаються у форматі RFC 7807 (`application/problem+json`) з `trace_id`; задокументовано коди 400, 401, 402, 403, 404, 409, 422, 429, 504.
- Пагінація журналів виконується за курсором (`cursor`, `limit ≤ 100`).
- Вебхук `/webhooks/bank` автентифікується заголовком `X-Signature` (HMAC-SHA256) і повертає 202 з подальшою асинхронною обробкою.

## 2.6 Моделювання логіки критичних вузлів

### 2.6.1 Алгоритм ідемпотентного розподілу доходу за баночками

Вхідні дані: подія `E` (event_id, budget_id, amount `A`, currency), впорядкований за `priority` масив активних правил `R[1..n]` з частками `R[i].percent`.

1. Початок; ввести подію `E`.
2. Якщо `event_id` уже є в `processed_events`, завершити без змін.
3. Отримати правила `R[1..n]`; якщо `n = 0 OR SUM(R[i].percent) ≠ 100`, записати помилку конфігурації та завершити.
4. `allocated := 0`; `i := 1`.
5. Поки `i ≤ n`: якщо `i < n`, то `share[i] := ROUND(A × R[i].percent / 100, 2)`, інакше `share[i] := A − allocated`; `allocated := allocated + share[i]`; `i := i + 1`.
6. Сформувати проводки: дебет `Jar[R[i].jar]` на `share[i]`, кредит рахунку надходження на `A`.
7. Якщо `SUM(debit) = SUM(credit) AND allocated = A`, зафіксувати транзакцію та `event_id` в одній ACID-транзакції; інакше відкотити транзакцію.
8. Кінець.

```mermaid
flowchart TD
    S(["Початок"]) --> IN[/"Ввід: E; A := E.amount"/]
    IN --> D1{"E.event_id ∈ ProcessedEvents?"}
    D1 -- Так --> F(["Кінець"])
    D1 -- Ні --> RD[/"Ввід: R[1..n] := активні правила"/]
    RD --> D2{"n = 0 OR Σ R[i].percent ≠ 100?"}
    D2 -- Так --> ERR[/"Вивід: помилка конфігурації"/] --> F
    D2 -- Ні --> P1["allocated := 0; i := 1"]
    P1 --> D3{"i ≤ n?"}
    D3 -- Так --> D4{"i < n?"}
    D4 -- Так --> P2["share[i] := ROUND(A × R[i].percent / 100, 2)"]
    D4 -- Ні --> P3["share[i] := A − allocated"]
    P2 --> P4["allocated := allocated + share[i]; i := i + 1"]
    P3 --> P4
    P4 --> D3
    D3 -- Ні --> P5["entries := Debit(Jar[R[i].jar], share[i]) ∪ Credit(Income, A)"]
    P5 --> D5{"Σ Debit = Σ Credit AND allocated = A?"}
    D5 -- Так --> P6["COMMIT(entries, E.event_id)"]
    D5 -- Ні --> P7["ROLLBACK"]
    P6 --> OUT[/"Вивід: transaction_id"/] --> F
    P7 --> F
```

### 2.6.2 Алгоритм реєстрації мультивалютної транзакції

Вхідні дані: проводки `Entry[1..m]` з `amount`, `currency`, `direction`; базова валюта бюджету `B`; дата операції `d`.

1. Початок; ввести `Entry[1..m]`, `B`, `d`.
2. Якщо `m < 2`, повернути 422.
3. `j := 1`; поки `j ≤ m`: якщо `Entry[j].currency = B`, то `rate := 1`, інакше `rate := FxRate(Entry[j].currency, B, d)`; якщо курс відсутній, повернути 422; `Entry[j].amount_base := ROUND(Entry[j].amount × rate, 2)`; `j := j + 1`.
4. `delta := ΣEntry[j].amount_base(D) − ΣEntry[j].amount_base(C)`.
5. Якщо `|delta| > 0 AND |delta| ≤ 0.01 × m`, додати проводку курсової різниці на рахунок `fx` на суму `delta`; якщо `|delta| > 0.01 × m`, повернути 422.
6. Зберегти транзакцію, проводки та подію Outbox в одній ACID-транзакції; повернути 201.
7. Кінець.

```mermaid
flowchart TD
    S(["Початок"]) --> IN[/"Ввід: Entry[1..m], B, d"/]
    IN --> D1{"m < 2?"}
    D1 -- Так --> E1[/"Вивід: 422"/] --> F(["Кінець"])
    D1 -- Ні --> P1["j := 1"]
    P1 --> D2{"j ≤ m?"}
    D2 -- Так --> D3{"Entry[j].currency = B?"}
    D3 -- Так --> P2["rate := 1"]
    D3 -- Ні --> P3["rate := FxRate(Entry[j].currency, B, d)"]
    P3 --> D4{"rate = NULL?"}
    D4 -- Так --> E1
    D4 -- Ні --> P4
    P2 --> P4["Entry[j].amount_base := ROUND(Entry[j].amount × rate, 2); j := j + 1"]
    P4 --> D2
    D2 -- Ні --> P5["delta := Σ Debit.amount_base − Σ Credit.amount_base"]
    P5 --> D5{"delta ≠ 0 AND |delta| ≤ 0.01 × m?"}
    D5 -- Так --> P6["Entry[m+1] := FxDifference(delta)"]
    D5 -- Ні --> D6{"delta ≠ 0?"}
    D6 -- Так --> E1
    D6 -- Ні --> P7
    P6 --> P7["COMMIT(Transaction, Entry[1..m], OutboxEvent)"]
    P7 --> OUT[/"Вивід: 201, transaction_id"/] --> F
```

## 2.7 Інтеграційний аудит та верифікація цілісності доменів

**Production Readiness Report** — результат перехресної перевірки ERD (2.4), контракту OpenAPI (2.5) та алгоритмів (2.6).

| Перевірка | Результат |
|---|---|
| Тип грошових сум: `NUMERIC(23,8)` у БД ↔ `Money` (decimal string, до 8 знаків) в API ↔ `Decimal` в алгоритмах | Узгоджено |
| Курси: `NUMERIC(22,10)` у `fx_rates` ↔ `FxRate.rate` (до 10 знаків) ↔ `rate` в алгоритмі 2.6.2 | Узгоджено |
| Частки правил: `NUMERIC(5,2)` ↔ `AllocationRule.percent` ↔ умова `ΣR[i].percent = 100` | Узгоджено |
| Ідемпотентність: `transactions.idempotency_key` UK ↔ заголовок `Idempotency-Key` ↔ `processed_events` в алгоритмі 2.6.1 | Узгоджено |
| Коди валют: `varchar(10)` ↔ `CurrencyCode` `^[A-Z0-9]{3,10}$` (ISO 4217 та тикери криптоактивів) | Виправлено: початково `char(3)` у `ledger_accounts` не вміщував тикери USDT/BTC-подібної довжини > 3 |
| Коментар зняття: `varchar(500)` ↔ `JarWithdrawal.comment maxLength 500` ↔ SRS-FUN-09 | Узгоджено |
| Порядок правил: `allocation_rules.priority` ↔ `AllocationRule.priority` ↔ впорядкований масив `R[1..n]` | Виправлено: поле `priority` додано в ERD після аудиту |
| Обмеження `transactions.kind`: значення `allocation` створюється лише воркером і відсутнє в `TransactionCreate.kind` | Узгоджено (навмисне обмеження API) |

Конфліктів типів і семантичних розбіжностей між шарами після виправлень не виявлено.

## 2.8 Обґрунтування архітектурного стилю та патернів проєктування

**Глобальний архітектурний стиль.** Для FinJar обрано модульний моноліт (Modular Monolith) на Django REST Framework з виділеним асинхронним контуром обробки подій (Celery, RabbitMQ). Доменні модулі `wallets`, `ledger`, `fx`, `jars`, `billing`, `assistant`, `reports` мають власні моделі, сервіси та публічні інтерфейси; міжмодульні виклики виконуються лише через сервісний шар, прямий імпорт моделей іншого модуля заборонено правилом статичного аналізу (import-linter). Усі модулі працюють з однією базою PostgreSQL, що дозволяє фіксувати транзакцію, проводки подвійного запису та подію Outbox в одній ACID-транзакції без розподілених саг.

Вибір забезпечує виконання NFR таким чином. Вимога затримки читання p95 ≤ 200 мс при 300 RPS ([SRS-NFR-PERF-01](srs.md#srs-nfr-perf-01)) досягається відсутністю мережевих викликів між модулями в межах запиту, кешуванням курсів валют у Redis та курсорною пагінацією. Вимога розподілу доходу ≤ 5 с ([SRS-NFR-PERF-03](srs.md#srs-nfr-perf-03)) виконується винесенням розподілу з HTTP-запиту в Celery-воркери; кількість воркерів масштабується горизонтально без зміни коду. Stateless-контейнери Core API ([SRS-NFR-OPS-01](srs.md#srs-nfr-ops-01)) масштабуються за балансувальником Nginx.

Компроміс стилю: спільна база даних обмежує незалежне масштабування окремих модулів, а релізи всіх модулів виконуються синхронно. Для обсягу навантаження дипломного проєкту цей компроміс прийнятний; межі модулів спроєктовано так, щоб модуль `assistant` або `reports` за потреби виділявся в окремий сервіс без зміни доменної моделі реєстру.

**Патерни мікрорівня:**

- **Service Layer + Repository** — ізолюють бізнес-логіку від Django ORM; доменні сервіси залежать від інтерфейсів репозиторіїв (принцип інверсії залежностей), що знижує зв'язність і дозволяє тестувати логіку без бази даних.
- **Unit of Work** (`transaction.atomic`) — гарантує атомарність запису транзакції, проводок, `processed_events` і Outbox.
- **Transactional Outbox + Idempotent Consumer** — забезпечують at-least-once доставку подій без дублювання ефектів ([SRS-NFR-REL-02](srs.md#srs-nfr-rel-02)).
- **Strategy** — окремі стратегії обчислення для типів гаманців (`credit_card` враховує ліміт і заборгованість) та для провайдерів курсів валют.
- **Dependency Injection** — збирання залежностей сервісів на рівні фабрик модулів; у тестах реалізації підміняються in-memory репозиторіями.
- **Read-only Tool Gateway** для LLM-асистента — асистент отримує доступ лише до переліку дозволених read-only функцій, що виключає зміну даних моделлю ([SRS-FUN-13](srs.md#srs-fun-13)).

## 2.9 Опис серверного ядра та контур безпеки застосунку (Application Security)

**Шлях даних через рівні Core API.** Inbound Layer: Nginx завершує TLS 1.2+ і передає запит у Gunicorn; middleware-ланцюжок Django виконує: присвоєння trace ID (OpenTelemetry), обмеження частоти запитів (Redis Token Bucket), автентифікацію JWT (SimpleJWT), перевірку RBAC-ролі в бюджеті (`IsBudgetMember`, `IsBudgetOwner`), перевірку тарифу (`PlanFeatureGuard`). Серіалізатори DRF валідують DTO: типи, діапазони, pattern для `Money` і `CurrencyCode`. Core Layer: доменні сервіси (`PostTransactionService`, `AllocationRuleService`, `JarWithdrawalService`) виконують бізнес-правила та інваріанти. Outbound Layer: репозиторії Django ORM, клієнти FX API та LLM API з таймаутами та circuit breaker.

**Контур AppSec (OWASP Top 10:2021, OWASP ASVS 4.0 L2):**

| Загроза | Механізм захисту |
|---|---|
| A01 Broken Access Control / IDOR | Кожен запит фільтрується за `budget_id` поточного учасника на рівні queryset репозиторію; доступ до чужого ресурсу повертає 404 (не 403), щоб не розкривати факт існування |
| A02 Cryptographic Failures | TLS 1.2+; Argon2id для паролів; AES-256-GCM для чутливих полів (адреси криптогаманців); секрети — у змінних середовища, не в репозиторії |
| A03 Injection | Лише параметризовані запити Django ORM; заборона raw SQL правилом Bandit; екранування вихідних даних у Next.js |
| A04 Insecure Design | Інваріант подвійного запису на рівні БД; ідемпотентні ключі; ліміт розміру тіла запиту 1 МБ |
| A05 Security Misconfiguration | `DEBUG=False`, суворий CORS-список доменів, заголовки HSTS, CSP, X-Content-Type-Options |
| A07 Identification & Auth Failures | Access JWT 15 хв, refresh 7 діб з ротацією та blacklist; 10 спроб входу/хв на IP |
| A08 Software & Data Integrity | HMAC-SHA256 для webhook з перевіркою timestamp (вікно 5 хв) проти replay; закріплені версії залежностей; Trivy та Dependabot |
| A09 Logging & Monitoring Failures | Append-only `audit_events`; структуровані логи з trace ID без персональних даних |
| A10 SSRF | Вихідні HTTP-запити лише на allowlist доменів FX і LLM провайдерів |
| LLM Prompt Injection | Асистент отримує лише агреговані дані; виклики інструментів обмежені read-only allowlist; вихід моделі не виконується як код |

## 2.10 Топологія API-маршрутів та потоки даних

Таблиця 2.1 — Карта інтерфейсних маршрутів системи

| ID | Метод | URI | Вхідні параметри / Payload | Призначення та контейнер-обробник |
|---|---|---|---|---|
| API-REQ-01 | POST | `/api/v1/auth/token` | body: `email: string(email)`, `password: string(12..128)` | Видача JWT; Core API (`auth`) |
| API-REQ-02 | POST | `/api/v1/auth/token/refresh` | body: `refresh: string` | Ротація refresh-токена; Core API (`auth`) |
| API-REQ-03 | GET | `/api/v1/wallets` | query: `budget_id: uuid`, `cursor`, `limit ≤ 100` | Список гаманців; Core API (`wallets`) |
| API-REQ-04 | POST | `/api/v1/wallets` | header `Idempotency-Key: uuid`; body `WalletCreate` | Створення гаманця; Core API (`wallets`) |
| API-REQ-05 | GET | `/api/v1/transactions` | query: `budget_id`, `wallet_id`, `from`, `to`, `cursor`, `limit` | Журнал транзакцій; Core API (`ledger`) |
| API-REQ-06 | POST | `/api/v1/transactions` | header `Idempotency-Key`; body `TransactionCreate` (2..50 проводок, `Money`) | Реєстрація транзакції, запис Outbox; Core API (`ledger`) → Async Workers (`jars`) |
| API-REQ-07 | GET | `/api/v1/jars` | query: `budget_id` | Баночки з балансами; Core API (`jars`) |
| API-REQ-08 | PUT | `/api/v1/jars/allocation-rules` | query: `budget_id`; body: масив `AllocationRule` (1..20) | Заміна правил розподілу; Core API (`jars`) |
| API-REQ-09 | POST | `/api/v1/jars/{jarId}/withdrawals` | path `jarId: uuid`; header `Idempotency-Key`; body `JarWithdrawal` | Зняття з баночки з коментарем; Core API (`jars`) |
| API-REQ-10 | GET | `/api/v1/fx/rates` | query: `base`, `quote`, `date` | Курс валют на дату; Core API (`fx`) |
| API-REQ-11 | POST | `/api/v1/assistant/queries` | body: `question: string(1..1000)`, `budget_id` | Запит до LLM-асистента; Core API → Async Workers (`assistant`) |
| API-REQ-12 | POST | `/api/v1/webhooks/bank` | header `X-Signature`; body: `event_id`, `amount`, `currency`, `occurred_at` | Прийом банківської події; Core API (`ledger`) |

Усі мутуючі маршрути (POST, PUT) крім API-REQ-01, API-REQ-02 та API-REQ-12 вимагають `Authorization: Bearer <JWT>`; API-REQ-12 автентифікується підписом HMAC.

**Data Flow: надходження заробітної плати та автоматичний розподіл.**

1. Web App надсилає `POST /api/v1/transactions` (kind `income`, заголовок `Idempotency-Key`) через HTTPS на Nginx.
2. Nginx завершує TLS і передає запит у Core API; middleware перевіряє rate limit у Redis, валідує JWT та роль `owner` або `member` у бюджеті.
3. Серіалізатор валідує DTO; `PostTransactionService` отримує курси з Redis (за відсутності — з таблиці `fx_rates`) та обчислює `amount_base` кожної проводки.
4. В одній ACID-транзакції PostgreSQL записуються `transactions`, `ledger_entries`, `audit_events` та подія `IncomeReceived` в `outbox_events`; клієнт отримує 201.
5. Celery Beat кожні 2 с запускає Outbox-релей, який публікує неопубліковані події в чергу `allocation` RabbitMQ (AMQP) і позначає `published_at`.
6. Allocation Service отримує подію, перевіряє `processed_events`, обчислює частки `share[i]` за правилами та записує проводки розподілу в одній транзакції; повідомлення підтверджується після commit.
7. Web App оновлює баланси баночок запитом `GET /api/v1/jars` (TanStack Query, інвалідація кешу після успішної мутації).

## 2.11 Моделювання користувацького досвіду та транзакційних сценаріїв (Use Cases)

```mermaid
flowchart LR
    U(("Individual User"))
    FM(("Family Member"))
    BA(("Business Accountant"))
    BANK(("Bank Webhook"))
    subgraph FinJar
        UC1(["UC-01 Зареєструвати надходження та автоматично розподілити дохід"])
        UC2(["UC-02 Зняти кошти з баночки"])
        UC3(["UC-03 Налаштувати правила розподілу"])
        UC4(["UC-04 Керувати гаманцями"])
        UC5(["UC-05 Запитати LLM-асистента"])
        UC6(["UC-06 Сформувати звіт"])
        INC1(["Автентифікуватися"])
        INC2(["Конвертувати валюту за курсом"])
        EXT1(["Відхилити дублікат події"])
    end
    U --- UC1
    U --- UC2
    U --- UC3
    U --- UC4
    U --- UC5
    FM --- UC2
    BA --- UC6
    BANK --- UC1
    UC1 -. "include" .-> INC1
    UC2 -. "include" .-> INC1
    UC1 -. "include" .-> INC2
    EXT1 -. "extend" .-> UC1
```

### UC-01: Реєстрація надходження та автоматичний розподіл доходу

| Атрибут | Значення |
|---|---|
| Актори | Individual User (основний), Bank Webhook (вторинний) |
| Передумови | Користувач автентифікований; існує гаманець отримувача; активні правила з сумою часток 100% |
| Тригер | Користувач вносить надходження або банк надсилає webhook |

Таблиця 2.2 — Main Success Scenario UC-01

| Крок | Дія актора | Реакція системи (Backend / БД) |
|---|---|---|
| 1 | Вводить суму 15000.00 UAH, гаманець, дату та натискає Save | Web App надсилає `POST /transactions` з `Idempotency-Key` |
| 2 | — | Core API валідує JWT, роль і DTO; перевіряє унікальність `Idempotency-Key` |
| 3 | — | Обчислює `amount_base`, записує транзакцію, проводки та Outbox-подію в одній транзакції; повертає 201 |
| 4 | Бачить транзакцію в журналі | Outbox-релей публікує `IncomeReceived` у RabbitMQ |
| 5 | — | Allocation Service записує `event_id` у `processed_events` і проводки розподілу (30/10/30/30%) в одній транзакції |
| 6 | Бачить оновлені баланси баночок | `GET /jars` повертає нові баланси |

**Постумови:** у реєстрі зафіксовано транзакцію надходження та транзакцію розподілу; суми дебету та кредиту рівні; записи аудиту створено.

Таблиця 2.3 — Alternative Flows UC-01

| ID | Старт із кроку | Умова | Реакція системи | Повернення |
|---|---|---|---|---|
| AF-01A | Крок 2 | Повторний `Idempotency-Key` з тим самим тілом | Повертає 201 з раніше створеною транзакцією без нового запису | Крок 4 |
| AF-01B | Крок 2 | Повторний `Idempotency-Key` з іншим тілом | Повертає 409 Conflict | Крок 1 |
| AF-01C | Крок 3 | Курс валюти на дату відсутній | Повертає 422 з полем `currency`; транзакцію не створено | Крок 1 |
| AF-01D | Крок 5 | Сума часток правил ≠ 100% після зміни правил | Розподіл не виконується, подія позначається `config_error`, користувач отримує сповіщення | Завершення |

Таблиця 2.4 — Exception Flows UC-01

| ID | Крок | Збій | Реакція системи |
|---|---|---|---|
| EF-01A | Крок 3 | PostgreSQL недоступний | Транзакція не фіксується; 503 з `Retry-After`; клієнт повторює запит з тим самим `Idempotency-Key` |
| EF-01B | Крок 4 | RabbitMQ недоступний | Подія залишається в Outbox з `published_at = NULL`; релей повторює публікацію після відновлення |
| EF-01C | Крок 5 | Воркер аварійно завершився до commit | Повідомлення не підтверджене (`acks_late`) і повторно доставляється; `processed_events` виключає дублювання |
| EF-01D | Крок 5 | 5 невдалих спроб обробки | Повідомлення переміщується в dead-letter queue; алерт Prometheus; ручний розбір |

### UC-02: Зняття коштів з баночки

| Атрибут | Значення |
|---|---|
| Актори | Individual User, Family Member (роль `member`) |
| Передумови | Користувач має роль `owner` або `member`; баланс баночки > 0 |

Таблиця 2.5 — Main Success Scenario UC-02

| Крок | Дія актора | Реакція системи (Backend / БД) |
|---|---|---|
| 1 | Обирає баночку, вводить суму, гаманець і коментар | Web App надсилає `POST /jars/{jarId}/withdrawals` з `Idempotency-Key` |
| 2 | — | Core API перевіряє роль, валідує коментар (1..500 символів) |
| 3 | — | Блокує рядок балансу баночки (`SELECT ... FOR UPDATE`), перевіряє достатність коштів |
| 4 | — | Записує проводки (дебет гаманця, кредит баночки) та аудит; повертає 201 |
| 5 | Бачить новий баланс і коментар в історії | — |

Таблиця 2.6 — Alternative та Exception Flows UC-02

| ID | Тип | Крок | Умова / збій | Реакція системи |
|---|---|---|---|---|
| AF-02A | Alternative | Крок 3 | Недостатньо коштів | 409 з `detail: insufficient_funds`; повернення на крок 1 |
| AF-02B | Alternative | Крок 2 | Роль `viewer` | 403; операція недоступна |
| EF-02A | Exception | Крок 3 | Таймаут блокування рядка (конкурентне зняття) | Відкат транзакції, 503 з `Retry-After: 1` |

## Висновки до розділу 2

У розділі обґрунтовано архітектуру модульного моноліту на Django REST Framework з асинхронним контуром Celery та RabbitMQ, що забезпечує атомарний запис транзакцій подвійного запису та подій Outbox в одній базі PostgreSQL без розподілених саг. Архітектуру змодельовано на трьох рівнях C4 Model з наскрізним трасуванням контейнерів і компонентів на вимоги SRS, а вибір брокера повідомлень зафіксовано в ADR-001 з аналізом компромісів.

Спроєктовано модель даних реєстру з типом `NUMERIC` для сум, унікальними ключами ідемпотентності та планом індексування; специфіковано REST-контракт OpenAPI 3.0.3 з 10 маршрутами, форматом помилок RFC 7807 та обов'язковими ключами ідемпотентності; алгоритмізовано розподіл доходу та реєстрацію мультивалютної транзакції. Контур безпеки охоплює категорії OWASP Top 10, а сценарії Use Case описують основні, альтернативні та виключні потоки з відкатом транзакцій.

Інтеграційний аудит підтвердив узгодженість типів між моделлю даних, API-контрактом та алгоритмами. Архітектурне ядро готове до програмної реалізації, тестування та розгортання, що є предметом Розділу 3.
