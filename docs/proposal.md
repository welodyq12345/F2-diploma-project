# Thesis Proposal: FinJar

## 1. Робоча назва проєкту

- **UA:** Проєктування та розробка мультивалютної платформи управління персональними та корпоративними фінансами з підтримкою гетерогенних гаманців, автоматизованим розподілом коштів за бюджетними категоріями та LLM-асистентом
- **EN:** Design and Development of a Multi-Currency Personal and Business Finance Management Platform with Heterogeneous Wallets, Rule-Based Fund Allocation and an LLM-Powered Assistant

Кодова назва продукту: **FinJar**.

## 2. Актуальність та проблематика

Користувач одночасно оперує коштами в кількох типах сховищ: дебетові та кредитні банківські картки, криптовалютні гаманці, готівка в національній та іноземних валютах. Наявні засоби обліку або прив'язані до екосистеми одного банку, або підтримують кілька валют лише як окремі ізольовані рахунки без єдиного балансу в базовій валюті.

Інженерна складність задачі визначається такими чинниками:

- **Консистентність фінансових даних.** Кожна операція змінює баланс кількох сутностей (гаманець, бюджетна категорія, кредитний ліміт). Без подвійного запису (double-entry ledger) та атомарних транзакцій виникають розбіжності залишків.
- **Мультивалютність.** Конвертація потребує історичних курсів на момент операції, фіксації курсу в записі транзакції та коректного округлення (Decimal, а не float).
- **Автоматизований розподіл доходу.** Надходження коштів має ініціювати розподіл суми між бюджетними категоріями («баночками») за процентними правилами. Операція повинна бути ідемпотентною та стійкою до повторної доставки подій.
- **Багатокористувацький доступ.** Сімейні та корпоративні бюджети вимагають рольової моделі доступу (RBAC) та журналу аудиту змін.
- **Інтеграція LLM.** ШІ-асистент обробляє чутливі фінансові дані, тому потрібні ізоляція контексту користувача, обмеження доступу асистента лише до read-only API та контроль витрат на запити до моделі.

## 3. Сформована ідея продукту

**Тип системи:** клієнт-серверна вебплатформа з архітектурою модульного моноліту на бекенді, асинхронною подієвою обробкою фонових задач та виділеним сервісом LLM-асистента.

- Frontend: SPA/SSR-застосунок на Next.js (TypeScript, Tailwind CSS), мова інтерфейсу — англійська.
- Backend: Django + Django REST Framework (REST API), доменні модулі `wallets`, `ledger`, `fx`, `jars`, `billing`, `assistant`.
- Асинхронний контур: Celery-воркери з брокером повідомлень RabbitMQ для розподілу коштів, оновлення курсів валют та генерації звітів.
- Монетизація: тарифні плани Free, Pro (спільний сімейний бюджет) та Business (облік для підприємств: план рахунків, рахунки-фактури, звіт P&L).

## 4. Ключові функції (Core Features)

1. **Гетерогенні гаманці.** Прив'язка до одного облікового запису гаманців типів `bank_card`, `credit_card` (кредитний ліміт, заборгованість, пільговий період), `crypto_wallet` та `cash` з довільною валютою кожного гаманця.
2. **Мультивалютний реєстр операцій.** Подвійний запис кожної транзакції з фіксацією курсу на момент операції; автоматичне отримання курсів через зовнішні FX API; зведений баланс у базовій валюті користувача.
3. **Автоматизований розподіл доходу за баночками.** Налаштовувані процентні правила (наприклад, 30% — харчування, 10% — податки, 30% — заощадження, 30% — дозвілля), які застосовуються до вхідного платежу автоматично; ідемпотентна обробка подій розподілу.
4. **Операції з баночками.** Зняття та переказ коштів між баночками й гаманцями з обов'язковим коментарем; незмінний журнал аудиту (append-only audit log).
5. **LLM-асистент.** Відповіді на запити природною мовою щодо фінансового стану, автоматична категоризація витрат, прогноз залишків на кінець місяця; доступ асистента лише до read-only інструментів API (tool calling) у межах даних поточного користувача.
6. **Рівні підписки та RBAC.** Тарифи Free / Pro / Business; у Pro — спільний сімейний бюджет із ролями `owner`, `member`, `viewer`; у Business — облік за планом рахунків, рахунки-фактури, звіти P&L та експорт для бухгалтерії.
7. **Аналітика та звітність.** Дашборди витрат за категоріями, валютами та гаманцями; фонова генерація звітів у CSV/PDF.

## 5. Технологічний стек

| Категорія | Технології |
|---|---|
| Мови програмування | Python 3.12, TypeScript 5 |
| Frontend | Next.js 15 (App Router), React, Tailwind CSS, TanStack Query, Zod |
| Backend | Django 5, Django REST Framework, drf-spectacular (OpenAPI 3), SimpleJWT |
| Асинхронна обробка | Celery, RabbitMQ (AMQP), Celery Beat (планувальник оновлення курсів) |
| Бази даних та кеш | PostgreSQL 16 (ACID, `NUMERIC` для грошових сум), Redis (кеш курсів, rate limiting, result backend) |
| Архітектурні патерни | Modular Monolith, Double-Entry Ledger, Transactional Outbox, Idempotent Consumer, Repository/Service Layer, RBAC |
| LLM-інтеграція | LLM API з tool calling, RAG по агрегатах транзакцій, pgvector |
| Безпека | OAuth 2.0 / JWT, OWASP ASVS, шифрування чутливих полів, Rate Limiting |
| DevOps та якість | Docker, Docker Compose, GitHub Actions (CI/CD), pytest, Jest/Playwright, SonarQube, Trivy, k6 |
| Спостережуваність | OpenTelemetry, Prometheus, Grafana, Sentry |

## 6. English Summary

FinJar is a client-server web platform for unified management of funds stored in heterogeneous wallets (bank cards, credit cards, crypto wallets and multi-currency cash). The backend is a Django REST Framework modular monolith with a double-entry multi-currency ledger, rule-based income allocation into budget jars processed asynchronously by Celery workers over RabbitMQ, and an LLM assistant restricted to read-only tool calls. The frontend is a Next.js (TypeScript, Tailwind CSS) application with an English-language UI. Monetization is based on Free, Pro (shared family budget with RBAC) and Business (accounting-grade bookkeeping) subscription tiers. The delivery pipeline includes Docker, GitHub Actions, SonarQube, Trivy, k6 and an OpenTelemetry/Prometheus/Grafana observability stack.
