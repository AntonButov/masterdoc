# Services Critical Logging Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** Ensure every Fixaverse runtime service logs critical ops events to stdout (Docker/journald) with structured-ish `event=` messages, without secrets or PHI.

**Architecture:** Copy api-gateway pattern where missing: SLF4J + console `logback.xml` + CallLogging (Ktor) or Spring Boot defaults. Domain WARN/ERROR on auth deny, upstream failures, swallowed catches, StatusPages, startup. Never log tokens, Authorization, request/response bodies, clinical text, PDF bytes.

**Tech Stack:** Kotlin, Ktor 3 / Spring Boot 3, Logback, SLF4J

## Global Constraints

- Message shape: `event=<name> key=value …` (space-separated)
- Levels: INFO startup/lifecycle, WARN auth deny / bad request / upstream non-2xx, ERROR unhandled / upstream unavailable / job failure
- Never log: JWT, Bearer, client_secret, refresh_token, BLACK_BOX_INTERNAL_TOKEN, document text, detect answers, case-report result text, Onyx body snippets
- No heavy local Gradle builds — edit only; CI validates after push
- Each service is its own git repo — commit & push that repo only
- Prefer minimal diffs; match existing style; do not refactor unrelated code

---

### Task 1: api-gateway-service critical logs

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/api-gateway-service`

- [x] Log JWT invalid in `JwksTokenValidator` (WARN, no token)
- [x] Log auth denials in `AdminAuth` (WARN + reason + requestId if available)
- [x] Log upstream failures in `Clients.kt`, proxy routes, equipment forward catches (ERROR/WARN)
- [x] INFO startup in `Application` (port, issuer present flags — no secrets)
- [x] Commit & push; do not run local Gradle

### Task 2: backend critical logs + PHI scrub

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/backend`

- [x] Add `logback.xml` + CallLogging (+ optional X-Request-Id)
- [x] Replace `println`/`printStackTrace` with Logger; scrub PHI (case-report result, detect answer, bodySnippet)
- [x] Log all Onyx upstream errors uniformly; INFO startup
- [x] Commit & push

### Task 3: feature-service startup/auth logs

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/feature-service`

- [x] INFO startup JWT config flags in SecurityConfig
- [x] Optional WARN auth failure handler (no token)
- [x] Commit & push

### Task 4: catalog + document + dashboard Ktor baseline

**Repos:** catalog-service, document-service, dashboard-service

- [x] Each: logback.xml + CallLogging + StatusPages WARN + INFO startup
- [x] document: recovery count, save meta (no text); dashboard: catalog lookup WARN, scheduler ERROR/INFO
- [x] Commit & push each repo

### Task 5: black-box + technologist + technologist-mcp

**Repos:** black-box-service, technologist-service, technologist-mcp

- [x] Each: logback + CallLogging + StatusPages logging + INFO startup
- [x] black-box: auth_denied WARN, audit_appended INFO (ids only)
- [x] technologist: job lifecycle + outbound HTTP status (no bodies)
- [x] mcp: tool_call / tool_denied / tool_error / upstream_call
- [x] Commit & push each repo
