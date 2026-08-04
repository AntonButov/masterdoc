# Manager reports — market leaders pack (B)

**Date:** 2026-08-04  
**Status:** approved for planning  
**Repos:** `dashboard-service`, `client-app` (gateway `/reports` proxy unchanged)  
**Approach:** separate `GET /reports/...` endpoint per new report (option 2)

## Problem

Каталог из 6 отчётов закрывает базу (MTTR/MTBF, planned vs emergency, ППР, backlog, downtime), но не дотягивает до набора Fiix / UpKeep / MaintainX: нет **трендов**, явного **% реактивных / completion**, **нагрузки инженеров** и **частоты отказов** по оборудованию. Demo-org после seed должен показывать **качественные, не плоские** графики.

## Goals

1. Добавить **4** новых отчёта в каталог `client-app` с периодом 7 / 30 / 90 дней и блоком «Справка» внизу.
2. Для каждого — **свой** публичный endpoint в `dashboard-service` под префиксом `/reports` (отдельные контракты, без раздувания `manager-kpis`).
3. Расширить `seed-manager-reports` так, чтобы в **Fixaverse Demo** (`382715225649971203`) графики были выразительными (несколько инженеров, волнистые тренды, разная нагрузка/частота отказов). Smoke-org тоже должен получать совместимый seed (не пустые ответы).
4. UI: имена людей и оборудования, **никогда** raw id (workspace rule).

## Non-goals

- Costs, OEE, FTR, PDF/export, кастомные дашборды
- Ломать существующие 6 отчётов и shape `GET /reports/manager-kpis` / `equipment-downtime`
- Новые фичи gateway (прокси `/reports` уже есть)

## Catalog (new items)

| `ReportId` | Title | Subtitle | API |
|---|---|---|---|
| `KpiTrends` | Динамика KPI | MTTR, MTBF, готовность во времени | `GET /reports/kpi-trends?from&to` |
| `ReactiveCompletion` | Реактивность и закрытие | % аварийных и доля закрытых | `GET /reports/reactive-completion?from&to` |
| `EngineerWorkload` | Нагрузка инженеров | Заявки и часы по людям | `GET /reports/engineer-workload?from&to` |
| `FailureFrequency` | Частота отказов | Топ оборудования по авариям | `GET /reports/failure-frequency?from&to` |

Существующие 6 пунктов остаются без изменения порядка вверху; новые — **после** них (или отдельной группой «Аналитика» без отдельного UI-section, если проще — просто append в список).

## API contracts

Общие query: `from`, `to` — как у текущих report routes (`YYYY-MM-DD` или ISO instant).

### `GET /reports/kpi-trends`

```json
{
  "bucket": "day|week",
  "points": [
    { "bucketStart": "2026-07-01", "mttrHours": 4.5, "mttrSampleSize": 3, "mtbfHours": 80.0, "mtbfSampleSize": 3, "availabilityPercent": 92.1 }
  ]
}
```

- Период ≤ 30 дней → `bucket=day`; 90 дней → `bucket=week` (сервер выбирает по длине интервала).
- Формулы MTTR/MTBF/availability — те же семейства, что в `computeManagerKpis`, но **per bucket**.
- `sampleSize == 0` → клиент рисует gap / `н/д`, не врёт нулём.

### `GET /reports/reactive-completion`

```json
{
  "createdCount": 120,
  "closedCount": 95,
  "completionRatePercent": 79.2,
  "emergencyCount": 40,
  "plannedCount": 80,
  "reactivePercent": 33.3
}
```

- **Created** / **closed** — заявки, созданные / закрытые в окне `[from, to]` (границы как у KPI).
- **Reactive %** = `emergencyCount / (emergencyCount + plannedCount) * 100` среди созданных в окне (0 знаменатель → `н/д` на UI).
- Completion = `closedCount / createdCount` (0 created → `н/д`).

### `GET /reports/engineer-workload`

```json
{
  "engineers": [
    { "userId": "…", "closedCount": 12, "hours": 34.5 }
  ]
}
```

- Группировка по `assigneeId` среди заявок, **закрытых** в периоде (часы = сумма длительностей `startedAt…closedAt` где возможно; иначе 0 с ненулевым count).
- Без assignee — отдельный бакет **не** отдаём в список (или один `unassigned` только если нужно для отладки — **не** для Demo UI).
- Клиент резолвит `userId` → display name / email; fallback «Инженер».

### `GET /reports/failure-frequency`

```json
{
  "assets": [
    { "assetId": "…", "emergencyCount": 7 }
  ]
}
```

- Считаем **аварийные** заявки, созданные (или начатые — зафиксировать в плане: **created in window**) в периоде.
- Топ N (например 15), сортировка по `emergencyCount` desc.
- Клиент: asset display name; fallback «Оборудование».

## UI

- Тот же master → detail, период 7/30/90, loading / error / empty / content.
- **Динамика KPI:** multi-series column или line (существующие chart-примитивы; line — если уже есть / минимально добавить).
- **Реактивность и закрытие:** metric rows + 1–2 простых бара (reactive share, completion).
- **Нагрузка инженеров:** horizontal bars по имени.
- **Частота отказов:** horizontal bars по имени оборудования.
- Внизу detail — `ReportHelpFooter` с description из каталога.

## Seed (Demo quality bar)

Расширить `ManagerReportsSeed` / workflow `seed-manager-reports`:

1. Несколько стабильных `assigneeId` (smoke/demo инженеры org), разная плотность закрытий.
2. Волнистый профиль аварий vs плановых по дням (не константа).
3. Несколько ассетов с явным «лидером» по отказам + длинный хвост.
4. Достаточно closed WOs с `startedAt`/`closedAt`, чтобы MTTR/MTBF per bucket не были пустыми на 30/90д.
5. После seed: Demo каталог открывает все 4 новых отчёта с ненулевыми сериями на 30 дней.

Ops: прогон seed на Demo после merge (как в `fixaverse-demo-org` rule).

## Testing

- Unit: bucket boundaries, reactive/completion edge (zeros), top-N sort.
- Route tests: 4 new GETs happy path + empty org.
- Client: catalog count includes 4; formatters; empty states.
- Smoke (после CI): Demo или Smoke — открыть 2 новых отчёта, визуально не пустые графики (org Smoke для agent smoke; Demo — ручной/ops check качества).

## Implementation order (for plan)

1. dashboard: DTOs + compute + routes + tests  
2. seed enrichment + workflow verify on Demo  
3. client: API models + catalog + 4 detail branches + charts  
4. commit/push each repo → green CI → smoke  

## Open decisions (resolve in plan if needed)

- Failure frequency: count by **createdAt** vs **startedAt** — default **createdAt**.
- Engineer hours without `startedAt`: count-only bar still shown.
- Exact chart type for trends (line vs column) — prefer column if line not ready.
