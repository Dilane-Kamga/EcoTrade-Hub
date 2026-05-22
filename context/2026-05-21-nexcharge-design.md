# NEXCharge — Design Spec

**Hackathon** : NEXLevel Reinvented — Energizing the Future (Accenture Mauritius)
**Date** : 2026-05-21
**Statut** : Design validé — prêt pour implémentation
**Équipe** : 4–6 personnes, durée > 1 mois

---

## 1. Vision produit

### Baseline existante (à remplacer)

Aujourd'hui, les chargeurs EV au NEX Tower et Nexteracom sont régis par des **guidelines manuelles** :
- Limite de session 3h max ou jusqu'à charge "suffisante" (subjectif, non-enforced).
- Rotation : déplacer le véhicule "promptly" une fois chargé (règle d'honneur).
- Coordination entre collègues par message/calendar note.
- Issues remontées par email à `Workplace.Mauritius.OfficeServices@accenture.com`.

NEXCharge **enforce** ces règles automatiquement et les rend **équitables, mesurables et explicables**.

### Pitch
Une plateforme PWA mobile-first qui rend la recharge EV au NEX Tower & NEXTERACOM **équitable, transparente et prédictive**, en s'appuyant sur OCPP en temps réel et une couche IA responsable qui explique chacune de ses décisions.

### Les 3 promesses différenciantes
1. **Équité visible** — chaque employé voit son "fair share" mensuel ; le système empêche les abus avec un algorithme transparent (pas de FIFO injuste).
2. **Predict-then-book** — l'IA prédit l'occupation des prochaines 24h ; l'utilisateur voit en direct quand sa borne est probablement disponible (avec niveau de confiance).
3. **Sustainability narrée** — le rapport ESG mensuel n'est pas qu'un dashboard : un agent IA rédige un récit explicatif (pourquoi la conso a baissé, quels patterns émergent, quelles actions prendre).

### Personas
- **Driver** — employé qui recharge ; usage mobile principal.
- **Facility Manager** — gestion bornes & politique ; usage desktop.
- **Sustainability Officer** — lecture des rapports ESG ; usage desktop.
- **Admin** — config, RBAC, audit ; usage desktop.

### Mapping brief → modules

| Brief | Module | Différenciateur |
|---|---|---|
| Slot Booking | #1 + fairness scoring | Algo d'équité explicable, durée 3h enforced |
| Real-Time Availability | #2 | Heatmap prédite vs réelle |
| OCPP Consumption Capture | #3 | Vraies bornes + simulateur |
| Smart Reminders & Releases | #4 | Auto-release loggé, **SoC-based "sufficient charge" alert** |
| Reporting & Sustainability | #5 + ESG narration | Récit narré par Claude |
| Responsible AI Layer | #6 (forecasting, anomaly, fairness, narration) | Explainability sur chaque décision |
| (Bonus) Issue Reporting | #7 | Remplace l'email manuel Workplace par 1 clic + photo |

---

## 2. Architecture technique

### Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                     │
│  PWA Mobile (driver)        Web Desktop (facility/sustain./admin)   │
└──────────────────┬──────────────────────────────────────────────────┘
                   │ HTTPS + WebSocket (STOMP)
┌──────────────────▼──────────────────────────────────────────────────┐
│              Frontend (Next.js 15 - App Router)                     │
│   - Server Components + Auth.js (OIDC → Entra ID)                   │
│   - Client TS généré depuis OpenAPI du backend Java                 │
│   - WebSocket STOMP client                                          │
└──────┬─────────────────────────────────────┬────────────────────────┘
       │ REST JSON (typé OpenAPI)            │ REST JSON
       │                                     │
┌──────▼──────────────────────┐    ┌─────────▼────────────────┐
│  Core Backend (Java 21)     │    │   AI Service (Python)    │
│  Spring Boot 3.x            │◄───┤   FastAPI                │
│  - Spring Web (REST)        │REST│   - Forecasting          │
│  - Spring Security (OIDC)   │    │     (Prophet/LightGBM)   │
│  - Spring Data JPA          │    │   - Anomaly detection    │
│  - Spring WebSocket (STOMP) │    │   - Fairness scoring     │
│  - Spring Data Redis        │    │   - LLM narration        │
│  - springdoc-openapi        │    │     (Claude API)         │
│  - Java-OCA-OCPP            │    │   - Explainability (SHAP)│
│  - Spring @Scheduled        │    │                          │
└──┬──────────────┬───────────┘    └─────────┬────────────────┘
   │              │                          │
┌──▼──┐ ┌─────────▼──┐ ┌──────────┐ ┌────────▼────────┐ ┌─────────┐
│ PG  │ │ Redis      │ │  MinIO   │ │ Feature Store   │ │ OCPP    │
│ 16  │ │ pub/sub    │ │ S3-compat│ │ (PG views/dbt)  │ │ bornes  │
│     │ │ + cache    │ │          │ │                 │ │ (réseau)│
└─────┘ └────────────┘ └──────────┘ └─────────────────┘ └─────────┘
```

### Containers Docker (docker-compose unique)

| Service | Stack | Rôle |
|---|---|---|
| `web` | Node 20 + Next.js 15 (PWA) | UI mobile/desktop + auth OIDC |
| `core` | Java 21 + Spring Boot 3.x | API REST métier + WebSocket STOMP + serveur OCPP |
| `ai` | Python 3.12 + FastAPI | Forecasting, anomaly, fairness, narration LLM |
| `postgres` | Postgres 16 | Données métier + feature views |
| `redis` | Redis 7 | Pub/sub WebSocket, cache, queue |
| `minio` | MinIO | Storage S3-compatible (rapports PDF, exports) |
| `traefik` | Traefik 3 | Reverse proxy + TLS termination |
| `keycloak` (dev only) | Keycloak | OIDC provider en local (remplacé par Entra en démo) |

### Choix structurants

- **Backend Java unifié** : un seul service Spring Boot porte la logique métier ET le serveur OCPP. Simplifie le hackathon ; extractible plus tard.
- **OpenAPI codegen** : le backend Java expose son OpenAPI ; le front génère son client TypeScript automatiquement (`openapi-typescript-codegen` ou `openapi-fetch`).
- **Real-time** : Spring WebSocket + STOMP (broker Redis pour scaler). Le front utilise `@stomp/stompjs`.
- **Service AI en Python** : reste séparé. Java l'appelle en REST. Frontière naturelle (écosystème ML = Python).
- **Auth** : Spring Security `oauth2-resource-server` valide les JWT Entra ID. Le frontend Next.js fait le flow OIDC avec Auth.js.
- **Pas de cloud lock-in** : tous services self-hostables (Postgres, Redis, MinIO, Keycloak). OIDC = protocole standard, on peut basculer Entra ↔ Keycloak sans changer le code.
- **Build Java** : Gradle (DX rapide) sur Java 21 LTS.

### Structure du module Java

```
com.accenture.nexcharge
├── booking/        BookingController, BookingService, repos, entities
├── chargers/       ChargerController, ocpp/, status/
├── sessions/       SessionService, énergie consommée
├── users/          UserController, RBAC, fairness quotas
├── reporting/      ReportingController, ESG aggregations
├── ocpp/           OcppServer, MessageHandler, BootNotification, MeterValues...
├── notifications/  Reminders, scheduled jobs
├── ai/             AiClient (HTTP client vers le service Python)
├── security/       SecurityConfig, OIDC
└── common/         config, websocket, redis
```

### Communications inter-services

| De | Vers | Protocole |
|---|---|---|
| `web` | `core` | REST JSON (typé OpenAPI) |
| `core` | `ai` | REST JSON |
| Bornes OCPP | `core` | WebSocket OCPP-J (1.6/2.0.1) |
| `core` | `web` | WebSocket STOMP via broker Redis |

Le front ne parle jamais directement à `ai` : tous les appels IA passent par le backend Java, qui orchestre, persiste les `ai_explanation`, et applique les garde-fous.

---

## 3. Modèle de données (Postgres)

### Entités principales

```
User (id, entra_oid, email, display_name, role, fair_share_kwh, created_at)
  ├─1:N─► Booking
  └─1:N─► UsageQuota (period YYYYMM, kwh_used, sessions_count)

Booking (id, user_id, charger_id, slot_start, slot_end, status,
         predicted_demand, fairness_score, released_reason, created_at)
  └─1:1─► ChargingSession (when active)

Charger (id, ocpp_id, site, location_label, status, max_power_kw,
         connector_type, last_heartbeat, created_at)
  ├─1:N─► ChargerStatusLog (status, event_payload, recorded_at)
  └─1:N─► ChargingSession

ChargingSession (id, booking_id, charger_id, user_id, started_at, ended_at,
                 kwh_total, peak_power_kw, co2_kg_avoided, status)
  └─1:N─► MeterValue (recorded_at, kwh, power_kw, voltage, current, soc_percent)
```

### Tables annexes

| Table | Contenu |
|---|---|
| `notification` | Push/email envoyés (reminder, release alert, anomaly) — audit |
| `ai_prediction` | Forecasts générés (charger_id, horizon, value, confidence, model_version) |
| `ai_explanation` | Texte d'explication par décision IA (inputs, output, raison) |
| `audit_log` | Toute action sensible (RBAC change, booking forcé, override) — append-only |
| `report_export` | Métadonnées des rapports ESG (mois, format, path MinIO, narration LLM) |
| `feature_view_*` | Vues SQL/dbt pour le ML (utilization horaire, demand par jour) |

### Enums

- `User.role` : `DRIVER`, `FACILITY_MANAGER`, `SUSTAINABILITY_OFFICER`, `ADMIN`
- `Booking.status` : `RESERVED`, `ACTIVE`, `COMPLETED`, `CANCELLED`, `RELEASED_AUTO`, `NO_SHOW`
- `Charger.status` : `AVAILABLE`, `OCCUPIED`, `RESERVED`, `OFFLINE`, `FAULTED`
- `ChargingSession.status` : `IN_PROGRESS`, `COMPLETED`, `INTERRUPTED`

### Décisions clés

- **`MeterValue`** est la table chaude (1 valeur / 30s par session active). Index sur `(session_id, recorded_at)` ; partitionnement mensuel si volumétrie le justifie. Postgres natif suffit pour le hackathon.
- **`fair_share_kwh`** dynamique (calculé) ; `UsageQuota` mensuel matérialisé pour la perf.
- **`predicted_demand` + `fairness_score`** sur Booking = snapshot au moment de la réservation, pour expliquer après coup.
- **`ai_prediction` + `ai_explanation`** séparées des entités métier — la couche AI ne pollue pas le modèle métier.
- **`audit_log` immuable** (append-only, pas d'UPDATE/DELETE) dès le départ — base de la "Responsible AI".
- **Migrations** : Flyway, scripts SQL versionnés (`V1__init.sql`, `V2__...`).

---

## 4. Modules fonctionnels

### Module 1 — Slot Booking (réservation équitable)

**UX driver** : sélection site + durée + fenêtre préférée → 3 créneaux ranked par équité × disponibilité prédite. Chaque créneau a un badge "Why this slot?" qui ouvre l'explication IA.

**Algorithme d'équité** :
```
score = w1 × (1 - kwh_used / fair_share_kwh)         # quota restant
      + w2 × (time_since_last_booking / 7j)          # délai depuis dernière charge
      - w3 × demand_pressure                         # désincite les heures de pointe
      - w4 × bookings_this_week / max_bookings_week  # anti-monopolisation
```
- `bookings_this_week` = nombre de bookings (statut ≠ `CANCELLED`) du user dans la semaine ISO en cours.
- `max_bookings_week` = plafond cible par user et par semaine (config admin, défaut 5).
- Tous les termes sont normalisés dans `[0,1]` ; le score final est dans `[-1, +1]` puis re-mappé `[0,1]` pour l'affichage.

Poids `w1..w4` configurables par admin, exposés via `/explain/booking/{id}`, affichés dans l'UI.

**Backend** : `booking/`. `BookingService.findBestSlots(userId, criteria)` appelle `AiClient.predictDemand(...)` puis applique le scoring.

**Garde-fous** :
- Pas de double-booking (verrou pessimiste).
- Max 1 actif + 1 futur par user.
- No-show → release auto à T+15 min.
- Cooldown 48h après 3 no-shows dans le mois.
- **Durée maximale d'un slot = 3h par défaut** (`MAX_BOOKING_DURATION_HOURS`, configurable par admin) — aligné avec les guidelines actuelles Accenture.

### Module 2 — Real-Time Availability (live map)

**UX** : 2 cartes (NEX Tower, NEXTERACOM) avec icônes colorées par status, mises à jour en temps réel via WebSocket STOMP. Click sur borne → drawer avec status, session courante, kW, ETA fin.

**Backend** : `chargers/` + `ChargerStatusController`. Topics STOMP : `/topic/chargers` (broadcast global) et `/topic/charger/{id}` (drilldown). Source de vérité = OCPP server → Redis pub/sub → `ChargerStatusListener` → broadcast STOMP.

**Front** : composant `LiveMap` qui s'abonne au topic, optimistic UI, fallback polling 5s si WebSocket indisponible.

### Module 3 — OCPP Consumption Capture

**Implémentation** : serveur OCPP intégré dans le backend Java (lib `Java-OCA-OCPP`, supporte 1.6 et 2.0.1). Bornes se connectent en WebSocket → backend, identifiées par `ocpp_id`.

**Messages gérés** :
- `BootNotification` → enregistre/met à jour le charger
- `Heartbeat` → met à jour `last_heartbeat` + status
- `StatusNotification` → change `Charger.status` + log
- `StartTransaction` → crée `ChargingSession`
- `MeterValues` → insère `MeterValue` (toutes les 30s)
- `StopTransaction` → ferme session + calcule `kwh_total`, `co2_kg_avoided`

**Mode dégradé** : un simulateur OCPP (script Java standalone) accompagne le projet pour développer/démontrer si une borne réelle est offline.

### Module 4 — Smart Reminders & Auto-Release

- **Pre-session reminder** : 15 min avant le slot → notification push + email.
- **Sufficient-charge alert** : déclenché dès `MeterValue.soc_percent >= 80%` (seuil `SUFFICIENT_CHARGE_THRESHOLD_PCT` configurable) → notification "Your vehicle reached sufficient charge — please make space for colleagues". Objective la règle floue "sufficient charge" des guidelines Accenture.
- **End-of-session alert** : T-10 min de la fin du slot → notification ("Almost done, please move shortly").
- **Auto-release** : borne reste `AVAILABLE` 15 min après le début du slot → `RELEASED_AUTO`, créneau libéré, alerte au user.
- **Grace period** configurable par facility manager.

**Backend** : `notifications/`. Spring `@Scheduled` toutes les minutes. Push web (Web Push API + VAPID), email (SMTP standard, Mailpit en dev). File de jobs Redis pour découpler envoi. Chaque release loggée dans `audit_log`.

### Module 5 — Reporting & Sustainability

**Live KPIs** (dashboard facility manager) : bornes actives/total, sessions en cours, kWh aujourd'hui, CO2 évité.

**Trends** (sustainability officer) : heatmap utilization 24h × 7j par borne ; courbes mensuelles (énergie, sessions, fairness index Gini) ; comparaison NEX Tower vs NEXTERACOM.

**ESG Report mensuel** : bouton "Generate" → backend agrège KPIs → appelle `/ai/narrate/esg-report` → narration Claude + recommandations → PDF (Apache PDFBox ou OpenPDF) avec graphiques + narration → MinIO + métadonnées dans `report_export`. Téléchargement / partage email.

### Module 6 — Responsible AI Layer (service Python)

**a) Demand Forecasting** (Prophet/LightGBM) — `POST /ai/forecast/demand`. Entrée : charger_id ou site, horizon 24h/7j. Sortie : timeseries probabilité d'occupation + intervalle de confiance. Affichée dans booking et heatmap predicted vs réelle.

**b) Anomaly Detection** — `POST /ai/anomaly/detect`. Analyse `MeterValue` récents : sessions trop longues, conso suspecte, sous-performance. Badge "anomaly" avec explication dans dashboard facility manager.

**c) Fairness Scoring** — `POST /ai/fairness/score`. Entrée : user_id + créneau visé. Sortie : score 0–1 + breakdown (`{quota_remaining, time_since_last, demand_pressure}`). Utilisé par `BookingService` ; breakdown affiché dans "Why this slot?".

**d) ESG Narration** — `POST /ai/narrate/esg-report`. Prompt structuré avec KPIs → narration Claude + recommandations. Garde-fous : prompt template versionné ; nombres fournis dans le prompt (jamais inventés par le LLM) ; vérification post-génération.

**Explainability transversale** : chaque décision IA écrit dans `ai_explanation` avec `model_version`, `inputs` (snapshot features), `output_value` + `confidence`, `explanation_text` (SHAP pour fairness/anomaly ; prompt+réponse pour ESG).

**Stack Python** : FastAPI, scikit-learn, Prophet, LightGBM, SHAP, Anthropic SDK. Modèles entraînés sur dataset synthétique au début, ré-entraînés sur données réelles dès qu'on en a. MLflow optionnel (si temps).

### Module 7 — Report Issue (frottement → 1 clic)

Aujourd'hui, signaler un problème sur une borne implique d'envoyer un email à `Workplace.Mauritius.OfficeServices@accenture.com`. NEXCharge intègre cette friction directement :

**UX** : sur la page d'une borne (et dans Live Map drawer), bouton "Report issue" → modal avec catégorie (`HARDWARE`, `CABLE`, `CARD_READER`, `OTHER`), description libre, photo optionnelle (PWA camera API).

**Backend** : nouvelle route `POST /api/chargers/{id}/issues`. Crée une `notification` de type `ISSUE_REPORT` (channel `EMAIL`, destinataire = config admin `WORKPLACE_OPS_EMAIL`, défaut `Workplace.Mauritius.OfficeServices@accenture.com`). L'email contient borne, user reporter, catégorie, description, photo (lien MinIO), heatbeat status récent.

**Audit** : chaque issue loggée dans `audit_log` (`action='ISSUE_REPORTED'`).

**Bonus AI (sprint 4)** : auto-catégorisation par LLM basée sur description + corrélation avec anomalies récentes du même charger.

---

## 5. Sécurité, RBAC, équité, Responsible AI

### Authentification (OIDC standard, portable)

1. User → "Sign in" → Next.js redirige vers Entra ID.
2. Retour ID token + access token JWT.
3. Auth.js stocke session ; chaque appel REST envoie le JWT en header `Authorization: Bearer ...`.
4. Backend Java valide le JWT (Spring Security `oauth2-resource-server`, JWKS Entra).
5. Première connexion → backend crée le `User` (`User.id` ↔ Entra `oid`), rôle par défaut `DRIVER`.

**Dev local** : Keycloak (image Docker, même protocole OIDC). L'app ne sait pas la différence.

### RBAC

| Rôle | Permissions |
|---|---|
| `DRIVER` | Créer/voir/annuler ses bookings, voir Live Map, voir ses propres stats |
| `FACILITY_MANAGER` | + Configurer bornes, override bookings, voir tous les users, déclencher rapports, ajuster grace period |
| `SUSTAINABILITY_OFFICER` | + Accès rapports ESG (lecture seule). Pas de droit users/bookings. |
| `ADMIN` | + Gérer rôles, accéder audit logs, configurer modèles AI |

**Implémentation** : Spring Security `@PreAuthorize("hasRole('FACILITY_MANAGER')")` au niveau controller. + Row-level security côté service (un `DRIVER` ne peut voir que ses bookings, vérifié dans `BookingService`).

### Équité

`fair_share_kwh` mensuel par user, calculé dynamiquement :
```
fair_share_kwh(user, month) = total_capacity_kwh(month) × 0.8 / active_users(month)
```
Marge 20% pour gérer les pics. Active_users = users avec ≥ 1 booking dans le mois.

Anti-abus : 1 booking actif + 1 futur max ; cooldown 48h après 3 no-shows / mois (override admin possible).

### Responsible AI — 5 garde-fous

1. **Explainability obligatoire** : chaque décision automatisée → `ai_explanation` avec inputs, outputs, raison lisible.
2. **Pas de boîte noire critique** : fairness score = formule transparente (pas un modèle ML opaque). Les modèles ML (forecast, anomaly) sont des suggestions ; les décisions critiques restent règle-based.
3. **Bias audit mensuel** : Gini coefficient sur kWh alloués → alerte si > 0.4. Visible dans dashboard sustainability.
4. **LLM grounding** : prompt template strict ; LLM commente, ne calcule pas. Vérification post-génération que tous les nombres viennent du prompt.
5. **Human override** : facility manager peut overrider toute décision IA ; override loggé avec raison textuelle dans `audit_log`.

### Sécurité technique

- HTTPS partout (Traefik en reverse proxy).
- Secrets via env vars + `.env` (template `.env.example` commité, vrai `.env` non commité).
- Rate limiting (Bucket4j).
- CSRF : Auth.js + SameSite cookies.
- Headers (CSP, HSTS, X-Frame-Options) via middleware Next.js.
- Pas de PII dans les logs (filter Logback custom).
- Audit log append-only.

### Configuration métier (admin-tunable)

| Clé | Défaut | Source / Justification |
|---|---|---|
| `MAX_BOOKING_DURATION_HOURS` | `3` | Guidelines Accenture actuelles |
| `SUFFICIENT_CHARGE_THRESHOLD_PCT` | `80` | Seuil pragmatique pour "sufficient charge" |
| `AUTO_RELEASE_GRACE_MINUTES` | `15` | Tolérance no-show |
| `NO_SHOW_COOLDOWN_HOURS` | `48` | Anti-abus (après 3 no-shows / mois) |
| `MAX_BOOKINGS_PER_WEEK` | `5` | Plafond fairness |
| `WORKPLACE_OPS_EMAIL` | `Workplace.Mauritius.OfficeServices@accenture.com` | Destinataire des `ISSUE_REPORT` |
| `BIAS_GINI_ALERT_THRESHOLD` | `0.4` | Seuil d'alerte audit fairness |

### GDPR

- `GET /me/export` → ZIP avec données personnelles (JSON + CSV).
- `DELETE /me` → soft delete (anonymisation : email/nom remplacés ; agrégats conservés).
- Rétention : `MeterValue` brutes purgées après 12 mois (configurable) ; agrégats conservés indéfiniment.

---

## 6. Tests, dev workflow, planning

### Tests

**Backend Java** :
- Unit : JUnit 5 + Mockito (fairness, scoring, validation booking).
- Integration : Spring Boot Test + **Testcontainers** (Postgres + Redis réels en container, pas de mock DB).
- OCPP : test d'intégration avec faux client OCPP (BootNotification → MeterValues → StopTransaction → vérification session).
- Coverage cible : 70% sur logique métier.

**AI service Python** : pytest sur fairness, anomaly, datasets synthétiques. Snapshot tests sur structure prompt narration. Validation modèles via MAE/MAPE sur dataset de validation.

**Frontend** : Vitest + Testing Library sur composants critiques. **Playwright** sur 3 parcours golden :
1. Driver : login → book → live map → cancel.
2. Facility manager : voir anomalie → override → logger raison.
3. Sustainability officer : générer rapport ESG → télécharger PDF.

**Pas de tests** : pas de mocks DB en intégration (Testcontainers à la place) ; pas de tests visuels regression.

### Structure repo (monorepo)

```
nexcharge/
├── apps/
│   ├── web/                  # Next.js 15 PWA
│   └── ai/                   # FastAPI Python
├── services/
│   └── core/                 # Spring Boot Java
├── packages/
│   ├── api-client/           # TS client généré depuis OpenAPI Java
│   └── shared-types/         # Types partagés front/back
├── infra/
│   ├── docker-compose.yml    # Stack complète locale
│   ├── docker-compose.dev.yml
│   ├── traefik/              # Config reverse proxy
│   └── ocpp-simulator/       # Script de simulation
├── docs/
│   └── superpowers/specs/    # Specs et plans
└── README.md
```

**Tooling** : `pnpm` workspaces (front), `gradle` multi-module (back), `uv` (Python). 1 seul repo. Branches `main` protégé + PR. CI GitHub Actions (lint + tests + build images Docker). Pre-commit hooks : `husky` + `lint-staged` (front), `pre-commit` (Python), `spotless` (Java). Quickstart : `make up`.

### Planning hackathon (1 mois+, 4–6 personnes)

**Sprint 1 — Fondations** : monorepo + docker-compose + CI ; auth OIDC end-to-end (Keycloak local, Entra démo) ; schéma Postgres + Flyway ; "Hello world" front/back/AI qui se parlent ; modèle de données complet. *Démo S1 : login fonctionnel.*

**Sprint 2 — Core métier** : CRUD bookings + algo fairness (sans IA) ; OCPP server intégré + simulateur ; sessions enregistrées via OCPP ; Live Map WebSocket STOMP. *Démo S2 : book un slot, simulateur fait une session, vue live.*

**Sprint 3 — IA & UX** : service AI (forecasting Prophet, fairness, anomaly) ; "Why this slot?" + explainability ; PWA polish (installable, push notifications) ; reminders & auto-release ; smoke tests vraies bornes. *Démo S3 : flow complet driver, IA visible.*

**Sprint 4 — Reporting & polish** : dashboards FM + SO ; rapport ESG mensuel narré → PDF ; bias audit Gini ; tests E2E Playwright ; polish design + responsive. *Démo S4 : produit complet.*

**Buffer (semaines 5+)** : intégration vraies bornes, performance, sécurité, préparation pitch.

### Répartition équipe (5 personnes)

| Personne | Focus | Modules |
|---|---|---|
| Tech lead / fullstack | Archi, intégration, démo | OCPP, glue, démo |
| Backend Java | Core métier, sécurité | Booking, sessions, RBAC |
| AI / Data | Service Python, modèles | Forecasting, fairness, narration |
| Frontend | UX, PWA, design system | Web, mobile, dashboards |
| Frontend / QA | Reporting, tests E2E | Reports, Playwright, polish |

(Si 6 personnes : dédoubler "Backend Java" — un sur OCPP, un sur métier.)

### Risques & mitigations

| Risque | Mitigation |
|---|---|
| Vraies bornes inaccessibles | Simulateur OCPP en parallèle dès S1 |
| API Claude rate-limitée en démo | Cache des narrations, mode dégradé sans LLM |
| Forecast IA peu précis (peu de données réelles) | Dataset synthétique réaliste pour entraîner |
| WebSocket instable (mauvais wifi démo) | Fallback polling 5s |
| Complexité OCPP > prévu | OCPP 1.6 d'abord (plus simple), 2.0.1 si temps |

### Démo finale (script suggéré, ~6 min)

- 30s — pitch avec contraste **avant / après** :
  > "Today at NEX Tower : 3-hour limit unenforced, 'sufficient charge' subjective, coordination by email and calendar notes, issues reported by email. Tomorrow with NEXCharge : booking equitable et expliqué, libération auto, SoC-based 'sufficient charge' alert, issues reportées en 1 clic — et toutes les décisions IA tracées."
- 2 min — parcours driver mobile (book avec explainability, live map, report issue).
- 1 min — facility manager (anomaly detected, override loggé).
- 1 min — sustainability officer (générer rapport ESG, narration Claude lue à l'écran, Gini index).
- 30s — slide architecture & responsible AI principles.
- 1 min — Q&A.

---

## Annexes

### Stack résumée

| Couche | Tech |
|---|---|
| Frontend | Next.js 15 (App Router) + React 19 + Tailwind + shadcn/ui + PWA |
| Backend | Java 21 + Spring Boot 3.x + Gradle |
| AI | Python 3.12 + FastAPI + Prophet + LightGBM + SHAP + Anthropic SDK |
| DB | Postgres 16 + Flyway migrations |
| Cache / pub-sub | Redis 7 |
| Storage | MinIO (S3-compatible) |
| Auth | OIDC (Entra ID en démo, Keycloak en dev) |
| Real-time | Spring WebSocket + STOMP + Redis broker |
| OCPP | `Java-OCA-OCPP` (1.6 + 2.0.1) |
| Reverse proxy | Traefik 3 |
| Containerisation | Docker + docker-compose |
| CI | GitHub Actions |
| Tests | JUnit 5 + Testcontainers (back) ; pytest (AI) ; Vitest + Playwright (front) |
