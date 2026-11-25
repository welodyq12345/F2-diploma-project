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
