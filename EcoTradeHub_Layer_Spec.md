# EcoTrade Hub — Couche de Différenciation NEXCharge (v2)

> **Contexte** : Cette spec décrit comment intégrer la couche **EcoTrade Hub** (sustainability + compliance, inspirée du GTIC Season 7) **directement dans le monorepo NEXCharge** en s'alignant sur sa stack (Java 21 + Spring Boot 3 + JPA + Flyway + Postgres + Python FastAPI + Next.js 15) et en **réutilisant** ce qui existe au lieu de le dupliquer.
>
> **v1 → v2 — changements clés** :
> - Stack TS/Prisma abandonnée → tout passe en Java/Spring (DB) + Python (AI) + Next.js (UI), comme NEXCharge.
> - Module "Predictive Dashboard" v1 **supprimé** (recouvrement avec le service AI Prophet/LightGBM existant). Ce qui reste est repackagé comme **extension** du service AI.
> - Schémas Prisma → entities JPA + migrations Flyway V3 / V4.
> - Nouvelle section : **Coordination avec NEXCharge** (Flyway numbering, branches, naming).
> - Pitch reformulé : "augmente NEXCharge", pas "couche par-dessus".

---

## 1. Vue d'ensemble

NEXCharge (Sprint 1-4) couvre booking équitable, OCPP, Live Map, IA forecasting, ESG report narré, audit log append-only.

EcoTrade Hub ajoute **3 modules + 1 UI** :

| # | Module | Type d'intervention | Stack |
|---|---|---|---|
| 1 | **Green Score variable horaire** | **Remplace** le calcul CO2 constant existant + nouvelle entity score par session/user | Java / Spring |
| 2 | **Audit Hash Chain** | **Complète** `audit_log` append-only avec une chaîne SHA-256 vérifiable end-to-end sur les sessions | Java / Spring |
| 3 | **Carbon Projection** | **Étend** le service AI Python (qui fait déjà Prophet) avec une projection CO2 évité 3/6/12 mois alignée cibles gouvernementales | Python FastAPI |
| 4 | **Sustainability Hub UI** | **Nouveau** : pages `/sustain/*` (badge profil, leaderboard, audit verifier, carbon projection chart) | Next.js / React |

**Objectif différenciation** : passer d'un système de booking + reporting (NEXCharge baseline) à une plateforme d'intelligence énergétique **traçable** (CO2 réaliste selon mix grid + chaîne d'intégrité prouvable).

---

## 2. Features NEXCharge existantes — réutiliser, étendre, remplacer

| Feature NEXCharge | Notre approche | Détail |
|---|---|---|
| `ChargingSession.co2_kg_avoided` (facteur constant `0.4` au `StopTransaction`) | **Remplacer** | Calcul variable selon heure + véhicule. Cf §3. |
| `audit_log` (append-only, action générique) | **Compléter** | On garde `audit_log`. On ajoute une nouvelle table `audit_chain` chaînée par hash SHA-256 sur les sessions terminées. Deux dispositifs complémentaires (audit métier vs intégrité crypto). Cf §4. |
| Service AI Python (Prophet/LightGBM forecasting demande, anomaly, fairness, ESG narration) | **Étendre** | Nouvel endpoint `POST /ai/sustain/carbon-projection`. Pas de nouveau service. Cf §5. |
| `ai_explanation` (texte d'explication par décision IA) | **Réutiliser** | La projection carbone écrit dans `ai_explanation` comme les autres décisions IA. |
| ESG report mensuel narré | **Réutiliser tel quel** | Pas de duplication. Notre projection carbone alimente la narration ESG existante (la query agrège nos green scores). |
| Live Map, Booking, OCPP, RBAC, OIDC | **Réutiliser tel quel** | Aucune modif. |
| Dashboards FM / Sustainability Officer (Sprint 4) | **Étendre** | Le dashboard SO embarque notre carbon projection chart + leaderboard via les routes Next.js qu'on ajoute. |

---

## 3. Module 1 — Green Score variable horaire

### 3.1 Concept

Le facteur d'émission constant `0.4 kg/kWh` actuel ignore que **le mix énergétique de Maurice varie fortement selon l'heure** : ~550 g/kWh la nuit (peakers à l'arrêt) vs ~850 g/kWh aux heures de pointe (mix fossile dominant).

On calcule, **par session de charge** :
- `gridCarbonIntensity` selon heure + jour (lookup table)
- `co2AvoidedGrams = thermalEquivalentEmissions − gridEmissions`
- `greenPoints = round(co2AvoidedGrams / 100)` (échelle simple pour le leaderboard)

Et on remplace `ChargingSession.co2_kg_avoided` par le résultat agrégé sur la session (intégrale sur les MeterValues, ou approximation à partir de `kwh_total` × intensité moyenne pondérée par la fenêtre horaire).

### 3.2 Lookup table grid Maurice (statique, dans `BusinessProperties`)

```yaml
nexcharge:
  business:
    grid:
      carbon-intensity:
        # gCO2/kWh par tranche horaire (0-23h), week-day
        weekday:
          off-peak:   { hours: [0, 1, 2, 3, 4, 5, 22, 23], value: 550 }
          shoulder:   { hours: [6, 7, 11, 12, 15, 16, 17, 21],     value: 700 }
          peak:       { hours: [8, 9, 10, 13, 14, 18, 19, 20],     value: 850 }
        weekend:
          off-peak:   { hours: [0, 1, 2, 3, 4, 5, 6, 22, 23], value: 530 }
          shoulder:   { hours: [7, 8, 9, 10, 11, 21],              value: 650 }
          peak:       { hours: [12, 13, 14, 15, 16, 17, 18, 19, 20], value: 780 }
      thermal-emission-g-per-km: 120     # ICE moyen
      ev-efficiency-km-per-kwh: 6.0      # EV moyen
```

(Valeurs de référence à valider avec le sustainability officer / dataset CEB Maurice si dispo.)

### 3.3 Modifications NEXCharge

**Fichiers à modifier** :
- `services/core/src/main/java/com/accenture/nexcharge/ocpp/handlers/StopTransactionHandler.java`
- `services/core/src/main/java/com/accenture/nexcharge/common/BusinessProperties.java` (déjà existant — ajouter sous-classe `Grid`)

**Nouveaux fichiers** :
```
services/core/src/main/java/com/accenture/nexcharge/ecotrade/
├── greenscore/
│   ├── GreenScore.java                  // @Entity
│   ├── GreenScoreRepository.java        // JpaRepository
│   ├── GreenScoreService.java           // calcul + persistance
│   ├── GreenScoreController.java        // GET /api/ecotrade/green-score/me, /:userId
│   └── GridCarbonIntensityResolver.java // lookup table → gCO2/kWh selon timestamp
└── EcotradeProperties.java              // @ConfigurationProperties("nexcharge.business.grid")
```

### 3.4 Migration Flyway V3

```sql
-- V3__green_score.sql
CREATE TABLE green_score (
  id                       UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id                  UUID NOT NULL REFERENCES users(id),
  session_id               UUID NOT NULL UNIQUE REFERENCES charging_sessions(id),
  energy_kwh               NUMERIC(10,3) NOT NULL,
  grid_intensity_g_per_kwh NUMERIC(6,1) NOT NULL,  -- moyenne pondérée sur la fenêtre
  co2_avoided_grams        NUMERIC(12,2) NOT NULL,
  green_points             INTEGER NOT NULL,
  calculated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_green_score_user ON green_score(user_id, calculated_at DESC);
```

`charging_sessions.co2_kg_avoided` reste, mais sa source de calcul change (cf §3.5). Pas de modif destructive.

### 3.5 Patch `StopTransactionHandler`

Pseudocode :

```java
// Avant (Sprint 2) :
double co2Kg = kwhTotal * businessProps.co2FactorKgPerKwh();  // 0.4 constant

// Après :
double avgGridIntensity = gridResolver.weightedAverage(
    session.getStartedAt(), session.getEndedAt());  // gCO2/kWh
double thermalEquivG = kwhTotal * businessProps.evEfficiencyKmPerKwh()
                              * businessProps.thermalEmissionGPerKm();
double gridG = kwhTotal * avgGridIntensity;
double co2AvoidedG = Math.max(0, thermalEquivG - gridG);

session.setCo2KgAvoided(co2AvoidedG / 1000.0);
greenScoreService.record(session, kwhTotal, avgGridIntensity, co2AvoidedG);
```

### 3.6 API

| Méthode | Path | Auth | Body / Response |
|---|---|---|---|
| GET | `/api/ecotrade/green-score/me` | DRIVER | `{totalCo2AvoidedKg, totalGreenPoints, sessionsCount, rank, percentile}` |
| GET | `/api/ecotrade/green-score/leaderboard` | DRIVER+ | Top 20 anonymisé : `[{rank, displayInitials, greenPoints}]` |
| GET | `/api/ecotrade/green-score/site/{site}` | FACILITY_MANAGER+ | Agrégat par site (NEX_TOWER vs NEXTERACOM) |

### 3.7 Tests

- `GridCarbonIntensityResolverTest` — chaque tranche horaire, weekend vs weekday, fenêtre cross-tranche (ex: charge 22h→2h : moyenne pondérée correcte).
- `GreenScoreServiceTest` — unit, calcul CO2 évité conforme à la formule, edge cases (kwh=0, fenêtre <1min).
- `GreenScoreControllerIT` — leaderboard, RBAC (DRIVER ne voit pas `/site/*`), pagination.
- Modif `OcppIntegrationIT` (existant) — vérifier qu'après `StopTransaction`, `green_score` row créée + `charging_sessions.co2_kg_avoided` cohérent.

---

## 4. Module 2 — Audit Hash Chain

### 4.1 Concept

`audit_log` (Sprint 2 V2) est append-only par convention applicative : aucun `UPDATE`/`DELETE` exposé, mais rien n'empêche techniquement une modif directe en DB. **Le hash chain rend toute modification détectable** : chaque session terminée écrit un record contenant le SHA-256 du précédent. Toute mutation casse la chaîne ; la vérification est `O(n)` lecture.

C'est inspiré de l'approche blockchain d'EcoTrade Hub GTIC, sans la complexité d'une vraie blockchain (pas de consensus, pas de réseau — juste l'intégrité).

**Complémentaire** à `audit_log` :
- `audit_log` = log métier riche (RBAC change, override booking, raisons textuelles)
- `audit_chain` = preuve d'intégrité crypto sur les données soumises au régulateur (sessions, kWh, CO2)

### 4.2 Migration Flyway V4

```sql
-- V4__audit_chain.sql
CREATE TABLE audit_chain (
  seq           BIGSERIAL PRIMARY KEY,           -- ordre déterministe
  session_id    UUID NOT NULL UNIQUE REFERENCES charging_sessions(id),
  payload_json  JSONB NOT NULL,                  -- snapshot canonique (cf §4.4)
  prev_hash     CHAR(64) NOT NULL,               -- hex SHA-256 du record précédent
  hash          CHAR(64) NOT NULL UNIQUE,        -- hex SHA-256(prev_hash || payload_canonical || timestamp)
  recorded_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_chain_recorded_at ON audit_chain(recorded_at);
-- En prod : REVOKE UPDATE, DELETE ON audit_chain FROM <app_role>;
```

Genesis : premier record → `prev_hash = '0' * 64`.

### 4.3 Nouveaux fichiers

```
services/core/src/main/java/com/accenture/nexcharge/ecotrade/audit/
├── AuditChainEntry.java              // @Entity (no setters on hash/prevHash/seq)
├── AuditChainRepository.java
├── AuditChainService.java            // append() + verifyChain()
├── AuditChainController.java         // GET /api/ecotrade/audit/verify, GET /api/ecotrade/audit/records
└── PayloadCanonicalizer.java         // serialise déterministe (clé triées, UTC, format fixé)
```

### 4.4 Sérialisation canonique du payload

**Critique** pour la vérification reproductible. Format figé :

```json
{
  "sessionId": "uuid",
  "userId": "uuid|null",
  "chargerId": "uuid",
  "ocppId": "SIM-NEX-001",
  "startedAt": "2026-05-22T14:30:00.000Z",
  "endedAt":   "2026-05-22T15:15:00.000Z",
  "kwhTotal": 12.345,
  "co2AvoidedGrams": 2580.5
}
```

Règles : clés triées alphabétiquement, timestamps UTC ISO-8601 ms, nombres avec 3 décimales pour kWh / 1 décimale pour CO2, pas de whitespace. Implémenté avec un `JsonMapper` Jackson dédié, **versionné** (`AUDIT_CHAIN_PAYLOAD_VERSION = "1.0"` stocké dans le payload).

### 4.5 Hash

```
hash = SHA-256( prev_hash + canonicalPayload + recordedAt.toString() )
```

`recordedAt` figé au moment du `append()` et inclus dans le payload `before` hashing pour éviter qu'on puisse modifier l'ordre.

### 4.6 Hook d'écriture

`StopTransactionHandler` (déjà patché en §3.5) appelle en fin :

```java
auditChainService.append(session);
```

**Idempotence** : `audit_chain.session_id` UNIQUE → `INSERT ... ON CONFLICT DO NOTHING` ; double `StopTransaction` (replay OCPP) ne casse pas la chaîne.

**Atomicité** : l'append fait partie de la même transaction que la fermeture de session ; si la DB rollback, pas de record orphelin.

### 4.7 API de vérification

```
GET /api/ecotrade/audit/verify
Auth: ADMIN ou SUSTAINABILITY_OFFICER

Response:
{
  "totalRecords": 1247,
  "verified": 1247,
  "broken": false,
  "brokenAtSeq": null,
  "lastVerifiedHash": "a3f...",
  "verifiedAt": "2026-05-22T16:00:00Z",
  "durationMs": 89
}
```

```
GET /api/ecotrade/audit/records?from=&to=&limit=
Auth: ADMIN ou SUSTAINABILITY_OFFICER
Response: liste paginée (timeline style blockchain explorer)
```

Côté UI (cf §6) : bouton "Verify integrity now" + timeline visuelle.

### 4.8 Tests

- `PayloadCanonicalizerTest` — sérialisation déterministe, ordre des clés, timestamps UTC, nombres formatés.
- `AuditChainServiceTest` — append séquentiel, chaîne valide, idempotence sur double-append.
- `AuditChainCorruptionIT` — Testcontainers Postgres : insert N records, modifie un payload en SQL direct, vérifie que `verifyChain()` détecte la cassure au bon `seq`.
- `AuditChainConcurrencyIT` — 10 sessions terminent en parallèle ; chaîne reste cohérente (lock applicatif sur insert via `SELECT pg_advisory_xact_lock`).

### 4.9 Détail concurrence

**Problème** : si deux `StopTransaction` arrivent en même temps, deux threads peuvent lire le même `prev_hash` et créer une fork. **Solution** : `pg_advisory_xact_lock(<constant>)` au début de `append()` — sérialise les insertions sur la chaîne. Ne bloque rien d'autre dans l'app.

---

## 5. Module 3 — Carbon Projection (extension service AI)

### 5.1 Concept

Le service Python AI fait déjà la prédiction de demande (Prophet/LightGBM) et la narration ESG (Claude). On ajoute **un endpoint dédié** qui projette les économies CO2 sur 3/6/12 mois en se basant sur :
- Trend historique green scores (input depuis Postgres, via une feature view dédiée)
- Trend croissance utilisateurs EV (vues `feature_view_*` existantes)
- Cibles gouvernementales 2026 (config statique, dataset publique Maurice)

Le calcul est fait côté Python (cohérent avec le reste de l'AI), pas côté Java (qui ne fait que servir le résultat via le `AiClient` existant).

### 5.2 Endpoint FastAPI

```python
# apps/ai/src/sustain/router.py
@router.post("/sustain/carbon-projection")
def carbon_projection(req: CarbonProjectionRequest) -> CarbonProjectionResponse:
    """
    Inputs: horizon_months (3|6|12), site_id (optional)
    Outputs: monthly projected CO2 avoided + confidence interval + government target
    """
```

Utilise `statsmodels` ou `Prophet` selon la même approche que `forecasting/`.

### 5.3 Côté Java

Étendre `AiClient` (existant) :

```java
public CarbonProjectionResponse fetchCarbonProjection(int horizonMonths, UUID siteId);
```

Et un controller :

```
GET /api/ecotrade/carbon-projection?horizon=6&siteId=...
Auth: SUSTAINABILITY_OFFICER+
```

### 5.4 Persistance optionnelle

Si on veut historiser les projections (utile pour comparer projection vs réel a posteriori), une table légère :

```sql
-- V5__carbon_projection.sql (sprint 4 si temps)
CREATE TABLE carbon_projection (
  id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  site_id             UUID REFERENCES charging_sites(id),
  month               DATE NOT NULL,
  projected_co2_kg    NUMERIC(12,2) NOT NULL,
  actual_co2_kg       NUMERIC(12,2),       -- rempli a posteriori
  government_target_kg NUMERIC(12,2),
  generated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Skip si la démo se contente d'un calcul live à chaque appel (recommandé pour le hackathon).

### 5.5 Tests

- `test_carbon_projection.py` (pytest) — projection sur dataset synthétique, IC > 0.
- `AiClientCarbonProjectionIT` — Java côté core, mock du service AI ou Testcontainers FastAPI.

---

## 6. Module 4 — Sustainability Hub UI

### 6.1 Pages Next.js (`apps/web/src/app/`)

| Path | Composant | Rôle | Auth |
|---|---|---|---|
| `/sustain` | `SustainHubLanding` | Landing : badge user + leaderboard top 5 + lien vers détails | DRIVER+ |
| `/sustain/leaderboard` | `Leaderboard` | Top 20 anonymisé, filtre site / mois | DRIVER+ |
| `/sustain/projection` | `CarbonProjectionView` | Line chart projection vs cibles, sélecteur horizon | SUSTAINABILITY_OFFICER+ |
| `/sustain/audit` | `AuditChainExplorer` | Timeline + bouton "Verify now" + indicateur intégrité vert/rouge | SUSTAINABILITY_OFFICER+ |
| Profil user (existant) | + `<GreenScoreBadge />` | Badge circulaire, équivalent "X arbres", rang | DRIVER |

### 6.2 Composants (`apps/web/src/components/sustain/`)

```
GreenScoreBadge.tsx          // badge circulaire SVG, animation count-up
Leaderboard.tsx              // tableau anonymisé, filtre, ma position highlightée
CarbonProjectionChart.tsx    // line chart (Recharts) projection + cible + zone confiance
AuditChainTimeline.tsx       // liste paginée avec hash tronqué + détail expand
AuditVerifierButton.tsx      // bouton + spinner + result modal (verified/broken)
TreeEquivalent.tsx           // helper : convertit kg CO2 → "équivalent à N arbres plantés"
```

Tous en TypeScript, alignés sur le design system `apps/web` (Tailwind + shadcn/ui ajouté en Sprint 4).

### 6.3 API client

Étendre `apps/web/src/lib/api-client.ts` (existant) avec les routes `/api/ecotrade/*`. Idéalement, le code TS est régénéré depuis l'OpenAPI Java (`openapi-typescript-codegen`, déjà en place selon NEXCharge §2).

### 6.4 Sidebar / nav

Ajouter un onglet "Sustainability" dans la sidebar principale `apps/web/src/components/layout/Sidebar.tsx`, visible pour tous les rôles avec sous-items conditionnels selon RBAC.

### 6.5 Tests

- Vitest sur `GreenScoreBadge`, `TreeEquivalent` (logique d'affichage).
- Playwright (`tests/sustain-flow.spec.ts`, skipped par défaut comme les autres E2E NEXCharge) : login DRIVER → /sustain → voir badge → /sustain/leaderboard → voir mon rang. Login SO → /sustain/audit → bouton verify → voir "verified".

---

## 7. Données de démo

NEXCharge a déjà un seed Sprint 1 (utilisateurs, chargers via `BootNotification`). Pour rendre la couche EcoTrade visuellement riche, on étend le seed :

**Approche** : un composant Java `EcotradeSeedRunner` (`@Profile("seed")`) qui :
1. Génère ~500 `ChargingSession` (et leurs `MeterValue`) sur les 30 derniers jours, **avec patterns horaires réalistes** (pic 8h-10h et 13h-14h, creux weekend).
2. À chaque session, le `StopTransactionHandler` patché calcule automatiquement `green_score` + `audit_chain` (pas de code dupliqué — on réutilise le hot path).
3. Les `carbon_projection` ne sont pas pré-générées : calcul live à l'appel.

**Pas de seed Prisma**, pas de scripts TS séparés. Tout dans le module Java existant.

---

## 8. Structure de fichiers (alignée monorepo NEXCharge)

```
nexcharge/
├── services/core/src/main/java/com/accenture/nexcharge/
│   └── ecotrade/                              ← NOUVEAU package
│       ├── EcotradeProperties.java
│       ├── greenscore/
│       │   ├── GreenScore.java
│       │   ├── GreenScoreRepository.java
│       │   ├── GreenScoreService.java
│       │   ├── GreenScoreController.java
│       │   └── GridCarbonIntensityResolver.java
│       └── audit/
│           ├── AuditChainEntry.java
│           ├── AuditChainRepository.java
│           ├── AuditChainService.java
│           ├── AuditChainController.java
│           └── PayloadCanonicalizer.java
│
├── services/core/src/main/resources/db/migration/
│   ├── V3__green_score.sql                    ← NOUVEAU
│   └── V4__audit_chain.sql                    ← NOUVEAU
│
├── services/core/src/main/java/com/accenture/nexcharge/ocpp/handlers/
│   └── StopTransactionHandler.java            ← MODIFIÉ (cf §3.5)
│
├── services/core/src/test/java/com/accenture/nexcharge/ecotrade/
│   ├── greenscore/
│   │   ├── GridCarbonIntensityResolverTest.java
│   │   ├── GreenScoreServiceTest.java
│   │   └── GreenScoreControllerIT.java
│   └── audit/
│       ├── PayloadCanonicalizerTest.java
│       ├── AuditChainServiceTest.java
│       ├── AuditChainCorruptionIT.java
│       └── AuditChainConcurrencyIT.java
│
├── apps/ai/src/sustain/                       ← NOUVEAU package Python
│   ├── __init__.py
│   ├── router.py
│   ├── projection.py
│   └── tests/test_projection.py
│
└── apps/web/src/
    ├── app/sustain/
    │   ├── page.tsx                           ← /sustain
    │   ├── leaderboard/page.tsx
    │   ├── projection/page.tsx
    │   └── audit/page.tsx
    └── components/sustain/
        ├── GreenScoreBadge.tsx
        ├── Leaderboard.tsx
        ├── CarbonProjectionChart.tsx
        ├── AuditChainTimeline.tsx
        ├── AuditVerifierButton.tsx
        └── TreeEquivalent.tsx
```

---

## 9. Coordination avec NEXCharge (à valider en call ~15 min)

### 9.1 Décisions à acter

| # | Sujet | Proposition par défaut |
|---|---|---|
| 1 | Branche / repo | Branche `feat/ecotrade-*` sur le repo NEXCharge, PRs reviewed par tech lead |
| 2 | Numérotation Flyway | EcoTrade prend V3 (green_score) et V4 (audit_chain). NEXCharge évite ces numéros sur PRs concurrentes ; coordination via Slack/canal hackathon. |
| 3 | Naming package | `com.accenture.nexcharge.ecotrade` (clair, isolé). Alternatives : `sustain`, `compliance`. |
| 4 | Remplacement `co2_kg_avoided` constant | OUI, on remplace. Le tech lead valide sur PR Module 1. |
| 5 | RBAC `/api/ecotrade/audit/*` | `ADMIN` + `SUSTAINABILITY_OFFICER` (pas accessible aux drivers). |
| 6 | Seed | EcoTrade ajoute son seed dans le runner existant Sprint 1, pas de runner séparé. |

### 9.2 Risques d'intégration

| Risque | Mitigation |
|---|---|
| PR EcoTrade collisionne avec PR Sprint 2 sur `StopTransactionHandler` | Attendre que Sprint 2 soit mergé sur `main` avant de partir sur Module 1. Code-review croisée. |
| Numéro Flyway pris par sprint en parallèle | Canal Slack pour réserver les numéros. Forward-only, pas de squash. |
| `BusinessProperties` cassée par ajout de la sous-section `grid` | Tests `BusinessPropertiesTest` (ajout / migration de config). |
| `AiClient` Python : breaking change de schema entre Module 3 et le reste | Versioning du contrat REST + tests d'intégration côté Java. |

---

## 10. Priorité d'implémentation

| Ordre | Module | Effort | Impact démo | Sprint NEXCharge cible |
|---|---|---|---|---|
| 1 | **Green Score** (Module 1) | ~1 jour (Java + 1 migration + tests) | **Très fort** : transforme un nombre constant en récit "votre charge à 23h évite 30% de plus" | Démarrer dès que Sprint 2 est mergé |
| 2 | **Audit Hash Chain** (Module 2) | ~1 jour (Java + 1 migration + tests + endpoint verify) | **Fort** : narratif compliance "prouvable" pour le pitch | Sprint 3, en parallèle des reminders |
| 3 | **Sustainability UI** (Module 4) | ~1-2 jours (Next.js + composants + Recharts) | **Très fort visuel** : leaderboard + badge + timeline blockchain-style | Sprint 4, avec le polish design |
| 4 | **Carbon Projection** (Module 3) | ~0.5-1 jour (Python endpoint + Java client + chart) | **Moyen** : alimente la narration ESG, sympa mais pas indispensable | Sprint 4 si temps, sinon drop |

Le **Green Score (1) + Audit Chain (2) + UI minimale (3)** suffisent à différencier. La projection (4) est nice-to-have.

---

## 11. Pitch narratif (revu, honnête)

> "NEXCharge enforce les règles de booking, capture la consommation OCPP, et prédit la demande avec son IA. Mais on est allés plus loin sur deux dimensions critiques pour la conformité.
>
> **D'abord, le CO2 traçable.** Le mix énergétique de Maurice varie fortement selon l'heure — un kWh chargé à 23h évite près de 30% plus de CO2 qu'un kWh chargé à 14h. NEXCharge calculait un facteur constant ; on l'a remplacé par un calcul horaire qui reflète la réalité du grid. Chaque session génère un green score précis, chaque employé voit son impact réel.
>
> **Ensuite, l'intégrité prouvable.** L'`audit_log` enregistre les actions ; on a ajouté une chaîne de hashes SHA-256 sur les sessions terminées. Toute modification d'un record passé casse la chaîne et est détectable en `O(n)`. Les données qu'on soumet au régulateur ne sont pas juste des lignes en base — c'est une chaîne vérifiable de bout en bout, inspirée de notre travail blockchain au GTIC Season 7 sur EcoTrade Hub.
>
> **Et au-dessus**, un Sustainability Hub : leaderboard anonymisé pour encourager la charge en heures creuses, projection carbone alignée sur les cibles 2026, audit verifier en un clic.
>
> NEXCharge gère le présent. EcoTrade Hub rend ce présent traçable et le projette dans le futur."

---

## 12. Hors scope (volontaire)

| Sujet | Raison |
|---|---|
| Vraie blockchain (smart contracts, consensus) | YAGNI pour le hackathon ; le hash chain SHA-256 prouve l'intégrité, sans la complexité réseau. Mentionnable comme évolution future. |
| ML pour prédire l'intensité grid (au lieu de lookup table statique) | Lookup table suffit ; le data réel CEB Maurice n'est pas accessible en open API. |
| Tokens / récompenses crypto pour le leaderboard | Hors scope — si demandé en démo, "phase 2 envisagée". |
| Export PDF de l'audit chain | Réutilise l'infra PDF existante de l'ESG report (Apache PDFBox / OpenPDF déjà en place). À ajouter Sprint 4 si temps. |
| Module v1 "Predictive Dashboard" | Supprimé — recouvrement avec le service AI Prophet/LightGBM existant. |

---

## Annexe A — Mapping v1 → v2

| v1 (TS/Prisma) | v2 (NEXCharge-aligned) |
|---|---|
| `lib/ecotrade/greenScore.ts` | `services/core/.../ecotrade/greenscore/GreenScoreService.java` |
| `lib/ecotrade/auditTrail.ts` | `services/core/.../ecotrade/audit/AuditChainService.java` |
| `lib/ecotrade/forecast.ts` | **Supprimé** (Prophet existe). Carbon projection → `apps/ai/src/sustain/projection.py` |
| `prisma/seed-ecotrade.ts` | Étendu dans le runner Java existant Sprint 1 |
| `app/api/ecotrade/green-score/route.ts` | `GreenScoreController.java` (Spring) |
| `components/ecotrade/GreenScoreBadge.tsx` | `apps/web/src/components/sustain/GreenScoreBadge.tsx` (inchangé sur le principe) |
| Schémas Prisma | Migrations Flyway V3, V4 |
