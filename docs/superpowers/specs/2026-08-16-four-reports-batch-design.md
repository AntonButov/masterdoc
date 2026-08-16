# Четыре отчёта: площадка, реакция, ППР план/факт, закрытия без фото

**Date:** 2026-08-16  
**Status:** approved (product decisions by best practices; user OK to implement remaining four)  
**Repos:** `dashboard-service`, `client-app` (gateway `/reports` без изменений)  
**Card:** existing `WorkOrderDetailScreen(readOnly = true, allowMediaMutations = false)`

## Shared rules

- UI: never raw ids; titles via `formatWorkOrderDisplayTitle`; sites/assets by name; assignee via `formatAssigneeLabel` / «не назначен».
- Error copy: fixed «Не удалось загрузить отчёт» / «Не удалось загрузить …».
- Empty list → `200 []`. Period `from`/`to` as other reports (`YYYY-MM-DD` / ISO via `parseReportBoundary`).
- Catalog: append **in this order** after «Просроченные» (12 → 16 items).
- Gateway: no new work.

---

## 1. По площадке

**Problem:** детальный отчёт есть по оборудованию; нужен тот же список по цеху.

**API:** `GET /reports/site-work-orders?siteId=&from=&to=` → `WorkOrder[]`  
**Rule:** `siteId` match; lifetime overlap same as equipment report (`createdAt`…`closedAt ?: to`); drop bad `createdAt`; bad non-null `closedAt` exclude; sort `createdAt` desc, `id` desc. Missing `siteId` → 400.

**UX:** `ReportId.SiteWorkOrders` · **«По площадке»** · «Заявки по цеху»  
Period 7/30/90 + site picker by name; empty until site selected («Выберите площадку»).  
Справка: «По выбранной площадке показывает заявки, пересекающие период. Нажмите строку, чтобы открыть карточку.»

---

## 2. Время реакции

**Problem:** не видно, где заявки долго висят в `new` до первого `in_progress`.

**API:** `GET /reports/time-to-first-action?from=&to=` → `WorkOrder[]`  
**Rule:** include if `createdAt` in `[from, to]` (parseable); sort: без `startedAt` первыми (ещё не взяли), затем по `(startedAt − createdAt)` desc, затем `id` desc.

**UX:** `ReportId.TimeToFirstAction` · **«Время реакции»** · «До перевода в работу»  
Period 7/30/90; row: title · type · dueAt · status · secondary line: «ещё не в работе» или «N ч до работы» (client computes from `createdAt`/`startedAt`).  
Пусто: «Нет заявок за выбранный период».

---

## 3. ППР: план и факт

**Problem:** KPI «Выполнение ППР» — агрегаты; нужен список пунктов с исходом.

**API:** `GET /reports/ppr-plan-fact?from=&to=` → `WorkOrder[]`  
**Rule:** `type == ppr`; `dueAt` date in `[fromDate, toDate]` (UTC dates from boundaries); drop bad `dueAt`; sort `dueAt` asc, `id` asc.

**UX:** `ReportId.PprPlanFact` · **«ППР: план и факт»** · «Пункты ТО за период»  
Period 7/30/90; row: title · dueAt · status chip · outcome label client-side:  
- open + due &lt; today → «Просрочен»  
- open + due ≥ today → «Ожидает»  
- closed + closedAt date ≤ dueAt → «Вовремя»  
- closed + closedAt date &gt; dueAt → «С опозданием»  
- closed without parseable closedAt → «Закрыт»  
Пусто: «Нет ППР с сроком в выбранном периоде».

---

## 4. Закрытия без фото

**Problem:** контроль качества закрытия; комментарии в отдельном сервисе — **v1 только фото** (`attachmentIds`).

**API:** `GET /reports/closures-without-photos?from=&to=` → `WorkOrder[]`  
**Rule:** `status == closed`; `closedAt` in `[from, to]` (parseable); `attachmentIds` empty; sort `closedAt` desc, `id` desc. Drop bad `closedAt`.

**UX:** `ReportId.ClosuresWithoutPhotos` · **«Закрытия без фото»** · «Закрыты без вложений»  
Period 7/30/90; row like overdue (title · type · dueAt · status · assignee).  
Пусто: «Нет закрытий без фото за выбранный период».  
Справка явно: «Учитываются только вложения (фото). Комментарии в v1 не проверяются.»

---

## Non-goals (all four)

- PDF/Excel, multi-select, editing from report  
- Cross-service comment counts in v1  
- Changing existing KPI formulas  
- Gateway ACL changes  

## Testing / ship

Each report: dashboard unit+route → PR merge → deploy → API smoke; then client catalog+UI → PR merge → deploy → UI smoke (Fixaverse Smoke).
