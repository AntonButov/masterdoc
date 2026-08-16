# Отчёт: просроченные открытые заявки

**Date:** 2026-08-16  
**Status:** approved for planning  
**Repos:** `dashboard-service`, `client-app` (gateway `/reports` уже проксирует; отдельный gateway-таск не нужен)  
**Approach:** `GET /reports/overdue-open-work-orders` + список + карточка read-only

## Problem

Агрегат «Очередь заявок» показывает счётчики возраста/просрочек, но диспетчер не видит **список** открытых просроченных заявок с переходом в карточку. Нужен actionable бэклог «что уже просрочено прямо сейчас».

## Goals

1. Каталог «Отчёты»: пункт **«Просроченные»** (открытые с истёкшим сроком) — **в конце** списка.
2. Экран: список открытых заявок с `dueAt` < сегодня; без селектора 7/30/90 (снимок «сейчас»).
3. Строка: название, тип · срок, статус, исполнитель **по имени** (не id).
4. Тап → существующая `WorkOrderDetailScreen(readOnly = true, allowMediaMutations = false)`.
5. Backend: `GET /reports/overdue-open-work-orders` под фичей `reports` / `admin`.

## Non-goals

- Период 7/30/90
- Фильтр по площадке / инженеру / оборудованию (v1)
- PDF / Excel
- Редактирование из отчёта
- Менять формулу KPI «Очередь заявок»

## Inclusion rule

Включить заявку, если:

1. `status` ∈ `{new, in_progress}` (не `closed`)
2. `dueAt` парсится как `YYYY-MM-DD` и **строго меньше** «сегодня» (UTC `LocalDate`, как в `ManagerKpis` / backlog overdue)
3. Неразборчивый `dueAt` — **исключить**

Сортировка: `dueAt` asc, затем `id` asc (дольше просроченные сверху).  
Пустой список — `200 []`.

## API

### `GET /reports/overdue-open-work-orders`

Без query-параметров периода.  
Ответ: JSON-массив `WorkOrder` (тот же shape, что `GET /work-orders`).

## UX

| `ReportId` | Title | Subtitle |
|---|---|---|
| `OverdueOpenWorkOrders` | Просроченные | Открытые с истёкшим сроком |

Справка: «Показывает открытые заявки, у которых срок уже прошёл. Нажмите строку, чтобы открыть карточку.»

Пусто: «Нет просроченных открытых заявок».  
Ошибка: фиксированная строка «Не удалось загрузить отчёт» (не `e.message`).

Исполнитель в строке: `formatAssigneeLabel` / «не назначен» если `assigneeId` null; никогда raw id.

## Testing

- Unit: filter / sort / drop bad dueAt / exclude closed
- Route: 200 list + empty `[]`
- Client: catalog 12 items; repository URL; UI smoke на Fixaverse Smoke

## Out of scope for gateway

`GET /work-orders/{id}` для `reports` уже разрешён предыдущей фичей. Новый route только под `/reports/...`.
