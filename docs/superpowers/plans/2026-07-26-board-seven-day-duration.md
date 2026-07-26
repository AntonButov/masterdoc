# 7-day board + durationHours Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Work orders store `durationHours`; the client board shows one Mon–Sun week with compact bars spanning `ceil(hours/8)` working days from `dueAt` (start); full fields stay on detail.

**Architecture:** Persist `durationHours` in `dashboard-service` (default 8). Keep board JSON shape (`weeks[]` + items), but include a WO in every week column whose Mon–Fri intersects its workday span (so cross-week clips work). Pure layout helpers live in `client-app` `auth` (no new date library — small ISO helpers). `BoardScreen` becomes a 7-column week grid; `WorkOrderBoardCard` is title + type accent only.

**Tech Stack:** Kotlin/Ktor (`dashboard-service`), KMP Compose (`client-app`), kotlinx.serialization, existing client design-system / `fixaverse-design`.

## Global Constraints

- `durationHours >= 1`; omit on create/scheduler → **8**
- `dueAt` = **start** date `YYYY-MM-DD`
- `spanDays = max(1, ceil(durationHours / 8.0))`; only **Mon–Fri** count
- Board card: **title + type accent** only (no status/assignee/hours/equipment)
- No drag-resize; no new board payload shape
- Builds/deploy via GitHub Actions after push; local: targeted unit tests only
- Commit + push each repo when its task is done; watch Actions

---

## File map

| File | Role |
|------|------|
| `dashboard-service/.../WorkOrderStore.kt` | `durationHours` on models; validate; create/update |
| `dashboard-service/.../WeekDates.kt` | workday span + week intersection helpers (JVM) |
| `dashboard-service/.../Application.kt` | PATCH `durationHours` |
| `dashboard-service/.../PprScheduler.kt` | create with default duration (via store default) |
| `dashboard-service/.../WorkOrderRoutesTest.kt` | API tests |
| `client-app/auth/.../WorkOrderModels.kt` | DTO + patch wire |
| `client-app/auth/.../WorkOrderDuration.kt` | `spanDays`, occupied dates, week clip |
| `client-app/auth/.../IsoDates.kt` | minimal YYYY-MM-DD + weekday math |
| `client-app/auth/.../WorkOrderDurationTest.kt` | unit tests |
| `client-app/auth/.../WorkOrdersRepositoryTest.kt` | decode/patch duration |
| `client-app/.../BoardScreen.kt` | 7-day week UI + nav |
| `client-app/.../WorkOrderBoardCard.kt` | compact bar |
| `client-app/.../WorkOrderDetailScreen.kt` | duration row + optional PATCH |
| `masterdoc/docs/.../2026-07-25-work-orders-board-design.md` | note `durationHours` + dueAt=start |

---

### Task 1: `durationHours` in dashboard-service

**Files:**
- Modify: `dashboard-service/src/main/kotlin/pro/masterdoc/dashboard/WorkOrderStore.kt`
- Modify: `dashboard-service/src/main/kotlin/pro/masterdoc/dashboard/Application.kt`
- Modify: `dashboard-service/src/test/kotlin/pro/masterdoc/dashboard/WorkOrderRoutesTest.kt`

**Interfaces:**
- Consumes: existing `CreateWorkOrderRequest`, `WorkOrder`, `update(...)`
- Produces: `WorkOrder.durationHours: Int`; create accepts optional `durationHours`; update accepts `durationHours: Int?`

- [ ] **Step 1: Write failing tests**

Add to `WorkOrderRoutesTest.kt`:

```kotlin
@Test
fun createDefaultsDurationHoursTo8() = testApplication {
    // same module setup as createEmergency...
    val res = client.post("/work-orders") {
        header("X-Org-Id", "o1")
        contentType(ContentType.Application.Json)
        setBody("""{"type":"emergency","title":"Утечка","assetId":"a1","siteId":"s1","dueAt":"2026-07-22"}""")
    }
    assertEquals(HttpStatusCode.Created, res.status)
    assertEquals(8, Json.parseToJsonElement(res.bodyAsText()).jsonObject["durationHours"]!!.jsonPrimitive.int)
}

@Test
fun createRejectsDurationHoursBelow1() = testApplication {
    // ...
    setBody("""{"type":"emergency","title":"X","assetId":"a1","siteId":"s1","dueAt":"2026-07-22","durationHours":0}""")
    assertEquals(HttpStatusCode.BadRequest, res.status) // or whatever existing error mapping uses
}

@Test
fun patchDurationHours() = testApplication {
    // create WO, then:
    val patched = client.patch("/work-orders/$id") {
        header("X-Org-Id", "o1")
        contentType(ContentType.Application.Json)
        setBody("""{"durationHours":16}""")
    }
    assertEquals(16, Json.parseToJsonElement(patched.bodyAsText()).jsonObject["durationHours"]!!.jsonPrimitive.int)
}
```

Mirror the existing testApplication / AllowAllAssetLookup setup from neighboring tests in the same file.

- [ ] **Step 2: Run tests — expect FAIL**

```bash
cd dashboard-service && ./gradlew test --tests 'pro.masterdoc.dashboard.WorkOrderRoutesTest.createDefaultsDurationHoursTo8' --tests 'pro.masterdoc.dashboard.WorkOrderRoutesTest.createRejectsDurationHoursBelow1' --tests 'pro.masterdoc.dashboard.WorkOrderRoutesTest.patchDurationHours'
```

Expected: FAIL (missing field / compile error).

- [ ] **Step 3: Implement models + store + PATCH**

In `WorkOrder` and `CreateWorkOrderRequest` add:

```kotlin
val durationHours: Int = 8, // on Create: use nullable Int? = null and resolve in create()
```

Prefer on create request:

```kotlin
val durationHours: Int? = null,
```

In `WorkOrderStore.create`:

```kotlin
val hours = req.durationHours ?: 8
require(hours >= 1) { "durationHours must be >= 1" }
// ...
dueAt = req.dueAt,
durationHours = hours,
```

In `update` add `durationHours: Int? = null` and:

```kotlin
if (durationHours != null) {
    require(durationHours >= 1) { "durationHours must be >= 1" }
    next = next.copy(durationHours = durationHours)
}
```

In `Application.kt` PATCH handler:

```kotlin
val durationHours = obj["durationHours"]?.jsonPrimitive?.intOrNull
// pass to workOrderStore.update(..., durationHours = durationHours)
```

Scheduler unchanged — omit field → default 8.

- [ ] **Step 4: Run tests — expect PASS**

Same gradle command as Step 2. Expected: PASS.

- [ ] **Step 5: Commit and push**

```bash
cd dashboard-service
git add src/main/kotlin/pro/masterdoc/dashboard/WorkOrderStore.kt \
        src/main/kotlin/pro/masterdoc/dashboard/Application.kt \
        src/test/kotlin/pro/masterdoc/dashboard/WorkOrderRoutesTest.kt
git commit -m "$(cat <<'EOF'
feat(work-orders): add durationHours (default 8)

EOF
)"
git push origin HEAD
gh run watch $(gh run list --limit 1 --json databaseId -q '.[0].databaseId') --exit-status
```

---

### Task 2: Board week intersection by workday span (server)

**Files:**
- Modify: `dashboard-service/src/main/kotlin/pro/masterdoc/dashboard/WeekDates.kt`
- Modify: `dashboard-service/src/main/kotlin/pro/masterdoc/dashboard/WorkOrderStore.kt` (`board`)
- Modify: `dashboard-service/src/test/kotlin/pro/masterdoc/dashboard/WorkOrderRoutesTest.kt`

**Interfaces:**
- Consumes: `durationHours`, `dueAt`
- Produces:
  - `WeekDates.spanWorkingDays(start: LocalDate, durationHours: Int): List<LocalDate>`
  - `WeekDates.intersectsWeek(occupied: List<LocalDate>, weekMonday: LocalDate): Boolean`
  - `board(...)` includes WO in a week column if span intersects that Mon–Sun week’s calendar days (occupied are weekdays only)

- [ ] **Step 1: Write failing tests**

```kotlin
@Test
fun weekDatesSpanFridayThreeWorkdays() {
    val start = LocalDate.parse("2026-07-24") // Friday
    val occupied = WeekDates.spanWorkingDays(start, durationHours = 24) // ceil(24/8)=3
    assertEquals(
        listOf(
            LocalDate.parse("2026-07-24"),
            LocalDate.parse("2026-07-27"),
            LocalDate.parse("2026-07-28"),
        ),
        occupied,
    )
}

@Test
fun boardIncludesWoInNextWeekWhenSpanCrossesWeekend() = testApplication {
    // create emergency dueAt=2026-07-24 (Fri), durationHours=24
    // GET /work-orders/board?weekStart=2026-07-27&weeks=1
    // expect items contain that WO (Mon/Tue of next week)
}
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
cd dashboard-service && ./gradlew test --tests 'pro.masterdoc.dashboard.WorkOrderRoutesTest.weekDatesSpanFridayThreeWorkdays' --tests 'pro.masterdoc.dashboard.WorkOrderRoutesTest.boardIncludesWoInNextWeekWhenSpanCrossesWeekend'
```

- [ ] **Step 3: Implement helpers + board filter**

```kotlin
// WeekDates.kt
fun spanWorkingDays(start: LocalDate, durationHours: Int): List<LocalDate> {
    val spanDays = maxOf(1, (durationHours + 7) / 8) // ceil for positive ints
    val out = ArrayList<LocalDate>(spanDays)
    var d = start
    while (out.size < spanDays) {
        val dow = d.dayOfWeek
        if (dow != DayOfWeek.SATURDAY && dow != DayOfWeek.SUNDAY) {
            out.add(d)
        }
        d = d.plusDays(1)
    }
    return out
}

fun intersectsWeek(occupied: List<LocalDate>, weekMonday: LocalDate): Boolean {
    val end = weekMonday.plusDays(7) // exclusive
    return occupied.any { !it.isBefore(weekMonday) && it.isBefore(end) }
}
```

In `board`, replace dueAt-in-column filter:

```kotlin
.filter { wo ->
    val start = WeekDates.parseDate(wo.dueAt) ?: return@filter false
    val occupied = WeekDates.spanWorkingDays(start, wo.durationHours)
    WeekDates.intersectsWeek(occupied, ws)
}
```

Horizon `inRange` for the multi-week query: include WO if span intersects `[start, endExclusive)`.

- [ ] **Step 4: Run tests — expect PASS**

```bash
cd dashboard-service && ./gradlew test --tests 'pro.masterdoc.dashboard.WorkOrderRoutesTest'
```

Expected: all PASS.

- [ ] **Step 5: Commit and push**

```bash
cd dashboard-service
git add src/main/kotlin/pro/masterdoc/dashboard/WeekDates.kt \
        src/main/kotlin/pro/masterdoc/dashboard/WorkOrderStore.kt \
        src/test/kotlin/pro/masterdoc/dashboard/WorkOrderRoutesTest.kt
git commit -m "$(cat <<'EOF'
fix(board): include work orders by workday span intersection

EOF
)"
git push origin HEAD
gh run watch $(gh run list --limit 1 --json databaseId -q '.[0].databaseId') --exit-status
```

---

### Task 3: Client DTOs + repository `durationHours`

**Files:**
- Modify: `client-app/auth/src/commonMain/kotlin/pro/masterdoc/client/auth/WorkOrderModels.kt`
- Modify: `client-app/auth/src/jvmTest/kotlin/pro/masterdoc/client/auth/WorkOrdersRepositoryTest.kt`

**Interfaces:**
- Consumes: gateway JSON with `durationHours`
- Produces: `WorkOrderDto.durationHours: Int = 8`; `patch(..., durationHours: Int? = null)`

- [ ] **Step 1: Write failing test**

```kotlin
@Test
fun getBoardDecodesDurationHours() = runBlocking {
    // response item includes "durationHours":16
    val board = repo.getBoard(weeks = 1)
    assertEquals(16, board.weeks[0].items[0].durationHours)
}

@Test
fun patchSendsDurationHours() = runBlocking {
    repo.patch("wo-1", durationHours = 24)
    // assert body JSON contains "durationHours":24
}
```

Update existing fixture JSON strings in the file to include `"durationHours":8` where a full WO is returned (or rely on default after DTO change).

- [ ] **Step 2: Run test — expect FAIL**

```bash
cd client-app && ./gradlew :auth:jvmTest --tests 'pro.masterdoc.client.auth.WorkOrdersRepositoryTest.getBoardDecodesDurationHours' --tests 'pro.masterdoc.client.auth.WorkOrdersRepositoryTest.patchSendsDurationHours'
```

- [ ] **Step 3: Implement DTO + patch**

```kotlin
// WorkOrderDto
val durationHours: Int = 8,

// CreateWorkOrderRequest (optional for later UI create)
val durationHours: Int? = null,

// patch(...)
durationHours: Int? = null,
// in buildJsonObject:
if (durationHours != null) put("durationHours", JsonPrimitive(durationHours))
```

- [ ] **Step 4: Run tests — expect PASS**

Same command. Expected: PASS.

- [ ] **Step 5: Commit and push**

```bash
cd client-app
git add auth/src/commonMain/kotlin/pro/masterdoc/client/auth/WorkOrderModels.kt \
        auth/src/jvmTest/kotlin/pro/masterdoc/client/auth/WorkOrdersRepositoryTest.kt
git commit -m "$(cat <<'EOF'
feat(auth): wire work order durationHours

EOF
)"
git push origin HEAD
gh run watch $(gh run list --limit 1 --json databaseId -q '.[0].databaseId') --exit-status
```

---

### Task 4: Client layout helpers (`WorkOrderDuration` + `IsoDates`)

**Files:**
- Create: `client-app/auth/src/commonMain/kotlin/pro/masterdoc/client/auth/IsoDates.kt`
- Create: `client-app/auth/src/commonMain/kotlin/pro/masterdoc/client/auth/WorkOrderDuration.kt`
- Create: `client-app/auth/src/commonTest/kotlin/pro/masterdoc/client/auth/WorkOrderDurationTest.kt`

**Interfaces:**
- Produces:

```kotlin
object IsoDates {
    /** @return epochDay (days since 1970-01-01) or null if invalid YYYY-MM-DD */
    fun parseToEpochDay(iso: String): Long?
    fun formatEpochDay(epochDay: Long): String
    /** ISO-8601: 1=Monday … 7=Sunday */
    fun dayOfWeekIso(epochDay: Long): Int
}

object WorkOrderDuration {
    fun spanDays(durationHours: Int): Int
    /** Occupied calendar dates (YYYY-MM-DD), working days only */
    fun occupiedDates(dueAt: String, durationHours: Int): List<String>
    /**
     * Clip to week Monday..Sunday.
     * @return startColumn 0..6, spanColumns >=1, or null if no intersection
     */
    fun clipToWeek(
        occupiedIsoDates: List<String>,
        weekMondayIso: String,
    ): WeekClip?
}

data class WeekClip(val startColumn: Int, val spanColumns: Int)
```

- [ ] **Step 1: Write failing tests**

```kotlin
class WorkOrderDurationTest {
    @Test
    fun spanDaysCeilEight() {
        assertEquals(1, WorkOrderDuration.spanDays(1))
        assertEquals(1, WorkOrderDuration.spanDays(8))
        assertEquals(2, WorkOrderDuration.spanDays(9))
        assertEquals(2, WorkOrderDuration.spanDays(16))
        assertEquals(3, WorkOrderDuration.spanDays(17))
    }

    @Test
    fun fridayThreeWorkdaysSkipsWeekend() {
        assertEquals(
            listOf("2026-07-24", "2026-07-27", "2026-07-28"),
            WorkOrderDuration.occupiedDates("2026-07-24", 24),
        )
    }

    @Test
    fun clipCrossWeekShowsMonTueOnNextWeek() {
        val occupied = WorkOrderDuration.occupiedDates("2026-07-24", 24)
        val clip = WorkOrderDuration.clipToWeek(occupied, "2026-07-27")!!
        assertEquals(0, clip.startColumn) // Monday
        assertEquals(2, clip.spanColumns) // Mon–Tue
    }
}
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
cd client-app && ./gradlew :auth:jvmTest --tests 'pro.masterdoc.client.auth.WorkOrderDurationTest'
```

(If commonTest does not run on jvmTest, use `:auth:allTests` or ensure commonTest is wired — follow existing `PkceTest` / `IdTokenProfileTest` pattern under `auth/src/commonTest`.)

- [ ] **Step 3: Implement IsoDates + WorkOrderDuration**

Implement civil-from-epoch / epoch-from-civil (Howard Hinnant algorithm is fine) so Wasm/Android/JVM share one code path. No new Gradle dependency.

```kotlin
fun spanDays(durationHours: Int): Int {
    val h = maxOf(1, durationHours)
    return (h + 7) / 8
}

fun occupiedDates(dueAt: String, durationHours: Int): List<String> {
    val start = IsoDates.parseToEpochDay(dueAt) ?: return emptyList()
    val need = spanDays(durationHours)
    val out = ArrayList<String>(need)
    var day = start
    while (out.size < need) {
        val dow = IsoDates.dayOfWeekIso(day) // 1..7
        if (dow in 1..5) out.add(IsoDates.formatEpochDay(day))
        day++
    }
    return out
}

fun clipToWeek(occupiedIsoDates: List<String>, weekMondayIso: String): WeekClip? {
    val monday = IsoDates.parseToEpochDay(weekMondayIso) ?: return null
    val weekDays = (0L..6L).map { IsoDates.formatEpochDay(monday + it) }
    val hitIndexes = weekDays.mapIndexedNotNull { i, d -> if (d in occupiedIsoDates.toSet()) i else null }
    if (hitIndexes.isEmpty()) return null
    val start = hitIndexes.first()
    val end = hitIndexes.last()
    return WeekClip(startColumn = start, spanColumns = end - start + 1)
}
```

Note: if occupied skips Sat/Sun, indexes are contiguous for a single bar; `end-start+1` is correct for a continuous workday block clipped to the week.

- [ ] **Step 4: Run tests — expect PASS**

Same gradle command. Expected: PASS.

- [ ] **Step 5: Commit and push**

```bash
cd client-app
git add auth/src/commonMain/kotlin/pro/masterdoc/client/auth/IsoDates.kt \
        auth/src/commonMain/kotlin/pro/masterdoc/client/auth/WorkOrderDuration.kt \
        auth/src/commonTest/kotlin/pro/masterdoc/client/auth/WorkOrderDurationTest.kt
git commit -m "$(cat <<'EOF'
feat(auth): workday span helpers for board layout

EOF
)"
git push origin HEAD
gh run watch $(gh run list --limit 1 --json databaseId -q '.[0].databaseId') --exit-status
```

---

### Task 5: Compact `WorkOrderBoardCard`

**Files:**
- Modify: `client-app/composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/WorkOrderBoardCard.kt`

**Interfaces:**
- Consumes: `WorkOrderDto` (title, type)
- Produces: composable showing title + type accent only

- [ ] **Step 1: Rewrite card**

Replace body so it no longer shows status chips, dueAt, assignee. Keep Surface + 4.dp accent strip by type (`emergency` → error, else primary). Single `AppText` title (`AppTextStyle.Body` or `Title`), `maxLines = 2`.

```kotlin
@Composable
fun WorkOrderBoardCard(
    order: WorkOrderDto,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
) {
    val accent =
        when (order.type) {
            "emergency" -> MaterialTheme.colorScheme.error
            else -> MaterialTheme.colorScheme.primary
        }
    Surface(
        modifier = modifier
            .fillMaxWidth()
            .border(1.dp, MaterialTheme.colorScheme.outlineVariant, MaterialTheme.shapes.medium)
            .clickable(onClick = onClick),
        shape = MaterialTheme.shapes.medium,
        color = MaterialTheme.colorScheme.surface,
    ) {
        Row(Modifier = Modifier.height(IntrinsicSize.Min)) {
            Box(Modifier = Modifier.width(4.dp).fillMaxHeight().background(accent))
            AppText(
                text = order.title,
                style = AppTextStyle.Body,
                modifier = Modifier.padding(ClientSpacing.sm).weight(1f),
            )
        }
    }
}
```

(Adjust to match existing `AppText` signature — if no `maxLines`, truncate in UI with shorter height.)

- [ ] **Step 2: Commit and push**

```bash
cd client-app
git add composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/WorkOrderBoardCard.kt
git commit -m "$(cat <<'EOF'
refactor(ui): compact board card to title and type accent

EOF
)"
git push origin HEAD
gh run watch $(gh run list --limit 1 --json databaseId -q '.[0].databaseId') --exit-status
```

---

### Task 6: Redesign `BoardScreen` (7-day week)

**Files:**
- Modify: `client-app/composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/BoardScreen.kt`

**Interfaces:**
- Consumes: `WorkOrdersRepository.getBoard`, `WorkOrderDuration`, `WorkOrderBoardCard`, `WorkOrderDetailScreen`
- Produces: week nav + 7 columns + clipped bars

- [ ] **Step 1: Replace week-columns UI with day grid**

State:

```kotlin
var weekMonday by remember { mutableStateOf(currentMondayIso()) } // helper using IsoDates
```

Load:

```kotlin
// Fetch 1 week from API for that Monday (server now returns span-intersecting items)
weeks = repository.getBoard(weekStart = weekMonday, weeks = 1).weeks
val items = weeks.firstOrNull()?.items.orEmpty()
```

Header row: «Доска», AppButton/Icon ← → changing `weekMonday` by ±7 epoch days, label like `20–26 июл` or raw `2026-07-20 — 2026-07-26`.

Day headers: for `col in 0..6` show short weekday RU (`Пн`…`Вс`) + `dd`.

Layout approach (keep simple, no absolute canvas):

1. Compute lanes: list of `BoardLaneItem(order, clip: WeekClip)` for items where `clipToWeek(occupiedDates(...), weekMonday) != null`.
2. For overlapping, assign lane index greedily (sort by startColumn, then span).
3. Render:
   - Background `Row` of 7 equal `weight(1f)` day columns (hairline dividers).
   - For each lane, a `Row` with `Spacer(weight = startColumn.toFloat())`, card `weight = spanColumns.toFloat()`, trailing spacer if needed.

```kotlin
data class BoardLaneItem(
    val order: WorkOrderDto,
    val clip: WeekClip,
    val lane: Int,
)

fun assignLanes(items: List<Pair<WorkOrderDto, WeekClip>>): List<BoardLaneItem> {
    val sorted = items.sortedWith(compareBy({ it.second.startColumn }, { -it.second.spanColumns }, { it.first.id }))
    val laneEnds = mutableListOf<Int>() // end column exclusive per lane
    return sorted.map { (order, clip) ->
        val end = clip.startColumn + clip.spanColumns
        val lane = laneEnds.indexOfFirst { it <= clip.startColumn }.let { idx ->
            if (idx < 0) {
                laneEnds.add(end)
                laneEnds.lastIndex
            } else {
                laneEnds[idx] = end
                idx
            }
        }
        BoardLaneItem(order, clip, lane)
    }
}
```

Empty: if no lane items → `AppText("Нет заявок")`.

Keep detail navigation via `selectedId` → `WorkOrderDetailScreen` as today.

Use `ClientSpacing`, `AppText`, `AppButton` / icon buttons from design-system only.

- [ ] **Step 2: Commit and push**

```bash
cd client-app
git add composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/BoardScreen.kt
git commit -m "$(cat <<'EOF'
feat(ui): 7-day work-order board with workday spans

EOF
)"
git push origin HEAD
gh run watch $(gh run list --limit 1 --json databaseId -q '.[0].databaseId') --exit-status
```

- [ ] **Step 3: Smoke (after deploy success)**

On `https://app.fixaverse.ru/` in org **Fixaverse Smoke**: open Доска → 7 day headers → ←/→ changes week → if any WO exists, card is title-only → open detail.

---

### Task 7: Detail shows / patches duration

**Files:**
- Modify: `client-app/composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/WorkOrderDetailScreen.kt`

**Interfaces:**
- Consumes: `wo.durationHours`, `repository.patch(..., durationHours = …)`
- Produces: «Длительность, ч» on detail

- [ ] **Step 1: Add duration UI**

After title / before «Срок»:

```kotlin
DetailRow("Длительность, ч", wo.durationHours.toString())
DetailRow("Начало", wo.dueAt) // rename label from «Срок» to «Начало» to match dueAt=start
```

MVP: display only is enough if editing is awkward without text field component; if `AppTextField` (or existing input) exists in design-system, add a small edit + save via `patch(durationHours = n)` for `status != closed`. Otherwise display-only and document in commit message.

Search design-system for text field before inventing one — do not add a new DS component in this task.

- [ ] **Step 2: Commit and push**

```bash
cd client-app
git add composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/WorkOrderDetailScreen.kt
git commit -m "$(cat <<'EOF'
feat(ui): show work order durationHours on detail

EOF
)"
git push origin HEAD
gh run watch $(gh run list --limit 1 --json databaseId -q '.[0].databaseId') --exit-status
```

---

### Task 8: Sync contract doc

**Files:**
- Modify: `masterdoc/docs/superpowers/specs/2026-07-25-work-orders-board-design.md`

- [ ] **Step 1: Patch contract**

Add under types/API:

- Field `durationHours` (int, >=1, default 8); `dueAt` is start date.
- Board columns still by ISO week, but items include WOs whose workday span intersects the week.

Link already points to `2026-07-26-board-seven-day-duration-design.md`.

- [ ] **Step 2: Commit and push**

```bash
cd masterdoc
git add docs/superpowers/specs/2026-07-25-work-orders-board-design.md
git commit -m "$(cat <<'EOF'
docs: note durationHours and workday board intersection

EOF
)"
git push origin HEAD
```

---

## Spec coverage (self-review)

| Spec requirement | Task |
|------------------|------|
| `durationHours` persist/validate/default 8 | 1 |
| Board JSON shape unchanged + span intersection | 2 |
| Client DTO/PATCH | 3 |
| `spanDays` / workdays / week clip unit tests | 4 |
| Compact card | 5 |
| 7-day UI + week nav + clip bars | 6 |
| Detail duration | 7 |
| Contract doc sync | 8 |
| No drag-resize / no new layout payload | honored (out of scope) |

## Placeholder / consistency check

- `spanDays` formula `(h + 7) / 8` matches `ceil(h/8)` for `h >= 1`.
- Server and client use the same workday rule (Mon–Fri).
- `WeekClip.startColumn` is 0=Monday … 6=Sunday.
- Default `durationHours = 8` on DTO matches server default for legacy JSON without the field (`ignoreUnknownKeys` + default).
