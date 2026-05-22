# Coordination NEXCharge ↔ EcoTrade Hub

> **Pour l'implémenteur NEXCharge** — ce document liste ce dont la couche EcoTrade Hub a besoin de ton sprint pour pouvoir se brancher sans friction. Lecture : ~10 min. Toutes les actions sont à valider avant que tu pushes Sprint 1.

**Contexte** : EcoTrade Hub est une couche de différenciation construite **dans le même Spring Boot app** que NEXCharge (pas un service séparé). Voir [`EcoTradeHub_Layer_Spec.md`](../EcoTradeHub_Layer_Spec.md) §2 et §9.

---

## 1. Réservations Flyway (CRITIQUE)

EcoTrade Hub réserve les versions suivantes — **ne pas les utiliser pour ton code** :

| Version | Module EcoTrade | Table créée |
|---------|-----------------|-------------|
| `V3__green_score.sql` | Module 1 (Green Score) | `green_score` |
| `V4__audit_chain.sql`  | Module 2 (Audit Hash Chain) | `audit_chain` |

- [ ] Tes migrations Sprint 1 utilisent **V1, V2** uniquement (et V5+ pour la suite)
- [ ] Aucun `Vx__green_score*` ni `Vx__audit_chain*` côté NEXCharge

**Si tu as besoin d'une 3e migration en Sprint 1**, prends V5 et préviens-moi.

---

## 2. Schéma `charging_sessions` — champs attendus par EcoTrade

Module 1 lit ces colonnes après qu'une session passe `COMPLETED` :

```sql
-- Colonnes nécessaires sur charging_sessions :
id              UUID PRIMARY KEY
status          VARCHAR  -- doit pouvoir valoir 'COMPLETED'
started_at      TIMESTAMPTZ NOT NULL
ended_at        TIMESTAMPTZ NOT NULL  -- NULL avant complétion
kwh_delivered   NUMERIC(10,3) NOT NULL
```

- [ ] Ces 5 colonnes existent sur `charging_sessions` après V1/V2
- [ ] `started_at` et `ended_at` sont en `TIMESTAMPTZ` (pas `TIMESTAMP` naïf) — Module 1 fait du slicing horaire en `Indian/Mauritius`

### Sur `co2_kg_avoided` (constante 0.4)

Ta spec NEXCharge prévoit `co2_kg_avoided = kwh * 0.4` stocké sur `charging_sessions`. **EcoTrade Hub remplace ce calcul** par une intensité grid variable horaire (550–850 g/kWh).

**Décision recommandée** :
- [ ] **Retirer** la colonne `co2_kg_avoided` de `charging_sessions` (V1)
- [ ] **OU** la laisser nullable et **ne pas la populer** côté NEXCharge

EcoTrade écrit la vraie valeur dans `green_score.co2_avoided_g` (BIGINT, en grammes), avec FK vers `charging_sessions(id)`. Tes dashboards FM/SO peuvent join sur cette table.

---

## 3. Hook après StopTransaction → événement Spring

EcoTrade a besoin d'être notifié **dans la même transaction** que la complétion de session, pour :
- calculer le Green Score (Module 1)
- ajouter un maillon dans la chaîne d'audit (Module 2)

**Approche recommandée — Spring Application Event** (couplage zéro) :

```java
// Dans ton StopTransactionHandler (NEXCharge), après session.setStatus(COMPLETED) :
applicationEventPublisher.publishEvent(new SessionCompletedEvent(session.getId()));
```

```java
// Événement à créer dans un package partagé (ex: com.accenture.nexcharge.events)
public record SessionCompletedEvent(UUID sessionId) {}
```

Côté EcoTrade, j'écoute avec :
```java
@EventListener
@Transactional(propagation = Propagation.REQUIRED)
public void onSessionCompleted(SessionCompletedEvent event) {
    greenScoreService.computeAndStore(event.sessionId());
    auditChainService.append(event.sessionId());
}
```

**Action** :
- [ ] Créer la classe `SessionCompletedEvent` dans `com.accenture.nexcharge.events`
- [ ] L'émettre **après** `session.setStatus(COMPLETED)` et **avant** la fin de la transaction `StopTransaction`
- [ ] Vérifier qu'`ApplicationEventPublisher` est bien injecté (Spring l'injecte par défaut)

> Alternative si tu préfères : injection directe de `GreenScoreService` + `AuditChainService` dans ton handler. Moins propre (couplage), mais OK pour le hackathon. Préviens-moi si tu choisis cette voie.

---

## 4. Package naming

EcoTrade Hub vit dans le sous-package suivant **du même app Spring Boot** :

```
com.accenture.nexcharge.ecotrade
├── greenscore/      (Module 1)
├── auditchain/      (Module 2)
└── config/          (EcotradeProperties, etc.)
```

- [ ] Ton root package reste `com.accenture.nexcharge` — `@SpringBootApplication` au-dessus suffit pour scanner mes beans
- [ ] Pas besoin de `@ComponentScan` explicite

---

## 5. Spring Security — rôles requis

Module 2 expose des endpoints REST `/api/audit-chain/**` protégés par RBAC :

```java
@PreAuthorize("hasAnyRole('ADMIN', 'SUSTAINABILITY_OFFICER')")
```

- [ ] Le rôle `ROLE_ADMIN` existe dans ta config Spring Security
- [ ] Le rôle `ROLE_SUSTAINABILITY_OFFICER` existe (ta spec NEXCharge le mentionne dans les personas — confirmer qu'il est mappé en `GrantedAuthority`)
- [ ] `@EnableMethodSecurity` est activé sur ta config

Si l'un manque, ajoute-le à ta config Sprint 1 (1-2 lignes).

---

## 6. Dépendances Maven/Gradle attendues

EcoTrade ajoute ces dépendances dans `pom.xml` (ou `build.gradle`) — préviens-moi si tu en as déjà certaines :

```xml
<!-- Déjà dans NEXCharge probablement -->
<dependency>spring-boot-starter-data-jpa</dependency>
<dependency>spring-boot-starter-web</dependency>
<dependency>spring-boot-starter-security</dependency>
<dependency>org.flywaydb:flyway-core</dependency>
<dependency>org.postgresql:postgresql</dependency>

<!-- À ajouter pour Module 2 (canonicalisation JSON) -->
<dependency>com.fasterxml.jackson.core:jackson-databind</dependency>
<!-- (Spring Boot l'inclut déjà via spring-boot-starter-web) -->

<!-- Tests -->
<dependency>org.testcontainers:postgresql:1.19.x</dependency>
<dependency>org.testcontainers:junit-jupiter:1.19.x</dependency>
```

- [ ] Confirmer Spring Boot **3.x** + **JDK 21** (cf. ta spec)
- [ ] Confirmer image Testcontainers Postgres **16** (matche prod)

---

## 7. Roadmap de synchro

```
[Toi] Sprint 1 NEXCharge          [Moi] EcoTrade Hub
─────────────────────────         ──────────────────────────
V1, V2 Flyway                     ⏸️  bloqué (lit ton schema)
ChargingSession entity            
StopTransactionHandler + event   →  débloqué : exécute Module 1 plan
                                     (V3, GreenScoreService, …)
                                  →  exécute Module 2 plan
                                     (V4, AuditChainService, …)
PR Sprint 1 mergé sur main        →  PR ecotrade/module-1
                                  →  PR ecotrade/module-2
```

- [ ] Pousser Sprint 1 sur une branche `nexcharge/sprint-1` puis PR vers `main`
- [ ] M'avertir dès que la PR est mergée — je démarre Module 1 dans la foulée
- [ ] Mes PRs cibleront `main` aussi, branches `ecotrade/module-1-green-score` et `ecotrade/module-2-audit-chain`

---

## 8. Plans détaillés à consulter

Si tu veux voir exactement ce que j'ajoute :

- [`docs/superpowers/plans/2026-05-22-module-1-green-score.md`](superpowers/plans/2026-05-22-module-1-green-score.md) — 8 tâches TDD (V3 migration, resolver intensity grid horaire Maurice, GreenScoreService, patch StopTransactionHandler via event listener)
- [`docs/superpowers/plans/2026-05-22-module-2-audit-hash-chain.md`](superpowers/plans/2026-05-22-module-2-audit-hash-chain.md) — 10 tâches TDD (V4 migration, PayloadCanonicalizer, HashCalculator SHA-256, AuditChainService idempotent + advisory lock, ITs corruption + concurrence, REST verify endpoints)

Aucun de ces plans ne touche tes fichiers — ils ajoutent des fichiers dans `com.accenture.nexcharge.ecotrade.*` et écoutent ton `SessionCompletedEvent`.

---

## 9. Récap des actions de ton côté (avant push Sprint 1)

- [ ] Ne pas utiliser V3, V4 Flyway
- [ ] Schéma `charging_sessions` avec les 5 colonnes listées §2 (TIMESTAMPTZ)
- [ ] Décider : retirer `co2_kg_avoided` ou le laisser nullable non-populé
- [ ] Créer `SessionCompletedEvent(UUID sessionId)` dans `com.accenture.nexcharge.events`
- [ ] L'émettre dans `StopTransactionHandler` après `setStatus(COMPLETED)`
- [ ] Confirmer rôles `ROLE_ADMIN` + `ROLE_SUSTAINABILITY_OFFICER` dans Spring Security
- [ ] `@EnableMethodSecurity` activé
- [ ] Spring Boot 3.x + JDK 21 confirmé

**Une fois ces 8 cases cochées et ton Sprint 1 mergé sur `main`, je démarre Module 1.**

---

## 10. Contact

Bloqué sur un point ? Désaccord sur une décision (ex: structure du `co2_kg_avoided`, choix event vs injection directe) ? Ouvre une issue sur ce repo ou ping-moi avant le push Sprint 1 — c'est plus simple de réaligner maintenant qu'après le merge.
