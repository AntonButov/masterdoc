# Default site «Цех 1» Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** On empty-org `GET /sites`, catalog-service creates one default site `id=ceh-1`, `name=Цех 1` so Equipment PDF upload works without Admin setup.

**Architecture:** Lazy seed inside `SiteStore.ensureDefaultIfEmpty(orgId)` called from the `GET /sites` route. Seed only when `list(orgId)` is empty. Concurrent creates with fixed id treat «already exists» as success. No client-app changes (Equipment already auto-selects `sites.firstOrNull()`).

**Tech Stack:** Kotlin 2.1, Ktor 3.0.3, kotlin.test, catalog-service in-memory `SiteStore`.

**Spec:** [docs/superpowers/specs/2026-07-28-default-site-ceh-1-design.md](../specs/2026-07-28-default-site-ceh-1-design.md)

## Global Constraints

- Default site id must be exactly `ceh-1` (Latin), name exactly `Цех 1`, address `null`
- Seed only when org has **zero** sites — never overwrite or recreate if any site exists
- Do not add Postgres / Flyway / client UI / MCP tools
- Work in repo `catalog-service/` (git root). Commit and push there; CI runs tests (prefer targeted `./gradlew test --tests …` only if needed locally)
- Existing test that POSTs Cyrillic id `цех-1` must keep passing (different id from `ceh-1`)

## File map

| File | Role |
|------|------|
| `catalog-service/src/main/kotlin/pro/masterdoc/catalog/Application.kt` | `SiteStore.ensureDefaultIfEmpty`; call from `GET /sites` |
| `catalog-service/src/test/kotlin/pro/masterdoc/catalog/SiteRoutesTest.kt` | Route tests for seed / idempotency / skip / re-seed after delete-all |

---

### Task 1: Ensure default site on empty `GET /sites`

**Files:**
- Modify: `catalog-service/src/main/kotlin/pro/masterdoc/catalog/Application.kt` (`SiteStore` ~330–378, `get("/sites")` ~74–77)
- Test: `catalog-service/src/test/kotlin/pro/masterdoc/catalog/SiteRoutesTest.kt`

**Interfaces:**
- Consumes: existing `SiteStore.create`, `SiteStore.list`, `CreateSiteRequest`, `SiteList`
- Produces: `SiteStore.ensureDefaultIfEmpty(orgId: String): List<Site>` — returns `list(orgId)` after optional seed

- [ ] **Step 1: Write the failing tests**

Add to `SiteRoutesTest.kt`:

```kotlin
@Test
fun emptyOrgListSeedsDefaultCeh1() = testApplication {
    application { module() }
    val list = client.get("/sites") { header("X-Org-Id", "org-empty") }
    assertEquals(HttpStatusCode.OK, list.status)
    val items = json.parseToJsonElement(list.bodyAsText()).jsonObject["items"]!!.jsonArray
    assertEquals(1, items.size)
    val site = items[0].jsonObject
    assertEquals("ceh-1", site["id"]!!.jsonPrimitive.content)
    assertEquals("Цех 1", site["name"]!!.jsonPrimitive.content)
    assertEquals("org-empty", site["orgId"]!!.jsonPrimitive.content)
}

@Test
fun secondListDoesNotDuplicateDefault() = testApplication {
    application { module() }
    repeat(2) {
        client.get("/sites") { header("X-Org-Id", "org-once") }
    }
    val list = client.get("/sites") { header("X-Org-Id", "org-once") }
    val items = json.parseToJsonElement(list.bodyAsText()).jsonObject["items"]!!.jsonArray
    assertEquals(1, items.size)
    assertEquals("ceh-1", items[0].jsonObject["id"]!!.jsonPrimitive.content)
}

@Test
fun existingSiteSkipsSeed() = testApplication {
    application { module() }
    client.post("/sites") {
        header("X-Org-Id", "org-has")
        contentType(ContentType.Application.Json)
        setBody("""{"id":"s-custom","name":"Свой цех"}""")
    }
    val list = client.get("/sites") { header("X-Org-Id", "org-has") }
    val items = json.parseToJsonElement(list.bodyAsText()).jsonObject["items"]!!.jsonArray
    assertEquals(1, items.size)
    assertEquals("s-custom", items[0].jsonObject["id"]!!.jsonPrimitive.content)
    assertTrue(!list.bodyAsText().contains("ceh-1"))
}

@Test
fun afterDeleteAllListReseedsDefault() = testApplication {
    application { module() }
    client.get("/sites") { header("X-Org-Id", "org-reseed") }
    client.delete("/sites/ceh-1") { header("X-Org-Id", "org-reseed") }
    val list = client.get("/sites") { header("X-Org-Id", "org-reseed") }
    val items = json.parseToJsonElement(list.bodyAsText()).jsonObject["items"]!!.jsonArray
    assertEquals(1, items.size)
    assertEquals("ceh-1", items[0].jsonObject["id"]!!.jsonPrimitive.content)
}
```

Add import if missing:

```kotlin
import kotlinx.serialization.json.jsonArray
```

- [ ] **Step 2: Run tests to verify they fail**

From `catalog-service/`:

```bash
./gradlew test --tests pro.masterdoc.catalog.SiteRoutesTest.emptyOrgListSeedsDefaultCeh1 --tests pro.masterdoc.catalog.SiteRoutesTest.secondListDoesNotDuplicateDefault --tests pro.masterdoc.catalog.SiteRoutesTest.existingSiteSkipsSeed --tests pro.masterdoc.catalog.SiteRoutesTest.afterDeleteAllListReseedsDefault
```

Expected: FAIL — list empty / size 0 / missing `ceh-1` (seed not implemented).

- [ ] **Step 3: Implement `SiteStore.ensureDefaultIfEmpty`**

In `Application.kt` inside `class SiteStore`, add:

```kotlin
fun ensureDefaultIfEmpty(orgId: String): List<Site> {
    if (list(orgId).isNotEmpty()) return list(orgId)
    try {
        create(orgId, CreateSiteRequest(id = "ceh-1", name = "Цех 1", address = null))
    } catch (_: IllegalArgumentException) {
        // concurrent create with same id — treat as already seeded
    }
    return list(orgId)
}
```

- [ ] **Step 4: Wire `GET /sites`**

Replace the empty-list handler body:

```kotlin
get("/sites") {
    val orgId = call.orgId()
    call.respond(SiteList(items = sites.ensureDefaultIfEmpty(orgId)))
}
```

Do **not** change `POST /sites`, `GET /sites/{id}`, or other routes.

- [ ] **Step 5: Run new + existing site tests**

```bash
./gradlew test --tests pro.masterdoc.catalog.SiteRoutesTest
```

Expected: all PASS (including `createListUpdateDelete`, `deleteBlockedWhenAssetsPresent`, `sitesScopedByOrg`).

- [ ] **Step 6: Commit and push**

```bash
cd catalog-service
git add src/main/kotlin/pro/masterdoc/catalog/Application.kt \
        src/test/kotlin/pro/masterdoc/catalog/SiteRoutesTest.kt
git commit -m "$(cat <<'EOF'
feat: seed default site Цех 1 on empty GET /sites

EOF
)"
git push origin HEAD
```

Then watch GitHub Actions for catalog-service and report result.

---

## Spec coverage (self-review)

| Spec requirement | Task |
|------------------|------|
| Seed on empty `GET /sites` with `ceh-1` / `Цех 1` | Task 1 |
| Idempotent second GET | Task 1 `secondListDoesNotDuplicateDefault` |
| Skip when any site exists | Task 1 `existingSiteSkipsSeed` |
| Concurrent already-exists → list, no 500 | Task 1 `ensureDefaultIfEmpty` catch |
| Re-seed after delete-all | Task 1 `afterDeleteAllListReseedsDefault` |
| No client / Postgres / MCP | Global constraints |

No placeholders remaining.
