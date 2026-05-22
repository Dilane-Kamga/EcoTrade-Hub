# Instructions for NEXCharge's Claude Code

> **How to use this file**: drop into the NEXCharge repo as `CLAUDE.md` (or append to existing `CLAUDE.md`). These are hard constraints from the parallel EcoTrade Hub layer that will be merged into the same Spring Boot app.

---

## Context

You are implementing **NEXCharge** (EV booking + OCPP + ESG) in Java 21 / Spring Boot 3 / Postgres 16 / Flyway. A second module — **EcoTrade Hub** — is being built in parallel against the same codebase by another developer. EcoTrade adds:

- **Module 1 — Green Score**: variable hourly grid carbon intensity (Mauritius), replacing the constant 0.4 kg/kWh CO2 factor
- **Module 2 — Audit Hash Chain**: SHA-256 chain over completed sessions, complementing the existing `audit_log`

EcoTrade lives in package `com.accenture.nexcharge.ecotrade.*` inside the same Spring Boot application (not a separate service).

**Source of truth for EcoTrade**: `EcoTradeHub_Layer_Spec.md` and `docs/superpowers/plans/2026-05-22-module-{1,2}-*.md` in the EcoTrade-Hub repo (https://github.com/Dilane-Kamga/EcoTrade-Hub).

---

## Hard Constraints

### C1. Flyway version reservations

**DO NOT use these migration versions** — they belong to EcoTrade Hub:

| Reserved | Owner | File |
|----------|-------|------|
| `V3` | EcoTrade Module 1 | `V3__green_score.sql` |
| `V4` | EcoTrade Module 2 | `V4__audit_chain.sql` |

NEXCharge migrations: **V1, V2, then V5+**.

If you genuinely need a 3rd migration in Sprint 1, use `V5` and flag it.

### C2. `charging_sessions` schema

The table MUST contain at least these columns after V1/V2:

```sql
id              UUID PRIMARY KEY
status          VARCHAR NOT NULL    -- must support value 'COMPLETED'
started_at      TIMESTAMPTZ NOT NULL
ended_at        TIMESTAMPTZ         -- NULL until completion
kwh_delivered   NUMERIC(10,3) NOT NULL
```

- `started_at` and `ended_at` MUST be `TIMESTAMPTZ` (not naive `TIMESTAMP`). EcoTrade slices durations across hour boundaries in `Indian/Mauritius` timezone — naive timestamps will produce wrong CO2 values.
- The `status` column must accept the literal string `'COMPLETED'` (or whatever enum value you choose for terminal state — tell EcoTrade dev which one).

### C3. CO2 column policy

The original NEXCharge spec mentions `co2_kg_avoided` populated as `kwh * 0.4` constant. **EcoTrade replaces this calculation.** Pick one:

**Option A (recommended)**: don't add the column to `charging_sessions`.
**Option B**: add it as `NULL` and never populate it — EcoTrade ignores it.

EcoTrade writes the real value to `green_score.co2_avoided_g` (BIGINT, in grams), with FK to `charging_sessions(id)`. Dashboards should join on `green_score`, not on a column of `charging_sessions`.

### C4. SessionCompletedEvent contract

EcoTrade needs a Spring `ApplicationEvent` published from your `StopTransactionHandler` to trigger Green Score computation and audit chain append in the same transaction.

**Required event class** — create it exactly like this:

```java
// File: src/main/java/com/accenture/nexcharge/events/SessionCompletedEvent.java
package com.accenture.nexcharge.events;

import java.util.UUID;

public record SessionCompletedEvent(UUID sessionId) {}
```

**Required publication point** — in your `StopTransactionHandler` (or whichever service finalizes the session):

```java
// After session.setStatus(COMPLETED) and BEFORE the @Transactional method returns:
applicationEventPublisher.publishEvent(new SessionCompletedEvent(session.getId()));
```

Constraints:
- The event MUST be published **after** `status` becomes `COMPLETED` and `kwh_delivered` is final
- The event MUST be published **inside** the same `@Transactional` boundary that persists the session — EcoTrade listeners join the transaction via `Propagation.REQUIRED`
- DO NOT use `@TransactionalEventListener(phase = AFTER_COMMIT)` semantics — EcoTrade's writes must be in the same TX so a rollback rolls back everything

`ApplicationEventPublisher` is auto-injected by Spring — just declare it as a constructor argument.

### C5. Package layout

NEXCharge code stays under `com.accenture.nexcharge.*`. EcoTrade adds:

```
com.accenture.nexcharge
├── events/                  ← you create SessionCompletedEvent here
└── ecotrade/                ← EcoTrade owns this entire subtree
    ├── greenscore/
    ├── auditchain/
    └── config/
```

Your `@SpringBootApplication` annotation at the root package will scan EcoTrade's beans automatically. **Do not add explicit `@ComponentScan` excludes** for the `ecotrade` subpackage.

### C6. Spring Security roles

EcoTrade Module 2 exposes REST endpoints protected by:

```java
@PreAuthorize("hasAnyRole('ADMIN', 'SUSTAINABILITY_OFFICER')")
```

Required in your security config:

- [ ] `@EnableMethodSecurity` annotation present on a `@Configuration` class
- [ ] Authority `ROLE_ADMIN` is granted to admin users
- [ ] Authority `ROLE_SUSTAINABILITY_OFFICER` is granted to SO users (your spec already mentions this persona — wire it in `GrantedAuthority` mapping)

If method security or roles are missing, add them in Sprint 1.

### C7. Tech stack lock

EcoTrade depends on these versions — do not downgrade:

- Java **21** (records, pattern matching, virtual threads OK)
- Spring Boot **3.x** (Jakarta EE, not javax)
- Postgres **16**
- Flyway **9+** or **10+**
- Testcontainers **1.19+** for ITs

---

## Validation commands

Before pushing Sprint 1, run:

```bash
# 1. Confirm no V3/V4 migrations exist
find . -name "V3__*.sql" -o -name "V4__*.sql"
# Expected: empty output

# 2. Confirm SessionCompletedEvent exists
find . -path "*/events/SessionCompletedEvent.java"
# Expected: exactly one match in com/accenture/nexcharge/events/

# 3. Confirm event is published in StopTransaction flow
grep -r "publishEvent(new SessionCompletedEvent" src/main/java
# Expected: at least one match in your StopTransactionHandler / equivalent

# 4. Confirm method security is enabled
grep -r "@EnableMethodSecurity" src/main/java
# Expected: at least one match in a @Configuration class

# 5. Build + tests
./mvnw clean verify   # or ./gradlew check
# Expected: green
```

If any of these checks fails, fix before pushing — EcoTrade's plans assume all 5 pass.

---

## Workflow

```
1. You implement NEXCharge Sprint 1 on branch  nexcharge/sprint-1
2. You open a PR to main with the 7 constraints above satisfied
3. EcoTrade dev reviews the PR for compliance with C1–C7
4. PR merges to main
5. EcoTrade dev starts Module 1 implementation on  ecotrade/module-1-green-score
6. EcoTrade dev opens PR (will not touch your files — only adds new ones + listens to your event)
7. Repeat for Module 2 on  ecotrade/module-2-audit-chain
```

You will NOT need to:
- Modify EcoTrade code
- Run EcoTrade migrations manually (Flyway picks them up from the merged PR)
- Configure anything for EcoTrade beans (Spring auto-scans them)

You only own constraints C1–C7. Everything else is EcoTrade's problem.

---

## If you disagree with a constraint

Open a GitHub issue on https://github.com/Dilane-Kamga/EcoTrade-Hub **before** merging Sprint 1. Reverting after merge is more painful than discussing first.

Common likely friction points:
- **C3**: you might prefer keeping `co2_kg_avoided` populated for backward compatibility with your existing dashboards. → Tell EcoTrade dev — they'll adapt their query layer instead of forcing schema change.
- **C4**: you might prefer direct service injection over Spring events. → That's also OK. Tell EcoTrade dev — they'll inject `GreenScoreService` and `AuditChainService` into your handler instead. Pick one approach, not both.

---

## Reading order if you want full context

1. This file (you are here) — hard constraints, ~5 min
2. `docs/coordination-with-nexcharge.md` (same repo) — human-readable version with rationale
3. `EcoTradeHub_Layer_Spec.md` (root) — full spec, §2 for reuse/extend/replace and §9 for coordination
4. `docs/superpowers/plans/2026-05-22-module-1-green-score.md` — exactly what EcoTrade adds for Module 1
5. `docs/superpowers/plans/2026-05-22-module-2-audit-hash-chain.md` — exactly what EcoTrade adds for Module 2

You can stop after this file unless you hit a friction point.
