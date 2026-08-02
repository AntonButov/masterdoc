# Tasks 5–7 Report: black-box-service persistence ship

**Status:** Implemented; local JVM suite completed, Testcontainers cases skipped because Docker is unavailable on this workstation.  
**Repo:** `black-box-service`  
**Branch:** `main`
**Commit:** `9a246e2` (`feat(black-box): persist audit events in postgres`)

## Implementation

- Added PostgreSQL, HikariCP, Flyway, and Testcontainers dependencies.
- Added `Db.connect()` with Flyway startup migration and `V1__audit_events.sql`.
- Replaced `ConcurrentLinkedDeque` with JDBC `AuditStore`; append retains redaction/truncation, and list filters are SQL `WHERE` predicates.
- Converted migration, persistence, and route tests to PostgreSQL Testcontainers.
- Added `black-box-postgres`, `black_box_pg`, runtime database environment, and README persistence notes.

## TDD evidence

- RED: `DbMigrateTest` initially failed at test compilation because Testcontainers/`Db`/JDBC store were absent.
- GREEN: targeted migration test completed with `BUILD SUCCESSFUL`.
- Full suite: `./gradlew test --no-daemon` completed with exit 0; 7 Testcontainers tests were skipped locally because Docker is unavailable.
- `git diff --check` passed.
- Compose validation could not run: Docker CLI is unavailable locally; CI is expected to execute the Testcontainers and deploy gates.

## CI / deploy

- GitHub Actions:
  [30763697804](https://github.com/masterdoc-app/black-box-service/actions/runs/30763697804)
  — test PASS, Compose deploy PASS.

## Concerns

- Local Docker absence prevented executing the Testcontainers tests and Compose validation; GitHub test and deploy gates passed on the runner.
