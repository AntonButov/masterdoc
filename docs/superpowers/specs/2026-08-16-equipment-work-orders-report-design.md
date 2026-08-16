# Детальный отчёт: заявки по оборудованию

**Date:** 2026-08-16  
**Status:** approved for planning  
**Repos:** `dashboard-service`, `client-app`, `api-gateway-service`  
**Approach:** отдельный `GET /reports/equipment-work-orders` + карточка заявки только для чтения

## Problem

Каталог отчётов даёт агрегаты (KPI, простои, частота отказов), но менеджер не может открыть **историю заявок по одной единице оборудования**. Нужен отчёт: выбрать станок → увидеть все заявки за период списком → открыть карточку.

## Goals

1. Добавить в каталог «Отчёты» пункт **«Детальный отчёт»** (заявки по единице оборудования).
2. На экране: период **7 / 30 / 90** дней (как у остальных отчётов), выбор оборудования **по названию**, список заявок за период.
3. Тап по строке открывает существующую карточку заявки **только для чтения**.
4. Backend: новый endpoint под `/reports` (фича «Отчёты» / admin). Gateway-прокси `/reports` не менять.
5. UI: имена оборудования и заголовки заявок, **никогда** raw id (workspace rule).

## Non-goals

- PDF / Excel export
- Мультивыбор оборудования, сравнение станков
- Редактирование, назначение, смена статуса из отчёта
- Новый экран карточки заявки (только `WorkOrderDetailScreen` + `readOnly = true`)
- Менять формулы и shape существующих отчётов
- Комментарии и фото в карточке, если у роли нет фич board/engineer/tickets

## UX

### Catalog

Новый пункт **в конце** текущего списка из 10 отчётов:

| `ReportId` | Title | Subtitle |
|---|---|---|
| `EquipmentWorkOrders` | Детальный отчёт | Заявки по единице оборудования |

Справка внизу экрана детали: «По выбранному оборудованию показывает все заявки, которые пересекают выбранный период: открытые и закрытые. Нажмите строку, чтобы открыть карточку.»

### Detail

1. Селектор периода 7 / 30 / 90.
2. Выбор оборудования: выпадающий список **имён** (`displayName` / name, иначе «Оборудование»). Пока не выбрано — список заявок не грузится, текст «Выберите оборудование».
3. После выбора — список заявок, новые сверху:
   - название (`formatWorkOrderDisplayTitle`)
   - тип · срок (`workOrderTypeLabelRu` · `dueAt`)
   - статус (`AppStatusChip` + `workOrderStatusLabelRu`)
4. Пусто: «Нет заявок по этому оборудованию за выбранный период».
5. Тап → `WorkOrderDetailScreen(readOnly = true)`. Назад → снова список этого отчёта (не каталог).
6. Состояния: loading / error (текст) / empty / content. Каталог всегда рисуется.

## Inclusion rule (period overlap)

Заявка попадает в отчёт, если **жизнь пересекает** `[from, to]`:

- `start` = `createdAt`
- `end` = `closedAt`, если закрыта; иначе конец периода `to` (открытая заявка, созданная до `to`, всегда пересекает окно)
- включить, если `start <= to` **и** `end >= from`

Исключить заявку с неразборчивым `createdAt`.  
`assetId` обязателен; без него — `400`.  
Пустой список — валидный `200`.

Сортировка: `createdAt` desc, затем `id` desc.

## API

### `GET /reports/equipment-work-orders?assetId=&from=&to=`

Фича: `reports` или `admin` (существующий proxy `/reports`).  
`from` / `to`: как у других report routes (`YYYY-MM-DD` или ISO).

Ответ: JSON-массив объектов **того же shape**, что `GET /work-orders` (`WorkOrder`), чтобы клиент переиспользовал `WorkOrderDto`.

Не включать заявки других `assetId`. Не фильтровать по assignee/createdBy.

### Карточка заявки

`WorkOrderDetailScreen` по-прежнему делает `GET /work-orders/{id}`.

Gateway: разрешить фиче `reports` **только** `GET /work-orders/{id}` (отдельный маршрут до prefix, либо сужение read-gate).  
**Не** открывать `GET /work-orders` (список), `GET /work-orders/board` и любые POST/PATCH для роли только «Отчёты».  
`writeFeatures` prefix `/work-orders` не менять.

Комментарии и вложения: проксируются только с board/engineer/tickets. Из отчёта передавать репозитории, если они уже есть в shell; при 403 карточка остаётся читаемой без фото/комментов.

## Client

- `ReportId.EquipmentWorkOrders` + строка в `reportCatalogItems()`.
- `WorkOrdersRepository.equipmentWorkOrders(assetId, from, to)`.
- `ReportsScreen`: для этого id — picker оборудования, без Gantt/KPI/чартов; период не грузит заявки до выбора `assetId`.
- Прокинуть в `ReportsScreen` то, что нужно для read-only карточки: `attachmentsRepository` / `commentsRepository` опционально, `currentUserId` не обязателен для просмотра.

## Error handling

| Ситуация | UI |
|---|---|
| Нет списка оборудования | сообщение ошибки загрузки оборудования |
| Оборудование не выбрано | «Выберите оборудование» |
| 400 / сеть / 5xx отчёта | текст ошибки, без падения shell |
| Пустой список после выбора | empty-state выше |
| 403 на GET заявки в карточке | ошибка на карточке, назад в список |

## Testing

- `ReportCatalogTest`: 11 пунктов, последний — `EquipmentWorkOrders`, непустая справка.
- Dashboard: overlap — открытая старая (входит), закрытая внутри периода (входит), закрытая до `from` (нет), созданная после `to` (нет), другой `assetId` (нет); без `assetId` → 400; сортировка desc по `createdAt`.
- Client repository: URL `/reports/equipment-work-orders?assetId=&from=&to=`.
- Gateway: `GET /work-orders/{id}` разрешён с фичей `reports`; список, board и `PATCH` для только-reports по-прежнему запрещены.
- Focused unit/route tests; без локальной Wasm/full Gradle distribution.

## Success criteria

1. Менеджер с фичей «Отчёты» видит «Детальный отчёт» в каталоге.
2. Выбирает период и оборудование по **имени** → видит все заявки, пересекающие период.
3. Тап открывает карточку только для чтения; «Назад» возвращает в список отчёта.
4. Роль без board/engineer/tickets не может менять заявку из этого экрана.
