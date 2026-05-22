# NEXCharge Sprint 2 — Core métier (OCPP + Bookings + Live Map) — Design Spec

**Hackathon** : NEXLevel Reinvented — Energizing the Future (Accenture Mauritius)
**Date** : 2026-05-22
**Statut** : Design validé — prêt pour implémentation
**Sprint cible** : Sprint 2 (4–6 personnes)
**Pré-requis** : Sprint 1 mergé sur `main` (monorepo, infra docker-compose, schema Postgres V1, OIDC/JIT, entités JPA, AI service skeleton, Next.js 15 + Auth.js, CI, Playwright stub).

---

## 1. Goal & démo cible

Livrer le **core métier** : un employé peut **réserver** un slot équitable et explicable, le **simulateur OCPP** déclenche une session sur la borne réservée, la **Live Map** affiche en temps réel le passage `Available → Charging → Available`, et un **Facility Manager** peut **override** un booking en cours avec audit log.

**Démo (~3 min)** :
1. `make up` → 8 services + `ocpp-simulator`. 6 bornes auto-provisionnées en `AVAILABLE`.
2. Login `driver/driver` → `/map` → cartes NEX Tower & NEXTERACOM avec 6 bornes vertes.
3. `/bookings/new` → recommend top-3 → fairness breakdown visible → confirm.
4. `make demo-session` → live update : `SIM-NEX-001` passe `OCCUPIED`, drawer montre session.
5. ~2 min plus tard, retour `AVAILABLE`, `/bookings` montre booking `COMPLETED` + kWh total.
6. Login admin → `/admin/bookings` → override → audit_log écrit avec reason.

---

## 2. Architecture — vue d'ensemble

```
┌─────────────────────────────────────────────────────────────────────┐
│  PWA Web (Next.js 15)                                               │
│  + /map (Live Map STOMP)                                            │
│  + /bookings, /bookings/new (CRUD + fairness)                       │
│  + /admin/bookings (override, FACILITY_MANAGER)                     │
└──────────────────┬───────────────────┬──────────────────────────────┘
                   │ REST              │ WebSocket STOMP /ws (JWT)
┌──────────────────▼───────────────────▼──────────────────────────────┐
│  Core Backend (Spring Boot 3, Java 21)                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐   │
│  │ booking/     │  │ chargers/    │  │ ocpp/                    │   │
│  │ - REST CRUD  │  │ - REST query │  │ - OcppServer (WebSocket) │   │
│  │ - fairness   │  │ - status     │  │ - 6 message handlers     │   │
│  │ - validation │  │ - cache view │  │ - Java-OCA-OCPP 1.6-J    │   │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ websocket/  STOMP broker (relay via Redis pub/sub)           │   │
│  │           Topics: /topic/chargers, /topic/charger/{id}       │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ audit/  AuditLogEntry + AuditService (append-only)           │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────┬──────────────────────────────────┘
                  │ OCPP-J 1.6 ws  │ Redis pub/sub
┌─────────────────▼────┐      ┌────▼─────────────────┐
│  ocpp-simulator      │      │  Redis 7             │
│  (services/ocpp-sim) │      │  channel: chargers   │
│  - boot N bornes     │      └──────────────────────┘
│  - REST /sim         │
│    /start-session    │
└──────────────────────┘
```

### Containers nouveaux

| Service | Stack | Rôle |
|---|---|---|
| `ocpp-simulator` | Java 21 + Spring Boot | Client OCPP-J 1.6 multi-borne ; endpoint REST `POST /sim/start-session` |

### Endpoints nouveaux côté `core`

| Méthode | Path | Auth |
|---|---|---|
| WebSocket | `/ocpp/{ocppId}` | aucune (S2 ; sprint 4 = TLS + idTag whitelist) |
| WebSocket | `/ws` (STOMP) | JWT handshake |
| GET | `/api/chargers` (list, `?site=`) | DRIVER+ |
| GET | `/api/chargers/{id}` | DRIVER+ |
| GET | `/api/bookings` (mine ou `?userId=` pour FM/ADMIN) | DRIVER+ |
| GET | `/api/bookings/{id}` | owner OR FM/ADMIN |
| POST | `/api/bookings` | DRIVER |
| DELETE | `/api/bookings/{id}` | owner OR FM |
| POST | `/api/bookings/recommend` | DRIVER |
| POST | `/api/bookings/{id}/override` | FACILITY_MANAGER OR ADMIN |

### Choix structurants

- **OCPP 1.6-J only** (Java-OCA-OCPP). 2.0.1 reporté sprint suivant.
- **Auto-provisioning Charger via `BootNotification`** ; pas de migration de seed (plus dynamique, moins fragile).
- **Redis pub/sub** entre OCPP handlers et STOMP broadcaster, même en single-instance — mis en place dès maintenant pour ne pas refaire l'architecture sprint 3.
- **Fairness rule-based déterministe** ; pas d'IA. La formule du spec global §4 Module 1 est implémentée intégralement, avec `demand_pressure` approximé par `bookings_count_in_slot / chargers_at_site` (proxy ; sprint 3 substitue le forecast IA).
- **Pas de table `usage_quota` matérialisée** en S2 ; calcul inline. YAGNI ; perf sprint 4 si nécessaire.

---

## 3. Modèle de données — deltas Sprint 1

Sprint 1 a livré les 4 entités JPA principales (`User`, `Charger`, `Booking`, `ChargingSession`, `MeterValue`) + ENUMs (`user_role`, `charger_status`, `booking_status`, `session_status`). Sprint 2 **n'introduit aucune nouvelle entité métier** ; il câble la logique sur celles existantes.

### 3.1 Migration Flyway V2 — `audit_log` (append-only)

```sql
-- V2__audit_log.sql
CREATE TABLE audit_log (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  actor_user_id UUID REFERENCES users(id),         -- nullable (system actor)
  action        VARCHAR(64) NOT NULL,              -- e.g. 'BOOKING_OVERRIDE_RELEASED'
  target_type   VARCHAR(32) NOT NULL,              -- e.g. 'BOOKING', 'CHARGER'
  target_id     VARCHAR(64) NOT NULL,
  reason        TEXT,
  metadata      JSONB,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_log_target ON audit_log(target_type, target_id, created_at DESC);
CREATE INDEX idx_audit_log_actor  ON audit_log(actor_user_id, created_at DESC);
-- En prod : REVOKE UPDATE, DELETE ON audit_log FROM <app_role>;
-- En dev local : convention (le code n'expose ni UPDATE ni DELETE).
```

JPA :
- `audit/AuditLogEntry.java` (entity, sans setter `id`/`createdAt`, pas de méthode `update()`).
- `audit/AuditLogRepository.java`.
- `audit/AuditService.java` — méthode `log(actorUserId, action, targetType, targetId, reason, metadata)`.

### 3.2 Booking lifecycle (state machine)

```
RESERVED ──user.cancel──────────────► CANCELLED
RESERVED ──fm.override──────────────► RELEASED_AUTO
RESERVED ──ocpp.StartTransaction───► ACTIVE
ACTIVE   ──ocpp.StopTransaction────► COMPLETED
ACTIVE   ──fm.override──────────────► RELEASED_AUTO
ACTIVE   ──user.cancel──────────────► CANCELLED   (rare, audit log)
```

Toute transition non listée → `409 Conflict`, code `INVALID_TRANSITION`.

### 3.3 Champs activés en S2

| Champ | Sprint 1 | Sprint 2 |
|---|---|---|
| `Booking.fairness_score` | nullable, non rempli | rempli au `POST /bookings` (snapshot) |
| `Booking.predicted_demand` | nullable | rempli (proxy : `bookingsInSlot / chargersAtSite`) |
| `Charger.last_heartbeat` | non utilisé | mis à jour à chaque `Heartbeat` OCPP |
| `Charger.status` | défaut `AVAILABLE` | mis à jour à chaque `StatusNotification` |
| `ChargingSession.id_tag` | (à vérifier dans entité S1) | rempli depuis OCPP `StartTransaction.idTag`. Si absent dans entité S1 → V2 ajoute la colonne. |
| `ChargingSession.kwh_total`, `peak_power_kw`, `co2_kg_avoided` | nullable | calculés à `StopTransaction` |
| `ChargingSession.status` | défaut `IN_PROGRESS` | transitionne vers `COMPLETED` ou `INTERRUPTED` |

### 3.4 V2 : pas de modification destructive

V2 = additions uniquement (`audit_log` table + index ; éventuellement `ALTER TABLE charging_sessions ADD COLUMN id_tag VARCHAR(64)` si manquant Sprint 1). Aucun changement de type/contrainte sur les tables existantes. Migration forward-only safe.

---

## 4. Couche OCPP

### 4.1 Lib & versions

```
eu.chargetime.ocpp:v1.6:1.2.x      (vérifier disponibilité Maven Central au début du sprint)
```

Si la lib est absente de Maven Central / GitHub releases, fallback : fork local dans `services/core/libs/` (jar checked-in) **OR** parsing manuel JSON OCPP-J (frame `[2, msgId, action, payload]`). Décision figée à la première task.

### 4.2 Endpoint serveur

```
ws://core:8090/ocpp/{ocppId}     subprotocol: ocpp1.6
```

Implémenté via `@Bean WebSocketHandler` Spring qui délègue au `JSONServer` de Java-OCA-OCPP. Le `{ocppId}` est extrait du path et utilisé comme identité de la borne. **Pas d'auth OCPP en S2** (TLS + idTag whitelist = sprint 4).

### 4.3 Handlers (`ocpp/handlers/`)

| Action OCPP 1.6 | Handler | Effet métier |
|---|---|---|
| `BootNotification` | `BootNotificationHandler` | Auto-provisioning : `INSERT INTO chargers ON CONFLICT (ocpp_id) DO UPDATE SET vendor=..., model=..., firmware_version=..., last_heartbeat=now()`. Réponse `Accepted`, `interval=300`. Site mappé depuis prefix de l'ocpp_id (`SIM-NEX-*` → `NEX_TOWER`, `SIM-NXR-*` → `NEXTERACOM`). |
| `Heartbeat` | `HeartbeatHandler` | Update `chargers.last_heartbeat=now()`. Réponse `currentTime`. |
| `StatusNotification` | `StatusNotificationHandler` | Map OCPP status → `charger_status` enum (cf §4.4). Update `Charger.status`. **Publie sur Redis channel `chargers.status`**. |
| `StartTransaction` | `StartTransactionHandler` | Trouve `Booking RESERVED` du user pour cette borne dans la fenêtre `[slotStart - 15min, slotEnd]` ; si trouvé : transition → `ACTIVE`. Crée `ChargingSession(IN_PROGRESS, started_at=now, charger, user, id_tag, booking_id)`. Walk-in autorisé (sans booking → session sans `booking_id`). Réponse `transactionId` (long auto-incr, mappé sur `ChargingSession.id` ou colonne dédiée). |
| `MeterValues` | `MeterValuesHandler` | Pour chaque sample du payload : `INSERT INTO meter_values (session_id, recorded_at, kwh, power_kw, voltage, current, soc_percent)`. Pas de broadcast STOMP en S2. |
| `StopTransaction` | `StopTransactionHandler` | Calcule `kwh_total = meterStop - meterStart`, `peak_power_kw = MAX(meter_values.power_kw)`, `co2_kg_avoided = kwh_total × CO2_FACTOR_KG_PER_KWH` (var `BusinessProperties.co2FactorKgPerKwh`, défaut `0.4` pour Mauritius grid). Ferme `ChargingSession(COMPLETED, ended_at=now)`. Si `booking_id` non null : transition `Booking ACTIVE → COMPLETED`. |

### 4.4 Mapping OCPP status → enum

| OCPP `ChargePointStatus` | `charger_status` |
|---|---|
| `Available` | `AVAILABLE` |
| `Preparing`, `Charging`, `SuspendedEV`, `SuspendedEVSE`, `Finishing` | `OCCUPIED` |
| `Reserved` | `RESERVED` |
| `Unavailable` | `OFFLINE` |
| `Faulted` | `FAULTED` |

### 4.5 Idempotence & resilience

- `BootNotification` : `INSERT ... ON CONFLICT (ocpp_id) DO UPDATE`. Idempotent.
- `StartTransaction` : si `transactionId` déjà connu pour ce charger, retourner le même.
- `Heartbeat watchdog` : `@Scheduled(fixedRate=60_000)` met `Charger.status = OFFLINE` (et publie Redis) pour tout charger dont `last_heartbeat < now() - 15min`.

### 4.6 Tests OCPP (`OcppIntegrationIT`)

- `JSONClient` test (côté test, lib Java-OCA-OCPP côté client) connecté au `core` lancé par `@SpringBootTest(webEnvironment=RANDOM_PORT)` + Testcontainers Postgres + Redis.
- Cycle vérifié : `BootNotification` → ligne `chargers` créée ; `StatusNotification(Charging)` → message Redis publié + `Charger.status='OCCUPIED'` ; `StartTransaction → MeterValues × 3 → StopTransaction` → `ChargingSession` créée puis fermée avec `kwh_total > 0`, `MeterValue` rows insérées.

---

## 5. Simulateur OCPP

### 5.1 Localisation & structure

Sous-projet Gradle Spring Boot dans `services/ocpp-simulator/` (frère de `services/core/`).

```
services/ocpp-simulator/
├── build.gradle.kts
├── Dockerfile               (eclipse-temurin:21-jre-alpine)
├── settings.gradle.kts
└── src/main/java/com/accenture/nexcharge/ocppsim/
    ├── OcppSimulatorApplication.java
    ├── ChargerClient.java              (1 instance = 1 borne, wraps eu.chargetime.ocpp.JSONClient)
    ├── ChargerFleet.java               (gère N ChargerClient au boot)
    ├── SessionScenario.java            (Start → MeterValues×N → Stop scripté)
    ├── api/
    │   └── SimController.java          (REST endpoints on-demand)
    └── config/
        └── SimulatorProperties.java    (@ConfigurationProperties("nexcharge.sim"))
└── src/main/resources/application.yml
```

### 5.2 Boot sequence

À `ApplicationReadyEvent` :
1. Lit `SIM_CHARGER_COUNT=6` + `SIM_OCPP_URL=ws://core:8090/ocpp` + `SIM_SITES`.
2. Pour chaque borne (3 par site, prefix `SIM-NEX-001..003` / `SIM-NXR-001..003`) :
   - Ouvre `JSONClient` WebSocket vers `${SIM_OCPP_URL}/${ocppId}`.
   - Envoie `BootNotification(vendor=NexCharge, model=Sim, firmwareVersion=1.0)`.
   - Démarre `@Scheduled` heartbeat toutes les 60s.
   - Envoie `StatusNotification(Available)`.
3. Si la connexion drop, retry exponentiel (1s, 2s, 4s, ..., max 30s).

### 5.3 Endpoint REST on-demand

```
POST /sim/start-session
Content-Type: application/json
{
  "chargerId": "SIM-NEX-001",
  "idTag": "DRIVER-DEMO",
  "durationSeconds": 120,
  "meterValueIntervalSeconds": 10,
  "endKwh": 12.5,
  "peakPowerKw": 11.0
}
→ 202 Accepted
{ "transactionId": 42, "endsAt": "2026-05-22T14:35:00Z" }
```

Logique :
- `t=0` : `StatusNotification(Charging)` + `StartTransaction(idTag, meterStart=0)`.
- `t=10s, 20s, ...` : `MeterValues` interpolés linéairement entre 0 et `endKwh`, `power_kw` oscille autour de `peakPowerKw` (±10%).
- `t=duration` : `StopTransaction(meterStop=endKwh)` + `StatusNotification(Available)`.

Endpoints complémentaires :
```
GET  /sim/status                  → liste bornes connectées + sessions actives
POST /sim/stop-session/{txId}     → force StopTransaction
```

### 5.4 docker-compose

```yaml
ocpp-simulator:
  build:
    context: ../services/ocpp-simulator
  depends_on:
    core:
      condition: service_healthy
  environment:
    SIM_OCPP_URL: ws://core:8090/ocpp
    SIM_CHARGER_COUNT: 6
  ports:
    - "9100:9100"
  restart: unless-stopped
```

Optionnellement routé via Traefik sur `http://localhost/sim/*` pour `make demo-session`.

### 5.5 Makefile target

```makefile
demo-session:
	curl -X POST http://localhost:9100/sim/start-session \
	  -H 'Content-Type: application/json' \
	  -d '{"chargerId":"SIM-NEX-001","idTag":"DRIVER-DEMO","durationSeconds":120,"meterValueIntervalSeconds":10,"endKwh":12.5,"peakPowerKw":11.0}'
```

---

## 6. Bookings — controllers, validation, fairness

### 6.1 Endpoints détaillés

| Méthode | Path | Auth | Comportement |
|---|---|---|---|
| `GET` | `/api/bookings` | DRIVER (mine) / FM+ADMIN (`?userId=`) | Pagination basique `?from=&to=&status=`, défaut 30 derniers jours |
| `GET` | `/api/bookings/{id}` | owner OR FM+ADMIN | Détail booking + charger + session si existe |
| `POST` | `/api/bookings` | DRIVER | Body `{chargerId, slotStart, slotEnd}` ; validations §6.2 ; calcule `fairness_score` + `predicted_demand` (snapshot) ; status initial `RESERVED` |
| `DELETE` | `/api/bookings/{id}` | owner OR FM | Transition vers `CANCELLED`. Si `ACTIVE` → audit log `BOOKING_CANCELLED_DURING_SESSION` |
| `POST` | `/api/bookings/recommend` | DRIVER | Body `{site, durationMinutes, earliestStart, latestStart}` ; génère candidats (chargers du site × créneaux 15-min granularité) ; filtre disponibilité ; score + retourne top-3 avec `breakdown` |
| `POST` | `/api/bookings/{id}/override` | FM+ADMIN | Body `{reason}` (non vide) ; transition forcée `→ RELEASED_AUTO` ; audit log obligatoire |

### 6.2 Validations `BookingValidator`

| Règle | Erreur (HTTP, code) |
|---|---|
| Durée > `MAX_BOOKING_DURATION_HOURS` (3) | 400, `DURATION_EXCEEDED` |
| Chevauchement avec autre booking actif sur le même charger | 409, `BOOKING_OVERLAP` |
| Chevauchement avec mes propres bookings actifs | 409, `OWN_BOOKING_OVERLAP` |
| Mes 1 actif + 1 futur déjà atteints | 409, `MAX_CONCURRENT_REACHED` |
| Cooldown 48h actif (3 no-shows ce mois) | 403, `COOLDOWN_ACTIVE` (avec `until`) |
| Bookings cette semaine ≥ `MAX_BOOKINGS_PER_WEEK` (5) | 429, `WEEKLY_CAP` (avec `current`, `max`) |
| Charger `OFFLINE` ou `FAULTED` | 409, `CHARGER_UNAVAILABLE` |
| `slotStart` dans le passé | 400, `SLOT_IN_PAST` |
| `slotStart >= slotEnd` | 400, `INVALID_SLOT_RANGE` |

### 6.3 Concurrency

`POST /api/bookings` : verrou pessimiste `SELECT * FROM bookings WHERE charger_id=? AND status IN ('RESERVED','ACTIVE') AND tstzrange(slot_start,slot_end) && tstzrange(?,?) FOR UPDATE`. Évite la double-réservation côte-à-côte sous load. Transaction `REPEATABLE_READ`.

### 6.4 Fairness scoring (rule-based)

Implémentation `FairnessScorer.score(user, candidate) → ScoreResult`.

```
quotaRemaining = clamp01( 1 - kwhUsedThisMonth(user) / fairShareKwh(user) )
timeSinceLast  = clamp01( daysSince(user.lastBookingEnd) / 7 )
demandPressure = clamp01( bookingsCountInSlot(candidate) / chargersAtSite(candidate.site) )
weeklyRatio    = clamp01( bookingsThisWeek(user) / MAX_BOOKINGS_PER_WEEK )

raw = w1*quotaRemaining + w2*timeSinceLast - w3*demandPressure - w4*weeklyRatio
score = (raw + 1) / 2     # remap [-1, +1] → [0, 1]

breakdown = {
  quotaRemaining, timeSinceLast, demandPressure, weeklyRatio,
  weights: { quota: w1, recency: w2, demand: w3, weekly: w4 }
}
```

**Poids dans `BusinessProperties`** :
```yaml
nexcharge:
  business:
    fairness:
      weights:
        quota: 0.40
        recency: 0.30
        demand: 0.15
        weekly: 0.15
```
(somme = 1.0 ; tunable admin via env vars).

**`fair_share_kwh`** : pour S2, calcul inline simple :
```
fairShareKwh(user) = (totalSiteCapacityKwhPerMonth × 0.8) / activeUsersThisMonth
```
ou `User.fair_share_kwh` si fixé manuellement (sprint 1 avait la colonne nullable). `totalSiteCapacityKwhPerMonth` = constante `BusinessProperties` (défaut `5000`). Pas de table `usage_quota` matérialisée en S2.

### 6.5 Tests `FairnessScorerTest`

Scénarios couverts :
- User neuf (jamais bookéé) → `quotaRemaining=1`, `timeSinceLast=1`, score haut.
- User qui a tout consommé → `quotaRemaining=0`.
- Slot saturé (3 bookings sur 3 chargers) → `demandPressure=1` → pénalise.
- Weekly à 5/5 → `weeklyRatio=1` → pénalise.
- Chaque poids isolé en mettant les 3 autres à 0 → vérifie effet du poids.
- Score toujours dans `[0, 1]`.

### 6.6 ChargingSession lié au booking via OCPP

`StartTransactionHandler` :
```java
Optional<Booking> booking = bookingRepo.findReservedFor(
    chargerId, idTag, now, Duration.ofMinutes(15));
if (booking.isPresent()) {
    booking.get().transitionTo(ACTIVE);
    chargingSession = new ChargingSession(booking.get().id(), charger, booking.get().user(), idTag, now);
} else {
    // Walk-in autorisé en S2 : session sans booking_id
    chargingSession = new ChargingSession(null, charger, null, idTag, now);
}
```

`StopTransactionHandler` ferme la session, transitionne `Booking ACTIVE → COMPLETED` si `booking_id` non null, calcule métriques.

---

## 7. Live Map — STOMP + Redis pub/sub

### 7.1 Backend `websocket/`

Composants :
- `WebSocketConfig` — `@EnableWebSocketMessageBroker`, `/ws` endpoint, JWT handshake interceptor, broker relay (in-memory + Redis pub/sub bridge en S2).
- `JwtHandshakeInterceptor` — valide le JWT (Authorization header ou `?access_token=`), extrait `User`, stocke dans `Principal` du `StompSession`.
- `ChargerStatusBroadcaster` — Spring `@Component` listener Redis qui consomme `chargers.status` et fait `simpMessagingTemplate.convertAndSend("/topic/chargers", event)` + `/topic/charger/{id}`.
- `RedisPubSubConfig` — `ChannelTopic("chargers.status")`, `MessageListenerAdapter`.

Flow :
```
StatusNotificationHandler              (côté OCPP)
   └─► ChargerService.updateStatus(...)
         ├─► UPDATE chargers SET status=...      (PG)
         └─► redisTemplate.convertAndSend(
                "chargers.status",
                ChargerStatusEvent(chargerId, status, sessionId?, kwh?, eta?))

ChargerStatusBroadcaster.onMessage     (subscribe Redis)
   └─► simpMessagingTemplate.convertAndSend("/topic/chargers", event)
   └─► simpMessagingTemplate.convertAndSend("/topic/charger/" + chargerId, event)
```

### 7.2 Topics

| Topic | Payload | Use case |
|---|---|---|
| `/topic/chargers` | `{chargerId, ocppId, status, site}` | Heatmap globale |
| `/topic/charger/{id}` | `{chargerId, status, currentSession?: {startedAt, kwh, etaEnd}}` | Drawer drill-down |

### 7.3 Initial sync

À `@SubscribeMapping("/topic/chargers")` : retourne snapshot `chargerService.allCurrentStatuses()` au new subscriber. Évite l'écran vide au mount.

### 7.4 Frontend `apps/web`

```
src/app/map/page.tsx                     server component shell (auth gate)
src/components/live-map/
   ├── LiveMap.tsx                       'use client' ; 2 cartes
   ├── ChargerBadge.tsx                  icône status + animation flash on update
   ├── ChargerDrawer.tsx                 detail drawer (subscribe /topic/charger/{id})
   └── useStompClient.ts                 hook custom @stomp/stompjs
src/lib/stomp.ts                         singleton client config
```

Comportement :
- Au mount : `GET /api/chargers` (snapshot REST) + subscribe `/topic/chargers`.
- Optimistic merge : event STOMP met à jour le state local (immer ou setState).
- Fallback : si `socket.readyState !== OPEN` après 3 reconnect attempts, polling `/api/chargers` toutes les 5s.
- 2 cartes côte-à-côte (NEX_TOWER, NEXTERACOM), chaque borne = badge cliquable.

### 7.5 Pages frontend Sprint 2

| Path | Composant | Rôle |
|---|---|---|
| `/map` | `LiveMap` | Live Map STOMP |
| `/bookings` | `BookingsList` | Liste mes bookings, bouton cancel |
| `/bookings/new` | `BookingForm` | Formulaire site/durée/fenêtre → recommend top-3 → confirm |
| `/admin/bookings` | `AdminBookings` | Middleware role check FM+ ; list all + override modal |
| `/dashboard` (mise à jour) | — | Ajoute "My next booking" + lien `/map` |

UI : Tailwind seul. Pas de shadcn/ui en S2 (sprint 4 polish).

---

## 8. Audit & error handling

### 8.1 Audit log — quand écrire

| Action | `action` | Acteur |
|---|---|---|
| `POST /api/bookings/{id}/override` | `BOOKING_OVERRIDE_RELEASED` | `actor_user_id` = FM/ADMIN |
| `DELETE /api/bookings/{id}` quand status ∈ {`ACTIVE`} | `BOOKING_CANCELLED_DURING_SESSION` | owner |
| `Booking → RELEASED_AUTO` (sprint 3 le déclenchera ; signature en place) | `BOOKING_AUTO_RELEASED` | system (`actor_user_id=null`) |

**Pas d'écriture audit pour cancel normal RESERVED** (volume élevé ; sprint 4 le rajoutera si besoin).

### 8.2 Global exception handler

`@RestControllerAdvice GlobalExceptionHandler` dans `common/web/` :
- `BookingValidationException` → 400/403/409/429 selon le code (cf §6.2).
- `BookingNotFoundException`, `ChargerNotFoundException` → 404.
- `InvalidStateTransitionException` → 409, `INVALID_TRANSITION`.
- `AccessDeniedException` (Spring Security) → 403.
- Exception générique → 500 avec correlation id (logging only, pas de stack trace en réponse).

Format de réponse erreur :
```json
{
  "code": "BOOKING_OVERLAP",
  "message": "Charger SIM-NEX-001 already booked from 14:00 to 16:00",
  "details": { "chargerId": "...", "conflictingBookingId": "..." }
}
```

### 8.3 Resilience

| Scénario | Comportement |
|---|---|
| Heartbeat manquant > 15min | `@Scheduled(60_000)` met `Charger.status = OFFLINE`, publie Redis |
| Redis indisponible | STOMP broker direct (in-memory). Log warn. Pas de fail-fast. |
| Frontend STOMP disconnect | reconnect exponentiel + fallback polling 5s |
| Cancel sur ACTIVE pendant MeterValues en cours | `BookingService.cancel` prend lock pessimiste sur `charging_session` active si exists |
| `OCPP StartTransaction` charger inconnu | Réponse `{idTagInfo: {status: "Invalid"}}` + log warn (auto-provisioning seulement sur Boot) |
| `OCPP StopTransaction` sans transactionId connu | Log + ignore (orphan) |

---

## 9. Tests

### 9.1 Backend `services/core`

| Type | Classe | Couvre |
|---|---|---|
| Unit | `FairnessScorerTest` | Formule pure, edge cases (user neuf / quota épuisé / slot saturé / weekly à 5), poids isolés |
| Unit | `BookingValidatorTest` | Toutes les règles §6.2 |
| Unit | `BookingTransitionTest` | State machine §3.2 |
| Integration | `BookingControllerIT` | CRUD complet via MockMvc, RBAC, conflict 409 sur double-booking, recommend top-3 |
| Integration | `ChargerControllerIT` | List, filter par site, detail |
| Integration | `OverrideIT` | FM peut override, audit_log écrit avec reason |
| Integration | `OcppIntegrationIT` | Cycle complet `JSONClient` test → core (Testcontainers PG+Redis) |
| Integration | `WebSocketIT` | Client STOMP test → subscribe `/topic/chargers` → trigger `StatusNotification` via OCPP test → assert message broadcast (timeout 2s) |

Coverage cible : ≥ 70% sur logique métier (`booking/`, `chargers/`, `ocpp/handlers/`).

### 9.2 Backend `services/ocpp-simulator`

`SimControllerIT` : mock du serveur OCPP avec fake WebSocket, vérifie que `POST /sim/start-session` envoie la séquence `StartTransaction → MeterValues × N → StopTransaction`.

### 9.3 Frontend `apps/web`

Sprint 2 ajoute `tests/bookings-flow.spec.ts` (Playwright) : login → recommend → book → voir dans /bookings → cancel. **Skipped** par défaut comme `auth-flow.spec.ts` Sprint 1 (nécessite stack up). Pas wiré en CI.

### 9.4 CI

`.github/workflows/ci.yml` Sprint 1 reste compatible (le job `core` lance `./gradlew test` qui pickera les nouveaux tests). Ajouter un job `ocpp-simulator` parallèle qui build + teste le sous-projet.

---

## 10. Configuration métier (`BusinessProperties` deltas)

Sprint 1 a la classe ; Sprint 2 ajoute :

```yaml
nexcharge:
  business:
    # déjà Sprint 1
    max-booking-duration-hours: 3
    sufficient-charge-threshold-pct: 80
    auto-release-grace-minutes: 15
    no-show-cooldown-hours: 48
    max-bookings-per-week: 5
    bias-gini-alert-threshold: 0.4
    workplace-ops-email: Workplace.Mauritius.OfficeServices@accenture.com
    # nouveau Sprint 2
    fairness:
      weights:
        quota: 0.40
        recency: 0.30
        demand: 0.15
        weekly: 0.15
    co2-factor-kg-per-kwh: 0.4
    total-site-capacity-kwh-per-month: 5000
    ocpp:
      heartbeat-interval-seconds: 300
      heartbeat-timeout-minutes: 15
```

Validation : somme des poids ≈ 1.0 (tolérance ±0.01) au démarrage.

---

## 11. Découpage en tasks (preview pour writing-plans)

13 tasks, vertical-slice (approche A), chaque task ≈ 1–2 commits, livre une démo intermédiaire :

1. **OCPP server skeleton** — dépendance Java-OCA-OCPP, `OcppServerConfig`, endpoint `/ocpp/{ocppId}`, smoke test connexion.
2. **OCPP handlers BootNotification + Heartbeat + StatusNotification** — auto-provisioning Charger, update status, publish Redis.
3. **Simulateur OCPP service** — `services/ocpp-simulator/`, boot N bornes au démarrage, Dockerfile, docker-compose service.
4. **Migration V2 audit_log** + entity `AuditLogEntry` + `AuditService`.
5. **Live Map backend** — STOMP config + JWT handshake + Redis listener → broadcast + `@SubscribeMapping` snapshot initial.
6. **Live Map frontend** — page `/map`, hook `useStompClient`, 2 cartes, status badges, polling fallback.
7. **OCPP handlers StartTransaction + MeterValues + StopTransaction** — création/update ChargingSession, transition Booking, MeterValue insert.
8. **Simulateur endpoint REST** — `POST /sim/start-session`, scénario scripted, Makefile target `make demo-session`.
9. **Booking validator + fairness scorer** — logique pure, tests unit.
10. **Bookings CRUD endpoints** — controllers + service + global exception handler + tests integration.
11. **Bookings frontend** — `/bookings`, `/bookings/new`, formulaire recommend+book.
12. **Admin override** — endpoint `POST /api/bookings/{id}/override` + page `/admin/bookings` + audit log + tests.
13. **OCPP & STOMP integration tests + README S2 + CI green**.

---

## 12. Risks Sprint 2

| Risque | Mitigation |
|---|---|
| Java-OCA-OCPP API change ou Maven Central pas disponible | Vérifier coordonnées au début Task 1 ; fallback fork local jar si disparu |
| STOMP + Spring Security + JWT handshake = piège classique | Documenter dans `WebSocketConfig`, test d'intégration `WebSocketIT` tôt (Task 5) |
| `ocpp-simulator` healthcheck dépend du `core` qui dépend de PG/Redis — démarrage long | `depends_on: condition: service_healthy` sur tous + retry exponentiel côté simulateur |
| Auto-provisioning sur BootNotification crée n'importe quoi | OK en S2 (sandbox) ; whitelist sprint 4 |
| Race cancel-pendant-StartTransaction | Lock pessimiste sur `bookings` lors de `BookingService.cancel` ET `StartTransactionHandler` |
| OCPP transactionId mapping `long` ↔ JPA `UUID` ChargingSession | Ajouter colonne `ocpp_transaction_id BIGINT` UNIQUE dans `charging_sessions` (V2) ou utiliser un mapping séparé. Décision en Task 7. |

---

## 13. Hors scope Sprint 2 (différé)

| Sujet | Sprint cible |
|---|---|
| OCPP 2.0.1 dual-stack | Sprint 3+ si besoin |
| Auth OCPP (TLS + idTag whitelist) | Sprint 4 |
| Forecast IA `demand_pressure` | Sprint 3 |
| Reminders & auto-release scheduler | Sprint 3 |
| Sufficient-charge alert (SoC ≥ 80%) | Sprint 3 |
| Push notifications (Web Push + VAPID) | Sprint 3 |
| Anomaly detection | Sprint 3 |
| ESG narration LLM | Sprint 4 |
| Module 7 Issue Reporting | Sprint 4 |
| `usage_quota` table matérialisée | Sprint 4 (perf) |
| Dashboards facility manager / sustainability officer | Sprint 4 |
| shadcn/ui design system | Sprint 4 polish |
| Bias audit Gini | Sprint 4 |

---

## Annexe A — Schéma fichiers nouveaux Sprint 2

```
services/core/src/main/java/com/accenture/nexcharge/
├── ocpp/
│   ├── OcppServerConfig.java
│   ├── OcppWebSocketHandler.java
│   ├── handlers/
│   │   ├── BootNotificationHandler.java
│   │   ├── HeartbeatHandler.java
│   │   ├── StatusNotificationHandler.java
│   │   ├── StartTransactionHandler.java
│   │   ├── MeterValuesHandler.java
│   │   └── StopTransactionHandler.java
│   └── HeartbeatWatchdog.java
├── websocket/
│   ├── WebSocketConfig.java
│   ├── JwtHandshakeInterceptor.java
│   ├── ChargerStatusBroadcaster.java
│   ├── ChargerStatusEvent.java
│   └── RedisPubSubConfig.java
├── booking/
│   ├── BookingController.java
│   ├── BookingService.java
│   ├── BookingValidator.java
│   ├── FairnessScorer.java
│   ├── ScoreResult.java
│   ├── BookingDto.java
│   ├── RecommendationDto.java
│   ├── BookingValidationException.java
│   └── BookingTransitionService.java
├── chargers/
│   ├── ChargerController.java
│   ├── ChargerService.java
│   └── ChargerDto.java
├── audit/
│   ├── AuditLogEntry.java
│   ├── AuditLogRepository.java
│   └── AuditService.java
└── common/web/GlobalExceptionHandler.java

services/core/src/main/resources/db/migration/
└── V2__audit_log.sql

services/ocpp-simulator/        (sous-projet Gradle complet, cf §5.1)

apps/web/src/
├── app/
│   ├── map/page.tsx
│   ├── bookings/page.tsx
│   ├── bookings/new/page.tsx
│   ├── admin/bookings/page.tsx
│   └── dashboard/page.tsx       (mis à jour)
├── components/
│   ├── live-map/
│   │   ├── LiveMap.tsx
│   │   ├── ChargerBadge.tsx
│   │   ├── ChargerDrawer.tsx
│   │   └── useStompClient.ts
│   ├── bookings/
│   │   ├── BookingsList.tsx
│   │   ├── BookingForm.tsx
│   │   └── RecommendationCard.tsx
│   └── admin/
│       └── AdminBookings.tsx
└── lib/
    ├── stomp.ts
    └── api-client.ts            (étendu avec bookings/chargers endpoints)

infra/
└── docker-compose.yml           (ajoute service ocpp-simulator)

Makefile                          (ajoute target demo-session)
```
