# B2B MVP: scope и отношение к текущему masterdocapp

**Дата:** 2026-05-23  
**Стратегия:** [STRATEGY.md](STRATEGY.md)  
**Статус:** продуктовое ТЗ для фазы 1 B2B

---

## Резюме решения

| Трек | Решение |
|------|---------|
| **B2B эксплуатация (приоритет GTM)** | Новый домен в `shared/` + новая навигация в `composeApp/`; не «перекрасить» B2C Atlant |
| **B2C Atlant** | Оставить как **модуль** `consumer/` или flavor; источник AI и поиска по ИЭ |
| **Backend** | Расширить REST (объекты, заявки, журнал); Onyx — для слоя «поиск по документам / подсказка» |

**Принцип MVP:** сначала **журнал + заявка + привязка к оборудованию + документы + отчёты**. Склад ЗИП, кладовщик и подрядчики — **backlog**, не первые релизы.

---

## Текущее состояние masterdocapp (as-is)

```
Root (табы)
├── Chat      → Onyx persona, стриминг, история (B2C FAQ Atlant)
└── Search    → поиск по базе знаний

Отдельно: EquipmentSelection → выбор Assistant (модель/тип оборудования)
```

**Стек:** KMP, Decompose, MVIKotlin, Koin, Ktor → backend/Onyx ([backend/README.md](../backend/README.md)).

**Не покрывает B2B:** объекты сети, заявки, журналы событий, роли, печать/выгрузка, мульти-тенант организации. Учёт ЗИП тоже отсутствует, но он вынесен в backlog и не является требованием первых релизов.

---

## Целевая модель B2B (to-be)

```mermaid
flowchart TB
  Org[Организация]
  Site[Объект / площадка]
  Asset[Единица оборудования]
  Doc[Документы ИД]
  Ticket[Заявка на ремонт]
  Log[Запись журнала]
  Org --> Site
  Site --> Asset
  Asset --> Doc
  Asset --> Ticket
  Asset --> Log
  Ticket --> Log
```

### Сущности фазы 1 (MVP)

| Сущность | Поля (минимум) | Комментарий |
|----------|----------------|-------------|
| **Organization** | id, name | Один тенант = одна сеть |
| **Site** | id, orgId, name, address | Магазин, РЦ, цех |
| **Asset** | id, siteId, name, inventoryNo?, category | Витрина, шкаф, компрессорный |
| **Document** | id, assetId?, siteId?, file, type | ИЭ, паспорт, схема; PDF |
| **WorkOrder** | id, siteId, assetId?, title, status, assignee, createdAt, closedAt | Заявка на ремонт |
| **JournalEntry** | id, type, siteId, assetId?, workOrderId?, text, author, at | ТО / неисправность / устранение |
| **User** | id, role | admin, dispatcher, engineer, requester?, reporter |

### Сущности фазы 2 (не MVP)

- **MaintenancePlan** / PPR calendar — регламенты и графики  
- **ChecklistTemplate** — отраслевые чек-листы осмотра  
- **Integration1C** — выгрузка в 1С:ТОиР

### Backlog (не первые релизы)

- **PartRequest** queue, статус «Ждёт запчасть», soft reserve, min/max auto — FSM-расширение склада  
- Подрядчики / внешний инженер (`contractor`)  

### Фазированный трек: склад ЗИП (v1 — 2026-08-19)

- **SparePart**, **StockMovement**, **AssetPart**, роль `storekeeper`, feature `warehouse`, приход/списание к WO, текстовые рекомендации — см. [docs/superpowers/specs/2026-08-19-warehouse-zip-design.md](docs/superpowers/specs/2026-08-19-warehouse-zip-design.md)  

---

## Функциональный scope MVP

### Must-have (релиз 1)

| # | Функция | Роль | Критерий готовности |
|---|---------|------|---------------------|
| 1 | Справочник объектов и оборудования | dispatcher (+ admin: только типы `EquipmentCategory`) | CRUD объектов и активов; admin — CRUD категорий типов |
| 2 | Заявка на ремонт | engineer, dispatcher | Создание с существующего asset, статусы: новая → в работе → закрыта |
| 3 | Журнал событий | engineer | Запись неисправности/ТО/устранения с привязкой к asset |
| 4 | Документы на оборудовании | dispatcher, engineer | Загрузка PDF — dispatcher; просмотр — engineer |
| 5 | Поиск по документам | engineer | Текстовый поиск; опционально Onyx RAG по document set |
| 6 | Выгрузка журнала и отчёты | reporter | PDF/Excel за период по объекту или сети |
| 7 | Список заявок по сети | dispatcher | Фильтр по объекту, статусу, просрочке |
| 8 | Auth + org scope | all | OIDC Zitadel **self-host РФ** → JWT; identity = `sub`; см. TOIR_AI_SYSTEM_DESIGN §4 (zitadel) и private [`masterdoc-zitadel`](https://github.com/masterdoc-app/masterdoc-zitadel) |
| 9 | Конфигурация системы | admin | Invite, роли, `EquipmentCategory`; фичи org/user — через feature-service |

### Should-have (релиз 1.1, сразу после MVP)

| # | Функция |
|---|---------|
| 10 | Фото к заявке и записи журнала |
| 11 | Push / уведомление диспетчеру о новой заявке |
| 12 | AI-наставник по документации asset |
| 13 | Офлайн-черновик заявки (mobile) |

### Won't-have в MVP

- Полноценный PPR / календарь регламентов  
- Склад ЗИП и списание по наряду  
- Кладовщик, заявки на запчасти, остатки, закупки  
- Подрядчики / contractor  
- Пожарные / лифтовые сертифицированные формы  
- Тендерный функционал 1С:ТОиР  
- B2C сценарий Atlant в том же продукте  

---

## Экраны MVP (composeApp)

| Экран | Описание |
|-------|----------|
| Login | email + пароль через Zitadel self-host (OIDC); регистрация только по invite от admin |
| Home dispatcher | Открытые заявки, счётчики по объектам |
| Sites list | Объекты сети |
| Site detail | Оборудование на объекте + быстрые действия |
| Asset detail | Документы, история журнала, открытые заявки, «спросить AI» |
| Create work order | Форма заявки |
| Journal timeline | Лента записей (фильтр по типу) |
| Documents | Список + viewer |
| Reports / Export | Выбор периода, объекта, статуса, инженера → PDF/Excel |

**Навигация:** заменить B2C-табы `Chat | Search` на B2B: `Заявки | Объекты | Журнал` (+ профиль). Чат — **внутри карточки оборудования**, не главный таб.

---

## Backend MVP

Архитектура: **микросервисы** (см. [TOIR_AI_SYSTEM_DESIGN.md](TOIR_AI_SYSTEM_DESIGN.md) §4). Клиент знает **один API base** — `https://api.masterdoc.pro` (api-gateway). Login — `https://auth.formaverse.ru`. Сервисы за gateway общаются через REST (sync) и NATS (async).

### Сервисы фазы 1 (MVP)

| Сервис | Endpoint'ы |
|--------|-------------|
| api-gateway | публично: `GET /health`, `GET /me`, `/v1/*` → backend; blue-green |
| feature-service | внутренний `GET /me` → `userInfo` + фичи (dispatcher → `board`; engineer → `copilot`) |
| catalog-service | `GET/POST /sites`, `GET/POST /assets`, `GET/POST /equipment-categories`, `POST /assets/from-documents` (Технолог: доки + оборудование → карточки автоматически) |
| dashboard-service | `GET/POST /work-orders`, `PATCH /work-orders/{id}/status`, `GET/POST /journal-entries` |
| document-service | `POST /documents`, `GET /documents/{id}`, `GET /assets/{id}/documents` |
| report-service | `GET /export/journal` (stub; read-replica dashboard_db на старте) |

### Сервисы фазы 1.1+

| Сервис | Endpoint'ы |
|--------|-------------|
| ai-gateway | `POST /ai/intake`, `POST /ai/mentor`, `POST /ai/scribe`, `POST /ai/technologist` (доки + оборудование → карточки автоматически) |
| search-service | `GET /search/docs` |
| notification-service | push по событиям (без публичного API в MVP) |

### Сервисы фазы 2 (расширение dashboard-service, без нового deployable)

| Сервис | Endpoint'ы |
|--------|-------------|
| dashboard-service | `GET/POST /maintenance-plans`, `GET/POST /checklists`, `GET /work-orders/calendar?type=preventive`; `WorkOrder.type`: `corrective` \| `preventive` |
| ai-gateway | `POST /ai/technologist` → карточки автоматически (Asset + план + checklist + preventive WO) в catalog/dashboard; шильдик не вход |

### Инфраструктура

| Компонент | Назначение |
|-----------|------------|
| API Gateway | `api-gateway-service` (Ktor): единственный `https://api.masterdoc.pro`; JWT на `/me`; blue-green |
| PostgreSQL | schema per service (один кластер на MVP) |
| NATS JetStream | domain events |
| MinIO / S3 | PDF, фото |
| Onyx | RAG-индексы (search-service) |

---

## Что переиспользовать из текущего кода

| Компонент | Переиспользование |
|-----------|-------------------|
| `ChatApi`, `HttpChatRepository`, стриминг | AI-наставник на карточке asset |
| `EquipmentSelection` / `Assistant` | Модель «тип оборудования» → seed категорий asset |
| Decompose + MVIKotlin + Koin | Архитектура новых фич |
| `MasterdocTheme`, UI kit | Визуальная целостность |
| Onyx document sets | Индексация ИЭ по asset/site |
| Wasm/Android/iOS targets | Те же платформы; приоритет **Android + Web** для инженера |

---

## Структура модулей (рекомендация)

```
shared/
  domain/
    consumer/     # B2C: chat, assistant
    facility/     # B2B: site, asset, workorder, journal
  data/
    consumer/
    facility/
  presentation/
    consumer/
    facility/

composeApp/
  ui/
    consumer/       # Atlant flow (опционально отключаемый)
    facility/       # B2B screens
```

**Сборка:** product flavor `consumer` | `facility` или feature flag `BuildConfig.B2B_MODE` для одного бинарника на пилот.

---

## Roadmap по фазам

| Фаза | Срок (ориентир) | Содержание |
|------|-----------------|------------|
| **0** | 1–2 нед | Документы стратегии ✅, custdev 5 интервью, лендинг SEO |
| **1 MVP** | 6–8 нед | Сущности, API, экраны §, выгрузка журнала, 1 пилотный клиент |
| **1.1** | +3 нед | Фото, push, AI в asset |
| **2** | +6 нед | MaintenancePlan, чек-листы, календарь ТО, плановые заявки из документации |
| **3** | +8 нед | Расширенные отчёты, интеграция 1С (если custdev подтвердит) |
| **Backlog** | — | Склад ЗИП, кладовщик, contractor |

---

## Критерии «MVP готов к пилоту»

1. Один объект: ≥10 единиц оборудования, ≥3 документа.  
2. Инженер создаёт заявку и запись журнала с телефона &lt; 2 мин.  
3. Руководитель выгружает журнал за месяц одним действием.  
4. Диспетчер видит все открытые заявки сети.  
5. `./gradlew check` зелёный; CI все платформы по [AGENTS.md](../masterdocapp/AGENTS.md).

---

## Риски реализации

| Риск | Митигация |
|------|-----------|
| Смешение B2C/B2C в одном UX | Flavor или отдельный applicationId для B2B пилота |
| Scope creep (ЗИП, contractor) | Жёсткий backlog; не включать в первые релизы |
| Нет серверной БД | Backend milestone 0 до UI заявок |
| Custdev опровергнет холод | Универсальные журналы в SEO остаются; ICP скорректировать в STRATEGY |

---

## Связь с лендингом

На лендинге обещать только **Must-have MVP** ([SEO_KEYWORDS.md](SEO_KEYWORDS.md) — без «склад ЗИП» в H1). После релиза 1.1 — добавить блок AI и фото.
