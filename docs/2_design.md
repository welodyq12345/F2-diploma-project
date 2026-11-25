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
