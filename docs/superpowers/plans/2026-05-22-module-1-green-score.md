# Module 1 — Green Score variable horaire — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remplacer le facteur CO2 constant `0.4` du `StopTransactionHandler` NEXCharge par un calcul variable selon l'heure du grid Maurice, et exposer un score par user / site / leaderboard.

**Architecture:** Lookup table statique grid Maurice (gCO2/kWh par tranche horaire) → resolver pur Java → service qui persiste un `GreenScore` par session terminée → controller REST `/api/ecotrade/green-score/*`. Hot path : la modif du `StopTransactionHandler` appelle le resolver + le service de façon synchrone dans la même transaction OCPP. Pas de duplication de calcul, le champ existant `charging_sessions.co2_kg_avoided` reste mais sa source devient le résultat variable.

**Tech Stack:** Java 21 + Spring Boot 3.x + JPA (Hibernate) + Flyway 9 + Postgres 16 + JUnit 5 + Mockito + Testcontainers.

**Pré-requis :** Sprint 2 NEXCharge mergé sur `main` (présence de `StopTransactionHandler`, `ChargingSession` entity, `BusinessProperties`, `users` table, OIDC). Branche de travail : `feat/ecotrade-green-score`.

**Coordination collègue :** Avant de commencer, valider §9.1 spec v2 — Flyway V3 réservé pour ce module, package `com.accenture.nexcharge.ecotrade`, remplacement du `co2_kg_avoided` constant accepté.

---

## File Structure

| Fichier | Création / Modif | Responsabilité |
|---|---|---|
| `services/core/src/main/resources/db/migration/V3__green_score.sql` | Create | Table `green_score` + index |
| `services/core/src/main/java/com/accenture/nexcharge/ecotrade/EcotradeProperties.java` | Create | `@ConfigurationProperties("nexcharge.ecotrade")` — lookup table grid + facteurs ICE/EV |
| `services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GridCarbonIntensityResolver.java` | Create | Pure : `(Instant from, Instant to) → averageGCo2PerKwh` |
| `services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScore.java` | Create | `@Entity` |
| `services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreRepository.java` | Create | `JpaRepository` + custom queries leaderboard / sum |
| `services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreService.java` | Create | `record()`, `summaryFor(user)`, `leaderboardTopN()`, `siteAggregate()` |
| `services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/dto/*.java` | Create | DTOs response (`GreenScoreSummaryDto`, `LeaderboardEntryDto`, `SiteAggregateDto`) |
| `services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreController.java` | Create | REST `/api/ecotrade/green-score/*` |
| `services/core/src/main/java/com/accenture/nexcharge/ocpp/handlers/StopTransactionHandler.java` | Modify | Remplace `co2 = kwh * 0.4` par appel resolver + `greenScoreService.record(...)` |
| `services/core/src/main/resources/application.yml` | Modify | Ajoute la section `nexcharge.ecotrade.*` |
| `services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GridCarbonIntensityResolverTest.java` | Create | Unit tests resolver |
| `services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreServiceTest.java` | Create | Unit tests service (Mockito) |
| `services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreControllerIT.java` | Create | Integration test (Testcontainers + MockMvc) |
| `services/core/src/test/java/com/accenture/nexcharge/ocpp/OcppIntegrationIT.java` | Modify | Vérifie qu'après StopTransaction un `GreenScore` row est créé |

---

## Task 1: Migration Flyway V3 — table `green_score`

**Files:**
- Create: `services/core/src/main/resources/db/migration/V3__green_score.sql`
- Test: aucun nouveau test ; le démarrage Spring Boot avec Testcontainers (déjà en place sprint 1) exécute Flyway et plantera si la migration est invalide.

- [ ] **Step 1: Vérifier l'état Flyway local et la disponibilité du numéro V3**

```bash
cd services/core
ls src/main/resources/db/migration/
# Doit lister V1__*.sql (sprint 1) et V2__audit_log.sql (sprint 2). V3 doit être absent.
```

Expected: `V1__init.sql`, `V2__audit_log.sql` présents, **pas de V3**. Si V3 existe déjà → STOP et coordonner avec le collègue.

- [ ] **Step 2: Créer la migration V3**

`services/core/src/main/resources/db/migration/V3__green_score.sql`

```sql
-- V3__green_score.sql
-- Module 1 EcoTrade — score CO2 variable par session

CREATE TABLE green_score (
  id                       UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id                  UUID         NOT NULL REFERENCES users(id),
  session_id               UUID         NOT NULL UNIQUE REFERENCES charging_sessions(id),
  energy_kwh               NUMERIC(10,3) NOT NULL,
  grid_intensity_g_per_kwh NUMERIC(6,1)  NOT NULL,
  co2_avoided_grams        NUMERIC(12,2) NOT NULL,
  green_points             INTEGER       NOT NULL,
  calculated_at            TIMESTAMPTZ   NOT NULL DEFAULT now(),

  CONSTRAINT chk_green_score_kwh_nonneg     CHECK (energy_kwh >= 0),
  CONSTRAINT chk_green_score_co2_nonneg     CHECK (co2_avoided_grams >= 0),
  CONSTRAINT chk_green_score_points_nonneg  CHECK (green_points >= 0)
);

CREATE INDEX idx_green_score_user_calc
  ON green_score(user_id, calculated_at DESC);

CREATE INDEX idx_green_score_points_desc
  ON green_score(green_points DESC, calculated_at DESC);
```

- [ ] **Step 3: Lancer un dry-run Flyway**

Run:
```bash
./gradlew :core:flywayMigrate -i
```

Expected: `Successfully applied 1 migration to schema "public" (execution time ...): V3__green_score.sql`. Si erreur de FK (`charging_sessions` ou `users` absente) → la migration suppose Sprint 2 mergé ; coordonner avec le collègue.

- [ ] **Step 4: Vérifier la table en DB**

Run:
```bash
docker compose exec postgres psql -U nexcharge -d nexcharge -c "\d green_score"
```

Expected: les 7 colonnes + 2 index + 3 check constraints listés.

- [ ] **Step 5: Commit**

```bash
git add services/core/src/main/resources/db/migration/V3__green_score.sql
git commit -m "feat(ecotrade): add Flyway V3 green_score table"
```

---

## Task 2: Configuration `EcotradeProperties` + lookup grid Maurice

**Files:**
- Create: `services/core/src/main/java/com/accenture/nexcharge/ecotrade/EcotradeProperties.java`
- Modify: `services/core/src/main/resources/application.yml`
- Test: `services/core/src/test/java/com/accenture/nexcharge/ecotrade/EcotradePropertiesTest.java`

- [ ] **Step 1: Écrire le test de chargement de la config**

`services/core/src/test/java/com/accenture/nexcharge/ecotrade/EcotradePropertiesTest.java`

```java
package com.accenture.nexcharge.ecotrade;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class EcotradePropertiesTest {

    @Autowired
    EcotradeProperties props;

    @Test
    void loads_grid_lookup_for_weekday_peak() {
        // 8h-10h weekday = peak
        assertThat(props.grid().weekday().peak().value()).isEqualTo(850);
        assertThat(props.grid().weekday().peak().hours()).contains(8, 9, 10, 13, 14);
    }

    @Test
    void loads_thermal_and_ev_factors() {
        assertThat(props.thermalEmissionGPerKm()).isEqualTo(120.0);
        assertThat(props.evEfficiencyKmPerKwh()).isEqualTo(6.0);
    }

    @Test
    void grid_hours_cover_full_day_no_overlap_weekday() {
        var w = props.grid().weekday();
        var allHours = new java.util.HashSet<Integer>();
        allHours.addAll(w.offPeak().hours());
        allHours.addAll(w.shoulder().hours());
        allHours.addAll(w.peak().hours());
        assertThat(allHours).hasSize(24);
    }

    @Test
    void grid_hours_cover_full_day_no_overlap_weekend() {
        var w = props.grid().weekend();
        var allHours = new java.util.HashSet<Integer>();
        allHours.addAll(w.offPeak().hours());
        allHours.addAll(w.shoulder().hours());
        allHours.addAll(w.peak().hours());
        assertThat(allHours).hasSize(24);
    }
}
```

- [ ] **Step 2: Run test (it should fail — no class yet)**

Run: `./gradlew :core:test --tests EcotradePropertiesTest`
Expected: FAIL — `EcotradeProperties` not found / Spring context fails to load.

- [ ] **Step 3: Créer `EcotradeProperties.java`**

`services/core/src/main/java/com/accenture/nexcharge/ecotrade/EcotradeProperties.java`

```java
package com.accenture.nexcharge.ecotrade;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.NotNull;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

import java.util.List;

@ConfigurationProperties(prefix = "nexcharge.ecotrade")
@Validated
public record EcotradeProperties(
        @NotNull Grid grid,
        @Min(0) double thermalEmissionGPerKm,
        @Min(0) double evEfficiencyKmPerKwh
) {

    public record Grid(@NotNull DaySchedule weekday, @NotNull DaySchedule weekend) {}

    public record DaySchedule(
            @NotNull Tranche offPeak,
            @NotNull Tranche shoulder,
            @NotNull Tranche peak
    ) {}

    public record Tranche(@NotEmpty List<Integer> hours, @Min(0) double value) {}
}
```

- [ ] **Step 4: Activer `@ConfigurationProperties` dans la config Spring**

Vérifier qu'une classe `@SpringBootApplication` ou `@EnableConfigurationProperties` existe. Sinon ajouter dans la classe principale (probablement `NexchargeApplication.java`) :

```java
@SpringBootApplication
@ConfigurationPropertiesScan("com.accenture.nexcharge")
public class NexchargeApplication { ... }
```

(Si déjà scanné, no-op.)

- [ ] **Step 5: Ajouter la config dans `application.yml`**

Modifier `services/core/src/main/resources/application.yml` — ajouter sous `nexcharge:` :

```yaml
nexcharge:
  ecotrade:
    thermal-emission-g-per-km: 120.0
    ev-efficiency-km-per-kwh: 6.0
    grid:
      weekday:
        off-peak:
          hours: [0, 1, 2, 3, 4, 5, 22, 23]
          value: 550
        shoulder:
          hours: [6, 7, 11, 12, 15, 16, 17, 21]
          value: 700
        peak:
          hours: [8, 9, 10, 13, 14, 18, 19, 20]
          value: 850
      weekend:
        off-peak:
          hours: [0, 1, 2, 3, 4, 5, 6, 22, 23]
          value: 530
        shoulder:
          hours: [7, 8, 9, 10, 11, 21]
          value: 650
        peak:
          hours: [12, 13, 14, 15, 16, 17, 18, 19, 20]
          value: 780
```

- [ ] **Step 6: Re-run test**

Run: `./gradlew :core:test --tests EcotradePropertiesTest`
Expected: PASS (4/4).

- [ ] **Step 7: Commit**

```bash
git add services/core/src/main/java/com/accenture/nexcharge/ecotrade/EcotradeProperties.java \
        services/core/src/test/java/com/accenture/nexcharge/ecotrade/EcotradePropertiesTest.java \
        services/core/src/main/resources/application.yml
git commit -m "feat(ecotrade): add EcotradeProperties with Mauritius grid lookup"
```

---

## Task 3: `GridCarbonIntensityResolver` — calcul moyenne pondérée

**Files:**
- Create: `services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GridCarbonIntensityResolver.java`
- Test: `services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GridCarbonIntensityResolverTest.java`

**Comportement :**
- Input : `Instant from`, `Instant to` (UTC, mais l'heure de Maurice est UTC+4 — fixée à `ZoneId.of("Indian/Mauritius")` pour le mapping h→tranche).
- Output : `double averageGCo2PerKwh` = moyenne des intensités pondérée par la durée passée dans chaque tranche horaire.
- Si `from == to` ou `from > to` : retourne l'intensité de l'heure `from` (zéro durée → pas de division par zéro).

- [ ] **Step 1: Écrire les tests unitaires**

`services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GridCarbonIntensityResolverTest.java`

```java
package com.accenture.nexcharge.ecotrade.greenscore;

import com.accenture.nexcharge.ecotrade.EcotradeProperties;
import com.accenture.nexcharge.ecotrade.EcotradeProperties.DaySchedule;
import com.accenture.nexcharge.ecotrade.EcotradeProperties.Grid;
import com.accenture.nexcharge.ecotrade.EcotradeProperties.Tranche;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class GridCarbonIntensityResolverTest {

    private static final ZoneId MU = ZoneId.of("Indian/Mauritius");

    private GridCarbonIntensityResolver resolver;

    @BeforeEach
    void setUp() {
        var weekday = new DaySchedule(
                new Tranche(List.of(0, 1, 2, 3, 4, 5, 22, 23), 550),
                new Tranche(List.of(6, 7, 11, 12, 15, 16, 17, 21), 700),
                new Tranche(List.of(8, 9, 10, 13, 14, 18, 19, 20), 850));
        var weekend = new DaySchedule(
                new Tranche(List.of(0, 1, 2, 3, 4, 5, 6, 22, 23), 530),
                new Tranche(List.of(7, 8, 9, 10, 11, 21), 650),
                new Tranche(List.of(12, 13, 14, 15, 16, 17, 18, 19, 20), 780));
        var props = new EcotradeProperties(new Grid(weekday, weekend), 120.0, 6.0);
        resolver = new GridCarbonIntensityResolver(props);
    }

    private Instant atMu(int year, int month, int day, int hour, int minute) {
        return LocalDateTime.of(year, month, day, hour, minute).atZone(MU).toInstant();
    }

    @Test
    void weekday_full_hour_in_peak_tranche_returns_peak_value() {
        // Mardi 2026-05-26 09:00 → 09:30 (tout en peak)
        Instant from = atMu(2026, 5, 26, 9, 0);
        Instant to   = atMu(2026, 5, 26, 9, 30);
        assertThat(resolver.weightedAverage(from, to)).isEqualTo(850.0);
    }

    @Test
    void weekday_full_hour_in_off_peak_returns_off_peak_value() {
        // Mardi 2026-05-26 02:00 → 04:00 (off-peak)
        Instant from = atMu(2026, 5, 26, 2, 0);
        Instant to   = atMu(2026, 5, 26, 4, 0);
        assertThat(resolver.weightedAverage(from, to)).isEqualTo(550.0);
    }

    @Test
    void weekday_window_crossing_two_tranches_weighted_average() {
        // Mardi 2026-05-26 07:30 → 08:30
        // 30min en shoulder (7h, 700) + 30min en peak (8h, 850)
        Instant from = atMu(2026, 5, 26, 7, 30);
        Instant to   = atMu(2026, 5, 26, 8, 30);
        // moyenne = (30*700 + 30*850) / 60 = 775
        assertThat(resolver.weightedAverage(from, to)).isEqualTo(775.0);
    }

    @Test
    void weekday_window_crossing_three_tranches_weighted_average() {
        // Mardi 2026-05-26 21:30 → 23:30 = 30min shoulder(700) + 60min off-peak(550) + 30min off-peak(550)
        Instant from = atMu(2026, 5, 26, 21, 30);
        Instant to   = atMu(2026, 5, 26, 23, 30);
        // (30*700 + 90*550) / 120 = (21000 + 49500)/120 = 587.5
        assertThat(resolver.weightedAverage(from, to)).isEqualTo(587.5);
    }

    @Test
    void weekend_uses_weekend_schedule() {
        // Samedi 2026-05-23 13:00 → 14:00 (peak weekend = 780, pas weekday 850)
        Instant from = atMu(2026, 5, 23, 13, 0);
        Instant to   = atMu(2026, 5, 23, 14, 0);
        assertThat(resolver.weightedAverage(from, to)).isEqualTo(780.0);
    }

    @Test
    void window_crossing_midnight_within_same_local_day_logic() {
        // Mardi 2026-05-26 23:30 → mercredi 2026-05-27 00:30
        // 30min en off-peak weekday (23h, 550) + 30min en off-peak weekday (0h mercredi, 550)
        Instant from = atMu(2026, 5, 26, 23, 30);
        Instant to   = atMu(2026, 5, 27, 0, 30);
        assertThat(resolver.weightedAverage(from, to)).isEqualTo(550.0);
    }

    @Test
    void window_crossing_weekday_to_weekend_boundary() {
        // Vendredi 2026-05-22 23:30 → Samedi 2026-05-23 00:30
        // 30min weekday off-peak (23h, 550) + 30min weekend off-peak (0h sam, 530)
        Instant from = atMu(2026, 5, 22, 23, 30);
        Instant to   = atMu(2026, 5, 23, 0, 30);
        // (30*550 + 30*530)/60 = 540
        assertThat(resolver.weightedAverage(from, to)).isEqualTo(540.0);
    }

    @Test
    void zero_duration_returns_intensity_at_from() {
        Instant from = atMu(2026, 5, 26, 9, 0);
        assertThat(resolver.weightedAverage(from, from)).isEqualTo(850.0);
    }

    @Test
    void inverted_window_throws() {
        Instant from = atMu(2026, 5, 26, 10, 0);
        Instant to   = atMu(2026, 5, 26, 9, 0);
        assertThatThrownBy(() -> resolver.weightedAverage(from, to))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run test (FAIL — class not yet created)**

Run: `./gradlew :core:test --tests GridCarbonIntensityResolverTest`
Expected: FAIL — `GridCarbonIntensityResolver` not found.

- [ ] **Step 3: Implémenter le resolver**

`services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GridCarbonIntensityResolver.java`

```java
package com.accenture.nexcharge.ecotrade.greenscore;

import com.accenture.nexcharge.ecotrade.EcotradeProperties;
import com.accenture.nexcharge.ecotrade.EcotradeProperties.DaySchedule;
import com.accenture.nexcharge.ecotrade.EcotradeProperties.Tranche;
import org.springframework.stereotype.Component;

import java.time.DayOfWeek;
import java.time.Duration;
import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.time.ZonedDateTime;
import java.util.List;

@Component
public class GridCarbonIntensityResolver {

    private static final ZoneId MAURITIUS = ZoneId.of("Indian/Mauritius");

    private final EcotradeProperties props;

    public GridCarbonIntensityResolver(EcotradeProperties props) {
        this.props = props;
    }

    /**
     * Returns the duration-weighted average gCO2/kWh between {@code from} and {@code to}.
     * Window is sliced at each hour boundary in Mauritius local time.
     */
    public double weightedAverage(Instant from, Instant to) {
        if (from.isAfter(to)) {
            throw new IllegalArgumentException("from must be <= to");
        }
        if (from.equals(to)) {
            return intensityAt(from);
        }

        ZonedDateTime cursor = from.atZone(MAURITIUS);
        ZonedDateTime end    = to.atZone(MAURITIUS);
        double totalWeighted = 0.0;
        long   totalSeconds  = 0L;

        while (cursor.isBefore(end)) {
            ZonedDateTime nextHour = cursor.plusHours(1).withMinute(0).withSecond(0).withNano(0);
            ZonedDateTime sliceEnd = nextHour.isAfter(end) ? end : nextHour;
            long sliceSeconds = Duration.between(cursor, sliceEnd).getSeconds();
            double intensity  = intensityForLocalHour(cursor.getDayOfWeek(), cursor.getHour());
            totalWeighted += intensity * sliceSeconds;
            totalSeconds  += sliceSeconds;
            cursor = sliceEnd;
        }

        return totalWeighted / totalSeconds;
    }

    private double intensityAt(Instant t) {
        ZonedDateTime z = t.atZone(MAURITIUS);
        return intensityForLocalHour(z.getDayOfWeek(), z.getHour());
    }

    private double intensityForLocalHour(DayOfWeek dow, int hour) {
        DaySchedule day = isWeekend(dow) ? props.grid().weekend() : props.grid().weekday();
        for (Tranche tr : List.of(day.offPeak(), day.shoulder(), day.peak())) {
            if (tr.hours().contains(hour)) {
                return tr.value();
            }
        }
        throw new IllegalStateException(
                "No tranche covers hour " + hour + " for " + (isWeekend(dow) ? "weekend" : "weekday")
                        + ". Check application.yml grid config.");
    }

    private static boolean isWeekend(DayOfWeek dow) {
        return dow == DayOfWeek.SATURDAY || dow == DayOfWeek.SUNDAY;
    }
}
```

- [ ] **Step 4: Re-run tests**

Run: `./gradlew :core:test --tests GridCarbonIntensityResolverTest`
Expected: PASS (9/9).

- [ ] **Step 5: Commit**

```bash
git add services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GridCarbonIntensityResolver.java \
        services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GridCarbonIntensityResolverTest.java
git commit -m "feat(ecotrade): add GridCarbonIntensityResolver with weighted average"
```

---

## Task 4: Entity `GreenScore` + `GreenScoreRepository`

**Files:**
- Create: `services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScore.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreRepository.java`
- Test: `services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreRepositoryIT.java`

**Hypothèse :** les entités `User` (table `users`) et `ChargingSession` (table `charging_sessions`) existent depuis Sprint 1 dans des packages voisins (probablement `com.accenture.nexcharge.users.User` et `com.accenture.nexcharge.sessions.ChargingSession`). Vérifier les imports précis avant de coder l'entity. Si les noms diffèrent, adapter les `@JoinColumn`.

- [ ] **Step 1: Vérifier les classes d'entité existantes**

Run:
```bash
grep -rn "class ChargingSession" services/core/src/main/java/
grep -rn "class User " services/core/src/main/java/
```

Expected: trouve les classes ; noter le package exact pour les `import`.

- [ ] **Step 2: Écrire le test repository (Testcontainers)**

`services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreRepositoryIT.java`

```java
package com.accenture.nexcharge.ecotrade.greenscore;

// Adjust imports to actual package paths from Step 1
import com.accenture.nexcharge.sessions.ChargingSession;
import com.accenture.nexcharge.users.User;
// ... whatever Sprint 1 test fixtures exist (TestEntityBuilder?)

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.test.context.ActiveProfiles;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@Testcontainers
@ActiveProfiles("test")
class GreenScoreRepositoryIT {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");

    @Autowired GreenScoreRepository repo;

    @Test
    void persists_and_finds_by_session_id() {
        UUID userId    = UUID.randomUUID();
        UUID sessionId = UUID.randomUUID();
        // Note: dans un vrai test on persisterait User + ChargingSession via TestEntityManager.
        // Ici on suppose qu'un fixture builder existe (sprint 1) pour insérer rapidement.
        // À ADAPTER selon ce qui existe vraiment dans le repo.
        var entity = new GreenScore(
                userId, sessionId,
                new BigDecimal("12.345"),
                new BigDecimal("700.0"),
                new BigDecimal("3600.50"),
                36);
        repo.save(entity);

        var found = repo.findBySessionId(sessionId).orElseThrow();
        assertThat(found.getCo2AvoidedGrams()).isEqualByComparingTo("3600.50");
        assertThat(found.getGreenPoints()).isEqualTo(36);
    }

    @Test
    void leaderboard_top_n_orders_by_total_points_desc() {
        UUID alice = UUID.randomUUID();
        UUID bob   = UUID.randomUUID();
        repo.save(new GreenScore(alice, UUID.randomUUID(), bd("10.0"), bd("700"), bd("1000"), 10));
        repo.save(new GreenScore(alice, UUID.randomUUID(), bd("10.0"), bd("700"), bd("2000"), 20));
        repo.save(new GreenScore(bob,   UUID.randomUUID(), bd("10.0"), bd("700"), bd("4000"), 40));

        List<GreenScoreRepository.LeaderboardRow> top = repo.leaderboardTopN(10);
        assertThat(top).hasSize(2);
        assertThat(top.get(0).userId()).isEqualTo(bob);
        assertThat(top.get(0).totalPoints()).isEqualTo(40);
        assertThat(top.get(1).userId()).isEqualTo(alice);
        assertThat(top.get(1).totalPoints()).isEqualTo(30);
    }

    @Test
    void user_summary_aggregates_co2_and_points_for_user() {
        UUID alice = UUID.randomUUID();
        repo.save(new GreenScore(alice, UUID.randomUUID(), bd("5.0"), bd("700"), bd("1500.50"), 15));
        repo.save(new GreenScore(alice, UUID.randomUUID(), bd("5.0"), bd("700"), bd("2500.25"), 25));

        var summary = repo.aggregateForUser(alice).orElseThrow();
        assertThat(summary.totalCo2Grams()).isEqualByComparingTo("4000.75");
        assertThat(summary.totalPoints()).isEqualTo(40);
        assertThat(summary.sessionsCount()).isEqualTo(2);
    }

    private static BigDecimal bd(String s) { return new BigDecimal(s); }
}
```

- [ ] **Step 3: Run test (FAIL — entity & repo absents)**

Run: `./gradlew :core:test --tests GreenScoreRepositoryIT`
Expected: FAIL — compilation error.

- [ ] **Step 4: Implémenter l'entity**

`services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScore.java`

```java
package com.accenture.nexcharge.ecotrade.greenscore;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Index;
import jakarta.persistence.Table;
import org.hibernate.annotations.CreationTimestamp;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(
    name = "green_score",
    indexes = {
        @Index(name = "idx_green_score_user_calc",   columnList = "user_id, calculated_at desc"),
        @Index(name = "idx_green_score_points_desc", columnList = "green_points desc")
    }
)
public class GreenScore {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(name = "user_id", nullable = false)
    private UUID userId;

    @Column(name = "session_id", nullable = false, unique = true)
    private UUID sessionId;

    @Column(name = "energy_kwh", nullable = false, precision = 10, scale = 3)
    private BigDecimal energyKwh;

    @Column(name = "grid_intensity_g_per_kwh", nullable = false, precision = 6, scale = 1)
    private BigDecimal gridIntensityGPerKwh;

    @Column(name = "co2_avoided_grams", nullable = false, precision = 12, scale = 2)
    private BigDecimal co2AvoidedGrams;

    @Column(name = "green_points", nullable = false)
    private int greenPoints;

    @CreationTimestamp
    @Column(name = "calculated_at", nullable = false, updatable = false)
    private Instant calculatedAt;

    protected GreenScore() {}

    public GreenScore(UUID userId, UUID sessionId, BigDecimal energyKwh,
                      BigDecimal gridIntensityGPerKwh, BigDecimal co2AvoidedGrams,
                      int greenPoints) {
        this.userId = userId;
        this.sessionId = sessionId;
        this.energyKwh = energyKwh;
        this.gridIntensityGPerKwh = gridIntensityGPerKwh;
        this.co2AvoidedGrams = co2AvoidedGrams;
        this.greenPoints = greenPoints;
    }

    public UUID getId()                       { return id; }
    public UUID getUserId()                   { return userId; }
    public UUID getSessionId()                { return sessionId; }
    public BigDecimal getEnergyKwh()          { return energyKwh; }
    public BigDecimal getGridIntensityGPerKwh() { return gridIntensityGPerKwh; }
    public BigDecimal getCo2AvoidedGrams()    { return co2AvoidedGrams; }
    public int getGreenPoints()               { return greenPoints; }
    public Instant getCalculatedAt()          { return calculatedAt; }
}
```

- [ ] **Step 5: Implémenter le repository**

`services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreRepository.java`

```java
package com.accenture.nexcharge.ecotrade.greenscore;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface GreenScoreRepository extends JpaRepository<GreenScore, UUID> {

    Optional<GreenScore> findBySessionId(UUID sessionId);

    @Query("""
           SELECT new com.accenture.nexcharge.ecotrade.greenscore.GreenScoreRepository$UserSummary(
               COALESCE(SUM(g.co2AvoidedGrams), 0),
               COALESCE(SUM(g.greenPoints), 0),
               COUNT(g))
           FROM GreenScore g
           WHERE g.userId = :userId
           """)
    Optional<UserSummary> aggregateForUser(@Param("userId") UUID userId);

    @Query(value = """
           SELECT user_id        AS userId,
                  SUM(green_points) AS totalPoints,
                  SUM(co2_avoided_grams) AS totalCo2Grams
           FROM green_score
           GROUP BY user_id
           ORDER BY totalPoints DESC, MAX(calculated_at) DESC
           LIMIT :limit
           """,
           nativeQuery = true)
    List<LeaderboardRow> leaderboardTopN(@Param("limit") int limit);

    record UserSummary(BigDecimal totalCo2Grams, long totalPoints, long sessionsCount) {}

    /** Native projection — Spring Data binds by column alias. */
    interface LeaderboardRow {
        UUID getUserId();
        long getTotalPoints();
        BigDecimal getTotalCo2Grams();

        default UUID userId()           { return getUserId(); }
        default long totalPoints()      { return getTotalPoints(); }
        default BigDecimal totalCo2Grams() { return getTotalCo2Grams(); }
    }
}
```

- [ ] **Step 6: Re-run tests**

Run: `./gradlew :core:test --tests GreenScoreRepositoryIT`
Expected: PASS (3/3). Si la FK `users(id)` ou `charging_sessions(id)` bloque l'insertion (les UUIDs random ne référencent rien), il faut soit (a) instancier des fixtures `User`/`ChargingSession` via `TestEntityManager`, soit (b) en `@DataJpaTest` désactiver les FK avec un `@Sql` qui drop les contraintes en mode test. **Préféré : (a) — adapte le test pour persister des entités liées.** Si Sprint 1 a un builder de fixtures, l'utiliser.

- [ ] **Step 7: Commit**

```bash
git add services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScore.java \
        services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreRepository.java \
        services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreRepositoryIT.java
git commit -m "feat(ecotrade): add GreenScore entity and repository"
```

---

## Task 5: `GreenScoreService.record()` — calcul + persistance

**Files:**
- Create: `services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreService.java`
- Test: `services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreServiceTest.java`

**Formule appliquée :**
```
distance_km          = energy_kwh * ev_efficiency_km_per_kwh
thermal_emissions_g  = distance_km * thermal_emission_g_per_km
grid_emissions_g     = energy_kwh * avg_grid_intensity_g_per_kwh
co2_avoided_g        = max(0, thermal_emissions_g - grid_emissions_g)
green_points         = round(co2_avoided_g / 100)   // 1 point = 100g
```

- [ ] **Step 1: Écrire les tests unitaires (Mockito)**

`services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreServiceTest.java`

```java
package com.accenture.nexcharge.ecotrade.greenscore;

import com.accenture.nexcharge.ecotrade.EcotradeProperties;
import com.accenture.nexcharge.ecotrade.EcotradeProperties.DaySchedule;
import com.accenture.nexcharge.ecotrade.EcotradeProperties.Grid;
import com.accenture.nexcharge.ecotrade.EcotradeProperties.Tranche;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class GreenScoreServiceTest {

    private GreenScoreRepository repo;
    private GridCarbonIntensityResolver resolver;
    private EcotradeProperties props;
    private GreenScoreService service;

    @BeforeEach
    void setUp() {
        repo = mock(GreenScoreRepository.class);
        resolver = mock(GridCarbonIntensityResolver.class);
        var weekday = new DaySchedule(
                new Tranche(List.of(0,1,2,3,4,5,22,23), 550),
                new Tranche(List.of(6,7,11,12,15,16,17,21), 700),
                new Tranche(List.of(8,9,10,13,14,18,19,20), 850));
        var weekend = new DaySchedule(
                new Tranche(List.of(0,1,2,3,4,5,6,22,23), 530),
                new Tranche(List.of(7,8,9,10,11,21), 650),
                new Tranche(List.of(12,13,14,15,16,17,18,19,20), 780));
        props = new EcotradeProperties(new Grid(weekday, weekend), 120.0, 6.0);
        service = new GreenScoreService(repo, resolver, props);
    }

    @Test
    void records_co2_avoided_when_grid_cleaner_than_thermal() {
        UUID userId    = UUID.randomUUID();
        UUID sessionId = UUID.randomUUID();
        Instant from = Instant.parse("2026-05-26T22:00:00Z"); // Mauritius local 02h → off-peak 550
        Instant to   = Instant.parse("2026-05-26T23:00:00Z");
        when(resolver.weightedAverage(from, to)).thenReturn(550.0);

        // 10 kWh × 6 km/kWh × 120 g/km = 7200 g thermal
        // 10 kWh × 550 g/kWh                = 5500 g grid
        // CO2 avoided                       = 1700 g  → 17 points
        var result = service.record(userId, sessionId, new BigDecimal("10.000"), from, to);

        assertThat(result.gridIntensityGPerKwh()).isEqualByComparingTo("550.0");
        assertThat(result.co2AvoidedGrams()).isEqualByComparingTo("1700.00");
        assertThat(result.greenPoints()).isEqualTo(17);

        ArgumentCaptor<GreenScore> captor = ArgumentCaptor.forClass(GreenScore.class);
        verify(repo).save(captor.capture());
        assertThat(captor.getValue().getUserId()).isEqualTo(userId);
        assertThat(captor.getValue().getSessionId()).isEqualTo(sessionId);
    }

    @Test
    void clamps_co2_avoided_to_zero_when_grid_dirtier_than_thermal() {
        UUID userId    = UUID.randomUUID();
        UUID sessionId = UUID.randomUUID();
        Instant from = Instant.parse("2026-05-26T05:00:00Z");
        Instant to   = Instant.parse("2026-05-26T06:00:00Z");
        // intensité fictive très haute
        when(resolver.weightedAverage(from, to)).thenReturn(2000.0);

        // 10 kWh × 6 × 120 = 7200 g thermal ; 10 × 2000 = 20000 g grid → -12800 → clamp à 0
        var result = service.record(userId, sessionId, new BigDecimal("10.000"), from, to);

        assertThat(result.co2AvoidedGrams()).isEqualByComparingTo("0.00");
        assertThat(result.greenPoints()).isZero();
    }

    @Test
    void zero_kwh_results_in_zero_points() {
        UUID userId    = UUID.randomUUID();
        UUID sessionId = UUID.randomUUID();
        Instant from = Instant.parse("2026-05-26T22:00:00Z");
        Instant to   = Instant.parse("2026-05-26T22:30:00Z");
        when(resolver.weightedAverage(from, to)).thenReturn(550.0);

        var result = service.record(userId, sessionId, BigDecimal.ZERO, from, to);

        assertThat(result.co2AvoidedGrams()).isEqualByComparingTo("0.00");
        assertThat(result.greenPoints()).isZero();
    }
}
```

- [ ] **Step 2: Run test (FAIL — service absent)**

Run: `./gradlew :core:test --tests GreenScoreServiceTest`
Expected: FAIL — `GreenScoreService` not found.

- [ ] **Step 3: Implémenter le service**

`services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreService.java`

```java
package com.accenture.nexcharge.ecotrade.greenscore;

import com.accenture.nexcharge.ecotrade.EcotradeProperties;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

@Service
public class GreenScoreService {

    private static final BigDecimal POINTS_DIVISOR = new BigDecimal("100");

    private final GreenScoreRepository repo;
    private final GridCarbonIntensityResolver resolver;
    private final EcotradeProperties props;

    public GreenScoreService(GreenScoreRepository repo,
                             GridCarbonIntensityResolver resolver,
                             EcotradeProperties props) {
        this.repo = repo;
        this.resolver = resolver;
        this.props = props;
    }

    @Transactional
    public RecordResult record(UUID userId, UUID sessionId, BigDecimal energyKwh,
                               Instant startedAt, Instant endedAt) {
        double avgIntensity = resolver.weightedAverage(startedAt, endedAt);

        BigDecimal kwh        = energyKwh.setScale(3, RoundingMode.HALF_UP);
        BigDecimal intensity  = BigDecimal.valueOf(avgIntensity).setScale(1, RoundingMode.HALF_UP);

        BigDecimal thermalG = kwh
                .multiply(BigDecimal.valueOf(props.evEfficiencyKmPerKwh()))
                .multiply(BigDecimal.valueOf(props.thermalEmissionGPerKm()));
        BigDecimal gridG    = kwh.multiply(intensity);
        BigDecimal avoidedG = thermalG.subtract(gridG).max(BigDecimal.ZERO)
                                     .setScale(2, RoundingMode.HALF_UP);

        int points = avoidedG.divide(POINTS_DIVISOR, 0, RoundingMode.HALF_UP).intValueExact();

        var entity = new GreenScore(userId, sessionId, kwh, intensity, avoidedG, points);
        repo.save(entity);

        return new RecordResult(intensity, avoidedG, points);
    }

    public UserSummaryDto summaryFor(UUID userId) {
        var agg = repo.aggregateForUser(userId).orElse(
                new GreenScoreRepository.UserSummary(BigDecimal.ZERO, 0L, 0L));
        return new UserSummaryDto(
                agg.totalCo2Grams().divide(new BigDecimal("1000"), 3, RoundingMode.HALF_UP),
                agg.totalPoints(),
                agg.sessionsCount());
    }

    public List<LeaderboardEntryDto> leaderboardTopN(int limit) {
        return repo.leaderboardTopN(limit).stream()
                .map(row -> new LeaderboardEntryDto(row.userId(), row.totalPoints()))
                .toList();
    }

    public record RecordResult(BigDecimal gridIntensityGPerKwh,
                               BigDecimal co2AvoidedGrams,
                               int greenPoints) {}
}
```

(Les DTOs `UserSummaryDto`, `LeaderboardEntryDto` sont créés dans Task 7 mais référencés ici — créer des records minimaux maintenant pour que ça compile.)

- [ ] **Step 4: Créer les DTOs minimum**

`services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/dto/UserSummaryDto.java`

```java
package com.accenture.nexcharge.ecotrade.greenscore.dto;

import java.math.BigDecimal;

public record UserSummaryDto(BigDecimal totalCo2AvoidedKg, long totalGreenPoints, long sessionsCount) {}
```

`services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/dto/LeaderboardEntryDto.java`

```java
package com.accenture.nexcharge.ecotrade.greenscore.dto;

import java.util.UUID;

public record LeaderboardEntryDto(UUID userId, long totalPoints) {}
```

Adapter les imports dans `GreenScoreService.java` (`import com.accenture.nexcharge.ecotrade.greenscore.dto.*`).

- [ ] **Step 5: Re-run tests**

Run: `./gradlew :core:test --tests GreenScoreServiceTest`
Expected: PASS (3/3).

- [ ] **Step 6: Commit**

```bash
git add services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreService.java \
        services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/dto/ \
        services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreServiceTest.java
git commit -m "feat(ecotrade): add GreenScoreService with variable CO2 calculation"
```

---

## Task 6: Patcher `StopTransactionHandler` — remplacer le facteur constant

**Files:**
- Modify: `services/core/src/main/java/com/accenture/nexcharge/ocpp/handlers/StopTransactionHandler.java`
- Modify: `services/core/src/test/java/com/accenture/nexcharge/ocpp/OcppIntegrationIT.java` (existant)

**Risque** : ce handler est un point chaud du Sprint 2. Lire le code actuel **complet** avant d'éditer. Préserver le calcul `kwh_total` et `peak_power_kw` ; seulement remplacer le bloc CO2 et ajouter l'appel `greenScoreService.record(...)`.

- [ ] **Step 1: Lire `StopTransactionHandler.java` actuel**

Run:
```bash
cat services/core/src/main/java/com/accenture/nexcharge/ocpp/handlers/StopTransactionHandler.java
```

Identifier la ligne où `co2_kg_avoided` est calculé (probablement `session.setCo2KgAvoided(kwhTotal * businessProps.co2FactorKgPerKwh());`). Noter aussi : où est récupéré `userId` (depuis `session.getUser().getId()` ?), `startedAt`, `endedAt`.

- [ ] **Step 2: Adapter le test d'intégration OCPP existant**

Ouvrir `services/core/src/test/java/com/accenture/nexcharge/ocpp/OcppIntegrationIT.java`. Ajouter une assertion à la fin du scénario `StopTransaction` :

```java
// après l'assertion existante sur ChargingSession.kwhTotal :
var greenScore = greenScoreRepository.findBySessionId(session.getId()).orElseThrow();
assertThat(greenScore.getEnergyKwh()).isEqualByComparingTo(session.getKwhTotal());
assertThat(greenScore.getCo2AvoidedGrams()).isPositive();
assertThat(greenScore.getGridIntensityGPerKwh().doubleValue())
        .isBetween(530.0, 850.0); // dans la plage des tranches Maurice
```

Avec `@Autowired GreenScoreRepository greenScoreRepository;` ajouté en haut.

- [ ] **Step 3: Run le test (FAIL — handler ne crée pas encore le green score)**

Run: `./gradlew :core:test --tests OcppIntegrationIT`
Expected: FAIL — `findBySessionId(...)` retourne empty.

- [ ] **Step 4: Patcher le handler**

Modifier `StopTransactionHandler.java` :

1. Injecter dans le constructeur :
```java
private final GreenScoreService greenScoreService;
private final GridCarbonIntensityResolver gridResolver;
private final EcotradeProperties ecotradeProps;

public StopTransactionHandler(/* deps existantes */,
                              GreenScoreService greenScoreService,
                              GridCarbonIntensityResolver gridResolver,
                              EcotradeProperties ecotradeProps) {
    /* ... */
    this.greenScoreService = greenScoreService;
    this.gridResolver = gridResolver;
    this.ecotradeProps = ecotradeProps;
}
```

2. Dans la méthode qui ferme la session, remplacer :
```java
// AVANT :
double co2Kg = kwhTotal * businessProps.co2FactorKgPerKwh();
session.setCo2KgAvoided(co2Kg);

// APRÈS :
var greenResult = greenScoreService.record(
        session.getUser().getId(),    // À ADAPTER selon le getter exact
        session.getId(),
        BigDecimal.valueOf(kwhTotal),
        session.getStartedAt(),
        session.getEndedAt());
session.setCo2KgAvoided(
        greenResult.co2AvoidedGrams()
                   .divide(new BigDecimal("1000"), 3, RoundingMode.HALF_UP)
                   .doubleValue());
```

(Adapter le getter user — vérifier au Step 1 si c'est `session.getUser().getId()` ou `session.getUserId()` direct.)

3. **Cas walk-in (booking_id null, user null)** : si la `ChargingSession` n'a pas d'utilisateur (charge sans booking en sprint 2), skip le green score :
```java
if (session.getUser() != null) {
    greenScoreService.record(...);
} else {
    // walk-in sans user → calcul fallback simple, pas de score persisté
    double avg = gridResolver.weightedAverage(session.getStartedAt(), session.getEndedAt());
    double avoidedKg = Math.max(0,
        kwhTotal * ecotradeProps.evEfficiencyKmPerKwh() * ecotradeProps.thermalEmissionGPerKm() / 1000.0
        - kwhTotal * avg / 1000.0);
    session.setCo2KgAvoided(avoidedKg);
}
```

- [ ] **Step 5: Re-run le test OCPP IT**

Run: `./gradlew :core:test --tests OcppIntegrationIT`
Expected: PASS — green_score créé, intensité dans la plage.

- [ ] **Step 6: Run la suite complète pour vérifier rien d'autre n'est cassé**

Run: `./gradlew :core:test`
Expected: All green. Si un test du sprint 2 utilisait la valeur exacte `0.4` du facteur constant (ex: `assertThat(co2KgAvoided).isEqualTo(kwhTotal * 0.4)`), il faut l'adapter pour vérifier maintenant un range, pas une égalité stricte.

- [ ] **Step 7: Commit**

```bash
git add services/core/src/main/java/com/accenture/nexcharge/ocpp/handlers/StopTransactionHandler.java \
        services/core/src/test/java/com/accenture/nexcharge/ocpp/OcppIntegrationIT.java
git commit -m "feat(ecotrade): use variable grid intensity in StopTransactionHandler"
```

---

## Task 7: Controller REST `/api/ecotrade/green-score/*`

**Files:**
- Create: `services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreController.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/dto/SiteAggregateDto.java`
- Test: `services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreControllerIT.java`

- [ ] **Step 1: Créer le DTO site aggregate**

`services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/dto/SiteAggregateDto.java`

```java
package com.accenture.nexcharge.ecotrade.greenscore.dto;

import java.math.BigDecimal;

public record SiteAggregateDto(String site,
                               BigDecimal totalCo2AvoidedKg,
                               long totalPoints,
                               long sessionsCount) {}
```

- [ ] **Step 2: Écrire le test d'intégration controller**

`services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreControllerIT.java`

```java
package com.accenture.nexcharge.ecotrade.greenscore;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
class GreenScoreControllerIT {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");

    @Autowired MockMvc mvc;

    @Test
    @WithMockUser(roles = "DRIVER")
    void driver_can_get_their_summary() throws Exception {
        mvc.perform(get("/api/ecotrade/green-score/me"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.totalCo2AvoidedKg").exists())
           .andExpect(jsonPath("$.totalGreenPoints").exists())
           .andExpect(jsonPath("$.sessionsCount").exists());
    }

    @Test
    @WithMockUser(roles = "DRIVER")
    void driver_can_get_leaderboard() throws Exception {
        mvc.perform(get("/api/ecotrade/green-score/leaderboard"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$").isArray());
    }

    @Test
    @WithMockUser(roles = "DRIVER")
    void driver_cannot_access_site_aggregate() throws Exception {
        mvc.perform(get("/api/ecotrade/green-score/site/NEX_TOWER"))
           .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "FACILITY_MANAGER")
    void fm_can_access_site_aggregate() throws Exception {
        mvc.perform(get("/api/ecotrade/green-score/site/NEX_TOWER"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.site").value("NEX_TOWER"))
           .andExpect(jsonPath("$.totalCo2AvoidedKg").exists());
    }

    @Test
    void anonymous_is_unauthorized() throws Exception {
        mvc.perform(get("/api/ecotrade/green-score/me"))
           .andExpect(status().isUnauthorized());
    }
}
```

- [ ] **Step 3: Run test (FAIL — controller absent)**

Run: `./gradlew :core:test --tests GreenScoreControllerIT`
Expected: FAIL — 404 sur les routes.

- [ ] **Step 4: Étendre le service avec `siteAggregate`**

Ajouter dans `GreenScoreService.java` :

```java
public SiteAggregateDto siteAggregate(String siteCode) {
    var rows = repo.siteAggregate(siteCode);
    long totalPoints = 0;
    BigDecimal totalCo2 = BigDecimal.ZERO;
    long sessions = 0;
    for (var r : rows) {
        totalPoints += r.totalPoints();
        totalCo2     = totalCo2.add(r.totalCo2Grams());
        sessions    += r.sessionsCount();
    }
    return new SiteAggregateDto(
            siteCode,
            totalCo2.divide(new BigDecimal("1000"), 3, RoundingMode.HALF_UP),
            totalPoints,
            sessions);
}
```

Et ajouter la query au repository :

```java
@Query(value = """
       SELECT COUNT(g.id) AS sessionsCount,
              COALESCE(SUM(g.green_points), 0) AS totalPoints,
              COALESCE(SUM(g.co2_avoided_grams), 0) AS totalCo2Grams
       FROM green_score g
       JOIN charging_sessions s ON s.id = g.session_id
       JOIN chargers c ON c.id = s.charger_id
       WHERE c.site = :site
       """,
       nativeQuery = true)
List<SiteRow> siteAggregate(@Param("site") String site);

interface SiteRow {
    long getSessionsCount();
    long getTotalPoints();
    BigDecimal getTotalCo2Grams();

    default long sessionsCount()      { return getSessionsCount(); }
    default long totalPoints()        { return getTotalPoints(); }
    default BigDecimal totalCo2Grams() { return getTotalCo2Grams(); }
}
```

(Vérifier le nom exact de la colonne `site` sur `chargers` — Sprint 1 / 2 utilisent probablement un enum stocké en VARCHAR.)

- [ ] **Step 5: Implémenter le controller**

`services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreController.java`

```java
package com.accenture.nexcharge.ecotrade.greenscore;

import com.accenture.nexcharge.ecotrade.greenscore.dto.LeaderboardEntryDto;
import com.accenture.nexcharge.ecotrade.greenscore.dto.SiteAggregateDto;
import com.accenture.nexcharge.ecotrade.greenscore.dto.UserSummaryDto;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;
import java.util.UUID;

@RestController
@RequestMapping("/api/ecotrade/green-score")
public class GreenScoreController {

    private final GreenScoreService service;
    // À ADAPTER : NEXCharge expose probablement un service `CurrentUserService` ou
    // résout l'utilisateur via Principal/JWT. Inject ce qui existe.

    public GreenScoreController(GreenScoreService service) {
        this.service = service;
    }

    @GetMapping("/me")
    @PreAuthorize("hasAnyRole('DRIVER','FACILITY_MANAGER','SUSTAINABILITY_OFFICER','ADMIN')")
    public UserSummaryDto me(@AuthenticationPrincipal Object principal) {
        UUID userId = resolveUserId(principal); // helper à câbler avec le mécanisme NEXCharge
        return service.summaryFor(userId);
    }

    @GetMapping("/leaderboard")
    @PreAuthorize("hasAnyRole('DRIVER','FACILITY_MANAGER','SUSTAINABILITY_OFFICER','ADMIN')")
    public List<LeaderboardEntryDto> leaderboard(@RequestParam(defaultValue = "20") int limit) {
        return service.leaderboardTopN(Math.min(limit, 100));
    }

    @GetMapping("/site/{site}")
    @PreAuthorize("hasAnyRole('FACILITY_MANAGER','SUSTAINABILITY_OFFICER','ADMIN')")
    public SiteAggregateDto site(@PathVariable String site) {
        return service.siteAggregate(site);
    }

    private UUID resolveUserId(Object principal) {
        // Sprint 1 résout l'OID Entra → User.id. Brancher sur ce mécanisme.
        // Placeholder explicite à remplacer pendant l'exécution :
        throw new UnsupportedOperationException(
                "Wire to NEXCharge CurrentUserService — see SecurityConfig sprint 1");
    }
}
```

**Note** : le `resolveUserId` est l'unique point d'intégration manuelle. Pendant l'implémentation, identifier comment Sprint 1 remonte l'`User.id` depuis le JWT (probablement `principal.getName()` mappé via `UserService.findByEntraOid`) et brancher.

- [ ] **Step 6: Re-run tests controller**

Run: `./gradlew :core:test --tests GreenScoreControllerIT`
Expected: PASS (5/5). Si KO sur `/me` à cause du `resolveUserId` non câblé — fixer le câblage et re-run.

- [ ] **Step 7: Commit**

```bash
git add services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreController.java \
        services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreService.java \
        services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreRepository.java \
        services/core/src/main/java/com/accenture/nexcharge/ecotrade/greenscore/dto/SiteAggregateDto.java \
        services/core/src/test/java/com/accenture/nexcharge/ecotrade/greenscore/GreenScoreControllerIT.java
git commit -m "feat(ecotrade): expose green-score REST endpoints (me, leaderboard, site)"
```

---

## Task 8: Test démo end-to-end + check final

**Files:**
- Aucune création — vérification manuelle.

- [ ] **Step 1: Lancer la stack complète**

Run:
```bash
make up
```

Expected: tous les services up, `core` healthy, `ocpp-simulator` connecté.

- [ ] **Step 2: Déclencher une session simulée**

Run:
```bash
make demo-session
```

Expected: `202 Accepted`, `transactionId` retourné.

- [ ] **Step 3: Attendre la fin (~2 min) puis vérifier la DB**

Run:
```bash
docker compose exec postgres psql -U nexcharge -d nexcharge -c \
  "SELECT id, energy_kwh, grid_intensity_g_per_kwh, co2_avoided_grams, green_points FROM green_score ORDER BY calculated_at DESC LIMIT 1;"
```

Expected: une row avec `grid_intensity_g_per_kwh` ∈ [530, 850], `co2_avoided_grams > 0` (sauf si l'heure de simulation est en peak fort où ça peut être 0).

- [ ] **Step 4: Tester l'endpoint REST**

Run (avec un JWT valide, ou via `curl` sur le port traefik si pas d'auth en dev):
```bash
curl -s -H "Authorization: Bearer $JWT" http://localhost/api/ecotrade/green-score/leaderboard | jq
```

Expected: array JSON, possiblement vide si pas de seed, sinon top users.

- [ ] **Step 5: Run tous les tests**

Run:
```bash
./gradlew :core:test
```

Expected: BUILD SUCCESSFUL, ≥ 70% coverage sur `ecotrade/greenscore/*` (vérifier le rapport `build/reports/jacoco/test/html/index.html`).

- [ ] **Step 6: Push et ouvrir la PR**

Run:
```bash
git push -u origin feat/ecotrade-green-score
gh pr create --title "feat(ecotrade): Module 1 — variable grid green score" --body "$(cat <<'EOF'
## Summary
- Replaces constant CO2 factor (0.4 kg/kWh) in `StopTransactionHandler` with hour-of-day variable lookup based on Mauritius grid (550-850 gCO2/kWh).
- Adds `green_score` table (Flyway V3), entity, repository, service.
- Exposes `/api/ecotrade/green-score/{me,leaderboard,site/:site}` with RBAC.
- Reuses existing OCPP integration test to verify a `green_score` row is created on `StopTransaction`.

## Coordination
- Flyway V3 reserved per spec v2 §9.1.
- `co2_kg_avoided` is now sourced from the variable calculation (kept as field, source changed).

## Test plan
- [ ] `./gradlew :core:test` green
- [ ] `make up && make demo-session` then verify a row in `green_score`
- [ ] `curl /api/ecotrade/green-score/leaderboard` returns top users
EOF
)"
```

---

## Self-review

**Spec coverage** (vs `EcoTradeHub_Layer_Spec.md` v2 §3) :
- §3.1 concept variable horaire ✅ Task 3 (resolver), Task 5 (formule)
- §3.2 lookup table grid Maurice ✅ Task 2 (`application.yml`)
- §3.3 modifications NEXCharge (`StopTransactionHandler`, `BusinessProperties`) ✅ Task 6 + Task 2
- §3.4 migration Flyway V3 ✅ Task 1
- §3.5 patch `StopTransactionHandler` pseudocode ✅ Task 6
- §3.6 API endpoints `/me`, `/leaderboard`, `/site/{site}` ✅ Task 7
- §3.7 tests resolver / service / controller IT / OCPP IT modif ✅ Tasks 3, 5, 6, 7

**Placeholders** : aucun "TBD" / "TODO" / "implement later" dans les steps. Deux endroits sont **explicitement marqués "À ADAPTER"** car ils dépendent du code Sprint 1 réel :
1. Imports de `User` / `ChargingSession` (Task 4 Step 1) — la commande `grep` est fournie pour identifier les paths.
2. `resolveUserId(Object principal)` (Task 7 Step 5) — câblage sur le mécanisme JWT/Entra existant.

Ces points sont nécessairement contextuels au repo NEXCharge réel, pas des placeholders paresseux. L'engineer trouve l'info via les commandes `grep` fournies.

**Type consistency** :
- `RecordResult` (Task 5) — utilisé en Task 6 ✅
- `UserSummaryDto`, `LeaderboardEntryDto` (Task 5 Step 4) — utilisés en Task 7 ✅
- `SiteAggregateDto` (Task 7 Step 1) — utilisé en Task 7 service ✅
- `GreenScoreRepository.UserSummary`, `LeaderboardRow`, `SiteRow` — projections cohérentes via méthodes default ✅
- `GridCarbonIntensityResolver.weightedAverage(Instant, Instant)` signature stable Tasks 3/5/6 ✅
