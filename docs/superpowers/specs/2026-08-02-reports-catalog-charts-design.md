# Reports catalog + charts design

**Date:** 2026-08-02  
**Status:** approved for planning  
**Repos:** `client-app`, `dashboard-service` (+ ops seed workflow)

## Problem

Экран «Отчёты» выглядит несерьёзно: все KPI свалены в одну ленту текстом, числа с лишними знаками после запятой (`Double.toString()`), графиков нет. Нужен каталог отдельных отчётов, профессиональное форматирование, красивые чарты и seed-данные для smoke-org.

## Goals

1. Каждый отчёт выбирается отдельно из списка (master → detail с «Назад»).
2. Числа: 1 знак для часов и %, целые для счётчиков; `н/д` при `sampleSize = 0`.
3. Красивые графики через **Vico** (Compose Multiplatform).
4. Seed реальных work orders в API для **Fixaverse Smoke** (`383177088934346755`), чтобы KPI/Gantt не были пустыми.
5. Не менять формулы KPI и shape `GET /reports/manager-kpis` / `equipment-downtime`.

## Non-goals

- OEE, cost, FTR, PDF/export
- Новые KPI endpoints или смена формул
- Client-side mock data при пустом API
- Deep links на отдельные отчёты (можно позже)

## UX

### Catalog (`ReportsScreen`)

Всегда показывает 6 пунктов (не зависит от данных):

| Id | Title |
|----|--------|
| `kpi_summary` | Сводка KPI |
| `planned_vs_emergency` | Плановые vs аварийные |
| `ppr_compliance` | Выполнение ППР |
| `backlog` | Очередь заявок |
| `downtime_ranking` | Рейтинг простоев |
| `equipment_downtime` | Простои оборудования |

Тап → полный экран отчёта. Период **не** в каталоге.

### Detail

- Title = название отчёта; кнопка «Назад» → каталог.
- Период: 7 / 30 / 90 дней на detail.
- Состояния: loading / error (retry) / empty («нет данных за период») / content.
- Gantt (`equipment_downtime`) остаётся текущим timeline; остальные 5 — Vico + metric rows.
- UI copy: имена оборудования, никогда raw UUID (workspace rule).

### Number formatting

| Metric | Format |
|--------|--------|
| Hours (MTTR, MTBF, downtime, planned/emergency hours) | 1 decimal, comma: `4,5 ч` |
| Availability % | 1 decimal: `92,1%` |
| Counts | integers |
| `sampleSize == 0` | `н/д` |

Единые helpers: `formatHours`, `formatPercent` (заменить raw `toString().replace`).

### Charts (Vico)

| Report | Visualization |
|--------|----------------|
| Сводка KPI | 3 metric cards (MTTR, MTBF, готовность) + simple column comparing those three scaled values (or hours/% labels under bars) |
| Плановые vs аварийные | Grouped/stacked column: counts and/or hours |
| ППР | Column: onTime / late / openOverdue / openPending |
| Очередь | Column: &lt;7d / 7–30d / &gt;30d (+ overdue callout) |
| Рейтинг простоев | Horizontal bar, top N, asset **names** |
| Простои оборудования | Existing Gantt (no Vico) |

Practices: one report = one focus; chart + clear legend; period label visible; empty-state text instead of empty axes.

## Architecture

### Client (`client-app`)

- Local back-stack inside Reports nav destination: `Catalog | Detail(reportId)`.
- Load data **only on detail** (manager-kpis and/or equipment-downtime as needed).
- Pure modules: catalog items, formatters, DTO → Vico series mappers.
- Dependency: Vico in `composeApp` commonMain (version pinned in version catalog).

### Backend (`dashboard-service`)

- Keep existing report routes and `computeManagerKpis`.
- Seed path (prefer thin internal endpoint + GH workflow_dispatch, pattern of catalog `seed-demo-assets`):
  1. Resolve/use smoke org assets (or clear+create WOs against known asset ids from catalog).
  2. `clearOrg` work orders for org (existing internal delete).
  3. POST a deterministic set of emergency + ppr + backlog WOs covering all KPI buckets and Gantt open intervals.
- Default org for workflow: Smoke `383177088934346755` (overridable `org_id` input).

### Seed content (minimum)

- Closed emergency WOs with varied durations → MTTR, ranking, availability
- ≥2 closed emergencies on one asset → MTBF sample
- PPR: on-time, late, open overdue, open pending (dueAt in period)
- Open backlog across age buckets + overdue
- Open emergency with `startedAt` → Gantt «в работе»

## Error handling

- Catalog always renders.
- Detail failures show message + retry; do not crash shell.
- For screens needing only one API, do not block on the other.

## Testing

- Formatters unit tests (hours, percent, н/д).
- Catalog: 6 items; selection maps to report id.
- Chart mappers: stable series from fixture KPI JSON.
- Dashboard: seed creates enough WOs that `manager-kpis` returns non-zero samples for core fields (unit/route test with fixed clock).
- Focused Gradle tests only; no local Wasm/full desktop distribution builds.

## Success criteria

1. Manager opens Отчёты → видит список из 6 отчётов, не простыню KPI.
2. Каждый отчёт открывается отдельно; «Назад» возвращает в список.
3. Числа без «хвоста» doubles; графики видны на 5 KPI-отчётах.
4. После seed на Smoke KPI/Gantt заполнены; smoke checklist обновлён.
5. CI green on `client-app` + `dashboard-service`; deploy on default branch.

## Out of scope follow-ups

- Split pane catalog on wide desktop
- Deep links / share report URL
- Historical trend series beyond current period aggregates
