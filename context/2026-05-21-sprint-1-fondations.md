# NEXCharge Sprint 1 — Fondations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Mettre en place les fondations du monorepo NEXCharge — structure de projet, services Docker (`web`, `core`, `ai`, `postgres`, `redis`, `minio`, `keycloak`, `traefik`), authentification OIDC end-to-end, schéma Postgres complet via Flyway, et preuve que les 3 services applicatifs (`web` Next.js, `core` Spring Boot Java, `ai` FastAPI Python) communiquent entre eux. À la fin du sprint : un user peut se connecter via Keycloak, l'UI affiche son nom, et un endpoint de healthcheck du backend appelle l'AI service avec succès.

**Architecture:** Monorepo (pnpm workspaces + Gradle multi-module + uv pour Python). Stack containerisée Docker portable. Backend Java unifié (logique métier + serveur OCPP plus tard). Service AI séparé en Python. Auth OIDC standard (Keycloak en dev, Entra ID en démo). Communications : `web` → `core` en REST JSON (typé via OpenAPI codegen), `core` → `ai` en REST JSON.

**Tech Stack:** Next.js 15 (App Router) + React 19 + Tailwind + shadcn/ui + Auth.js ; Java 21 + Spring Boot 3.x + Gradle + Spring Security + Spring Data JPA + Flyway + springdoc-openapi ; Python 3.12 + FastAPI + uv + pytest ; Postgres 16 + Redis 7 + MinIO + Keycloak + Traefik 3 ; Docker + docker-compose.

**Référence spec:** `docs/superpowers/specs/2026-05-21-nexcharge-design.md`

---

## File Structure

À la fin du sprint, le repo aura la structure suivante :

```
nexcharge/
├── apps/
│   ├── web/                        # Next.js 15 PWA
│   │   ├── package.json
│   │   ├── next.config.mjs
│   │   ├── tailwind.config.ts
│   │   ├── tsconfig.json
│   │   ├── Dockerfile
│   │   ├── .env.example
│   │   ├── public/
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── layout.tsx
│   │   │   │   ├── page.tsx                # Landing
│   │   │   │   ├── (auth)/signin/page.tsx
│   │   │   │   ├── dashboard/page.tsx      # Page protégée "hello name"
│   │   │   │   └── api/auth/[...nextauth]/route.ts
│   │   │   ├── lib/
│   │   │   │   ├── auth.ts                 # Config Auth.js (OIDC generic)
│   │   │   │   └── api-client.ts           # Wrapper fetch avec JWT
│   │   │   ├── components/
│   │   │   │   └── sign-in-button.tsx
│   │   │   └── middleware.ts               # Redirect non-auth vers /signin
│   │   └── tests/
│   │       └── auth-flow.spec.ts           # Playwright (smoke)
│   │
│   └── ai/                         # FastAPI Python
│       ├── pyproject.toml
│       ├── Dockerfile
│       ├── .env.example
│       ├── src/
│       │   └── nexcharge_ai/
│       │       ├── __init__.py
│       │       ├── main.py                 # FastAPI app + /healthz + /version
│       │       ├── config.py
│       │       └── narrate/__init__.py     # placeholder pour S3
│       └── tests/
│           └── test_health.py
│
├── services/
│   └── core/                       # Spring Boot Java
│       ├── build.gradle.kts
│       ├── settings.gradle.kts
│       ├── gradle.properties
│       ├── Dockerfile
│       ├── .env.example
│       ├── src/main/java/com/accenture/nexcharge/
│       │   ├── NexchargeApplication.java
│       │   ├── common/
│       │   │   ├── config/AppProperties.java
│       │   │   └── web/CorsConfig.java
│       │   ├── security/
│       │   │   ├── SecurityConfig.java
│       │   │   └── CurrentUser.java
│       │   ├── users/
│       │   │   ├── User.java               # JPA entity
│       │   │   ├── UserRole.java           # enum
│       │   │   ├── UserRepository.java
│       │   │   ├── UserService.java
│       │   │   └── UserController.java     # GET /users/me
│       │   ├── chargers/
│       │   │   ├── Charger.java
│       │   │   ├── ChargerStatus.java
│       │   │   └── ChargerRepository.java
│       │   ├── booking/
│       │   │   ├── Booking.java
│       │   │   ├── BookingStatus.java
│       │   │   └── BookingRepository.java
│       │   ├── sessions/
│       │   │   ├── ChargingSession.java
│       │   │   ├── SessionStatus.java
│       │   │   ├── MeterValue.java
│       │   │   └── SessionRepository.java
│       │   ├── ai/
│       │   │   └── AiClient.java           # WebClient vers FastAPI
│       │   └── health/
│       │       └── HealthController.java   # GET /healthz, /healthz/ai
│       ├── src/main/resources/
│       │   ├── application.yml
│       │   ├── application-dev.yml
│       │   └── db/migration/
│       │       └── V1__init.sql
│       └── src/test/java/com/accenture/nexcharge/
│           ├── users/UserControllerIT.java
│           └── health/HealthControllerIT.java
│
├── packages/
│   ├── api-client/                 # TS client généré depuis OpenAPI Java
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── openapi.json            # Téléchargé depuis core en build
│   │   └── src/                    # Généré par openapi-typescript
│   └── shared-types/               # Types TS partagés (placeholder)
│       ├── package.json
│       └── src/index.ts
│
├── infra/
│   ├── docker-compose.yml          # Stack complète prod-like
│   ├── docker-compose.dev.yml      # Override hot reload
│   ├── .env.example
│   ├── traefik/
│   │   └── traefik.yml
│   ├── keycloak/
│   │   └── realm-export.json       # Realm "nexcharge" pré-configuré
│   ├── postgres/
│   │   └── init.sql                # CREATE DATABASE nexcharge
│   └── minio/
│       └── README.md               # Buckets à créer manuellement
│
├── docs/
│   └── superpowers/
│       ├── specs/
│       │   └── 2026-05-21-nexcharge-design.md
│       └── plans/
│           └── 2026-05-21-sprint-1-fondations.md   # ce fichier
│
├── .github/
│   └── workflows/
│       └── ci.yml                  # lint + tests + build images
│
├── .gitignore
├── .editorconfig
├── package.json                    # Root pnpm workspace
├── pnpm-workspace.yaml
├── Makefile                        # `make up`, `make down`, `make logs`
├── README.md
└── CONTRIBUTING.md
```

**Responsabilités par fichier clé** :
- `apps/web/src/lib/auth.ts` : config Auth.js (NextAuth) avec provider OIDC générique. Une seule responsabilité : auth flow.
- `services/core/.../security/SecurityConfig.java` : config Spring Security `oauth2-resource-server`. Valide les JWT.
- `services/core/.../users/UserService.java` : provisionning JIT du `User` Postgres au premier login (mapping Entra `oid` → `User.id`).
- `services/core/.../ai/AiClient.java` : WebClient typé vers le service Python. Frontière nette entre Java et Python.
- `services/core/.../db/migration/V1__init.sql` : schéma complet du sprint (toutes les tables principales du modèle de données).
- `apps/ai/src/nexcharge_ai/main.py` : FastAPI app minimale avec `/healthz` et `/version` — preuve que le service tourne.
- `infra/docker-compose.yml` : orchestre les 8 services, réseau dédié, volumes persistants.

---

## Tasks

### Task 1: Bootstrap monorepo & root config

**Files:**
- Create: `package.json`
- Create: `pnpm-workspace.yaml`
- Create: `.editorconfig`
- Create: `Makefile`
- Modify: `.gitignore` (déjà présent, on l'enrichit)
- Create: `README.md`
- Create: `CONTRIBUTING.md`

- [ ] **Step 1: Créer `package.json` racine**

```json
{
  "name": "nexcharge",
  "version": "0.1.0",
  "private": true,
  "packageManager": "pnpm@9.12.0",
  "scripts": {
    "dev": "docker compose -f infra/docker-compose.yml -f infra/docker-compose.dev.yml up",
    "down": "docker compose -f infra/docker-compose.yml down",
    "lint": "pnpm -r lint",
    "test": "pnpm -r test"
  },
  "devDependencies": {
    "prettier": "^3.3.3"
  },
  "engines": {
    "node": ">=20",
    "pnpm": ">=9"
  }
}
```

- [ ] **Step 2: Créer `pnpm-workspace.yaml`**

```yaml
packages:
  - "apps/web"
  - "packages/*"
```

- [ ] **Step 3: Créer `.editorconfig`**

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true

[*.{java,kt}]
indent_size = 4

[*.py]
indent_size = 4

[Makefile]
indent_style = tab
```

- [ ] **Step 4: Créer `Makefile`**

```makefile
.PHONY: up down logs ps clean test lint

up:
	docker compose -f infra/docker-compose.yml -f infra/docker-compose.dev.yml up -d

down:
	docker compose -f infra/docker-compose.yml down

logs:
	docker compose -f infra/docker-compose.yml logs -f

ps:
	docker compose -f infra/docker-compose.yml ps

clean:
	docker compose -f infra/docker-compose.yml down -v

test:
	cd services/core && ./gradlew test
	cd apps/ai && uv run pytest
	pnpm -F web test

lint:
	pnpm -r lint
	cd services/core && ./gradlew spotlessCheck
	cd apps/ai && uv run ruff check .
```

- [ ] **Step 5: Enrichir `.gitignore`**

Ouvrir `.gitignore` (déjà créé au sprint précédent) et vérifier qu'il contient bien les sections Node, Java, Python, IDE, OS, Docker. Si oui, ne rien faire. Sinon, fusionner avec ce contenu :

```
# (contenu déjà présent en grande partie)
# Vérifier que ces lignes existent :
.env
node_modules/
.next/
.gradle/
**/build/
__pycache__/
.venv/
.idea/
.vscode/
.DS_Store

# Ajouter si absent :
generated/
*.local.yml
infra/keycloak/data/
infra/postgres/data/
infra/minio/data/
```

- [ ] **Step 6: Créer `README.md`**

```markdown
# NEXCharge

Plateforme PWA mobile-first pour la recharge EV équitable et prédictive au NEX Tower & NEXTERACOM.

Hackathon Accenture Mauritius — NEXLevel Reinvented.

## Quickstart

```bash
make up        # lance toute la stack
make logs      # suit les logs
make down      # arrête
```

URLs locales :
- Web app : http://localhost
- Backend API : http://localhost/api
- AI service : http://localhost/ai
- Keycloak : http://localhost:8080
- MinIO console : http://localhost:9001

## Documentation

- Spec : [docs/superpowers/specs/](docs/superpowers/specs/)
- Plans d'implémentation : [docs/superpowers/plans/](docs/superpowers/plans/)
```

- [ ] **Step 7: Créer `CONTRIBUTING.md`**

```markdown
# Contributing

## Branches

- `main` : protégé, PR requises
- `feat/<short-name>` : nouvelles fonctionnalités
- `fix/<short-name>` : corrections de bugs
- `chore/<short-name>` : refacto, outillage

## Commit format

Conventional Commits : `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `refactor:`.

## Tests obligatoires avant PR

- `make lint`
- `make test`
- CI verte sur GitHub Actions
```

- [ ] **Step 8: Commit**

```bash
git add package.json pnpm-workspace.yaml .editorconfig Makefile .gitignore README.md CONTRIBUTING.md
git commit -m "chore: bootstrap monorepo root config"
```

---

### Task 2: docker-compose stack (Postgres + Redis + MinIO + Keycloak + Traefik)

**Files:**
- Create: `infra/docker-compose.yml`
- Create: `infra/docker-compose.dev.yml`
- Create: `infra/.env.example`
- Create: `infra/traefik/traefik.yml`
- Create: `infra/keycloak/realm-export.json`
- Create: `infra/postgres/init.sql`
- Create: `infra/minio/README.md`

- [ ] **Step 1: Créer `infra/.env.example`**

```dotenv
# Postgres
POSTGRES_USER=nexcharge
POSTGRES_PASSWORD=changeme-in-prod
POSTGRES_DB=nexcharge

# Keycloak admin
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=changeme-in-prod

# MinIO
MINIO_ROOT_USER=nexcharge
MINIO_ROOT_PASSWORD=changeme-in-prod-min12chars

# Service URLs (interne docker network)
DATABASE_URL=jdbc:postgresql://postgres:5432/nexcharge
REDIS_URL=redis://redis:6379
MINIO_ENDPOINT=http://minio:9000

# OIDC (dev points to local Keycloak; prod points to Entra)
OIDC_ISSUER_URL=http://keycloak:8080/realms/nexcharge
OIDC_CLIENT_ID=nexcharge-web
OIDC_CLIENT_SECRET=changeme-in-prod

# AI service
AI_SERVICE_URL=http://ai:8000
ANTHROPIC_API_KEY=

# Business config (enforced from Sprint 2 onwards, declared here for stability)
NEXCHARGE_MAX_BOOKING_DURATION_HOURS=3
NEXCHARGE_SUFFICIENT_CHARGE_THRESHOLD_PCT=80
NEXCHARGE_AUTO_RELEASE_GRACE_MINUTES=15
NEXCHARGE_NO_SHOW_COOLDOWN_HOURS=48
NEXCHARGE_MAX_BOOKINGS_PER_WEEK=5
NEXCHARGE_BIAS_GINI_ALERT_THRESHOLD=0.4
NEXCHARGE_WORKPLACE_OPS_EMAIL=Workplace.Mauritius.OfficeServices@accenture.com
```

- [ ] **Step 2: Créer `infra/postgres/init.sql`**

```sql
-- Créé automatiquement par l'image postgres si DB n'existe pas (POSTGRES_DB).
-- Ce script ajoute des extensions utiles.
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
```

- [ ] **Step 3: Créer `infra/traefik/traefik.yml`**

```yaml
api:
  dashboard: true
  insecure: true   # dev only, à durcir en prod

entryPoints:
  web:
    address: ":80"

providers:
  docker:
    exposedByDefault: false
    network: nexcharge

log:
  level: INFO
```

- [ ] **Step 4: Créer `infra/keycloak/realm-export.json`**

Realm Keycloak pré-configuré avec un realm `nexcharge`, un client `nexcharge-web`, et un user de test `driver@accenture.local` / mot de passe `driver`. Coller exactement :

```json
{
  "realm": "nexcharge",
  "enabled": true,
  "sslRequired": "none",
  "registrationAllowed": false,
  "loginWithEmailAllowed": true,
  "duplicateEmailsAllowed": false,
  "resetPasswordAllowed": false,
  "editUsernameAllowed": false,
  "bruteForceProtected": true,
  "roles": {
    "realm": [
      { "name": "DRIVER" },
      { "name": "FACILITY_MANAGER" },
      { "name": "SUSTAINABILITY_OFFICER" },
      { "name": "ADMIN" }
    ]
  },
  "users": [
    {
      "username": "driver",
      "email": "driver@accenture.local",
      "firstName": "Demo",
      "lastName": "Driver",
      "enabled": true,
      "emailVerified": true,
      "credentials": [
        { "type": "password", "value": "driver", "temporary": false }
      ],
      "realmRoles": ["DRIVER"]
    }
  ],
  "clients": [
    {
      "clientId": "nexcharge-web",
      "enabled": true,
      "publicClient": false,
      "secret": "changeme-in-prod",
      "redirectUris": ["http://localhost/api/auth/callback/keycloak"],
      "webOrigins": ["http://localhost"],
      "standardFlowEnabled": true,
      "directAccessGrantsEnabled": false,
      "protocol": "openid-connect",
      "attributes": {
        "post.logout.redirect.uris": "http://localhost"
      },
      "defaultClientScopes": ["profile", "email", "roles"]
    }
  ],
  "defaultRoles": ["DRIVER"]
}
```

- [ ] **Step 5: Créer `infra/minio/README.md`**

```markdown
# MinIO setup

Au premier démarrage, créer manuellement le bucket `reports` :

```bash
docker compose exec minio mc alias set local http://localhost:9000 $MINIO_ROOT_USER $MINIO_ROOT_PASSWORD
docker compose exec minio mc mb local/reports
```

Console web : http://localhost:9001
```

- [ ] **Step 6: Créer `infra/docker-compose.yml` (stack principale)**

```yaml
networks:
  nexcharge:
    name: nexcharge

volumes:
  postgres-data:
  minio-data:
  keycloak-data:

services:
  traefik:
    image: traefik:v3.1
    ports:
      - "80:80"
      - "8081:8080"   # dashboard traefik
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/traefik.yml:/etc/traefik/traefik.yml:ro
    networks: [nexcharge]

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/00-init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks: [nexcharge]

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks: [nexcharge]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    volumes:
      - minio-data:/data
    ports:
      - "9001:9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks: [nexcharge]

  keycloak:
    image: quay.io/keycloak/keycloak:25.0
    command: ["start-dev", "--import-realm"]
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: ${KEYCLOAK_ADMIN}
      KC_BOOTSTRAP_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_HTTP_ENABLED: "true"
      KC_HOSTNAME_STRICT: "false"
    volumes:
      - keycloak-data:/opt/keycloak/data
      - ./keycloak/realm-export.json:/opt/keycloak/data/import/realm.json:ro
    ports:
      - "8080:8080"
    networks: [nexcharge]

  core:
    build:
      context: ../services/core
    env_file: .env
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.core.rule=PathPrefix(`/api`)"
      - "traefik.http.services.core.loadbalancer.server.port=8090"
    networks: [nexcharge]

  ai:
    build:
      context: ../apps/ai
    env_file: .env
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.ai.rule=PathPrefix(`/ai`)"
      - "traefik.http.services.ai.loadbalancer.server.port=8000"
    networks: [nexcharge]

  web:
    build:
      context: ../apps/web
    env_file: .env
    depends_on:
      - core
      - keycloak
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=PathPrefix(`/`)"
      - "traefik.http.services.web.loadbalancer.server.port=3000"
    networks: [nexcharge]
```

- [ ] **Step 7: Créer `infra/docker-compose.dev.yml`**

```yaml
# Override pour dev : hot reload, ports exposés directement
services:
  core:
    build:
      target: dev
    volumes:
      - ../services/core/src:/app/src
      - ../services/core/build.gradle.kts:/app/build.gradle.kts:ro
      - ../services/core/settings.gradle.kts:/app/settings.gradle.kts:ro
    ports:
      - "8090:8090"
    environment:
      SPRING_PROFILES_ACTIVE: dev

  ai:
    build:
      target: dev
    volumes:
      - ../apps/ai/src:/app/src
    ports:
      - "8000:8000"
    command: ["uv", "run", "uvicorn", "nexcharge_ai.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

  web:
    build:
      target: dev
    volumes:
      - ../apps/web/src:/app/src
      - ../apps/web/public:/app/public
    ports:
      - "3000:3000"
    command: ["pnpm", "dev"]
```

- [ ] **Step 8: Copier le template `.env.example` en `.env`**

```bash
cp infra/.env.example infra/.env
```

(Le `.env` est dans `.gitignore`, donc local seulement.)

- [ ] **Step 9: Vérifier que les services d'infra démarrent**

Run: `cd infra && docker compose up -d postgres redis minio keycloak traefik`
Expected: 5 services en `running`, healthchecks passent en ~30s.

Vérifier :
- `docker compose ps` → tous `Up (healthy)` sauf traefik (pas de healthcheck défini, doit être `Up`).
- `curl http://localhost:8080` → redirige vers `/admin` Keycloak.
- `curl http://localhost:9001` → console MinIO.

- [ ] **Step 10: Arrêter les services**

```bash
docker compose down
```

(Les services applicatifs `core`, `ai`, `web` ne sont pas encore buildables — c'est normal, on les ajoute aux tâches suivantes.)

- [ ] **Step 11: Commit**

```bash
git add infra/
git commit -m "chore(infra): docker-compose stack with postgres/redis/minio/keycloak/traefik"
```

---

### Task 3: Backend Java — bootstrap Spring Boot avec Gradle

**Files:**
- Create: `services/core/settings.gradle.kts`
- Create: `services/core/build.gradle.kts`
- Create: `services/core/gradle.properties`
- Create: `services/core/gradle/wrapper/gradle-wrapper.properties`
- Create: `services/core/Dockerfile`
- Create: `services/core/.env.example`
- Create: `services/core/src/main/java/com/accenture/nexcharge/NexchargeApplication.java`
- Create: `services/core/src/main/resources/application.yml`
- Create: `services/core/src/main/resources/application-dev.yml`
- Create: `services/core/src/test/java/com/accenture/nexcharge/NexchargeApplicationTests.java`

- [ ] **Step 1: Créer `services/core/settings.gradle.kts`**

```kotlin
rootProject.name = "nexcharge-core"
```

- [ ] **Step 2: Créer `services/core/gradle.properties`**

```properties
org.gradle.jvmargs=-Xmx2g
org.gradle.parallel=true
org.gradle.caching=true
```

- [ ] **Step 3: Initialiser Gradle wrapper**

Run: `cd services/core && gradle wrapper --gradle-version 8.10`
Expected: création de `gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.jar`, `gradle/wrapper/gradle-wrapper.properties`.

(Si `gradle` n'est pas installé, télécharger depuis https://gradle.org/install/. Alternative : copier un `gradle/wrapper/` depuis un autre projet Spring Boot et écrire `gradlew`/`gradlew.bat` à la main.)

- [ ] **Step 4: Créer `services/core/build.gradle.kts`**

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    id("com.diffplug.spotless") version "6.25.0"
    id("org.springdoc.openapi-gradle-plugin") version "1.9.0"
}

group = "com.accenture"
version = "0.1.0"

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-webflux")  // pour WebClient (AI service)
    implementation("org.springframework.boot:spring-boot-starter-websocket")

    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")
    runtimeOnly("org.postgresql:postgresql")

    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")

    compileOnly("org.projectlombok:lombok:1.18.34")
    annotationProcessor("org.projectlombok:lombok:1.18.34")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
    testImplementation("org.testcontainers:junit-jupiter:1.20.1")
    testImplementation("org.testcontainers:postgresql:1.20.1")
    testImplementation("com.redis:testcontainers-redis:2.2.2")
    testImplementation("io.rest-assured:rest-assured:5.5.0")
}

tasks.test {
    useJUnitPlatform()
}

spotless {
    java {
        googleJavaFormat("1.22.0").aosp().reflowLongStrings()
        formatAnnotations()
    }
}

springBoot {
    mainClass = "com.accenture.nexcharge.NexchargeApplication"
}
```

- [ ] **Step 5: Créer `services/core/.env.example`**

```dotenv
SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/nexcharge
SPRING_DATASOURCE_USERNAME=nexcharge
SPRING_DATASOURCE_PASSWORD=changeme-in-prod
SPRING_DATA_REDIS_HOST=redis
SPRING_DATA_REDIS_PORT=6379
SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI=http://keycloak:8080/realms/nexcharge
NEXCHARGE_AI_SERVICE_URL=http://ai:8000
SERVER_PORT=8090
```

- [ ] **Step 6: Créer `services/core/src/main/resources/application.yml`**

```yaml
server:
  port: ${SERVER_PORT:8090}

spring:
  application:
    name: nexcharge-core
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        jdbc.lob.non_contextual_creation: true
  flyway:
    enabled: true
    locations: classpath:db/migration
  data:
    redis:
      host: ${SPRING_DATA_REDIS_HOST}
      port: ${SPRING_DATA_REDIS_PORT}
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI}

nexcharge:
  ai:
    service-url: ${NEXCHARGE_AI_SERVICE_URL}

management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus
  endpoint:
    health:
      show-details: when-authorized

springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
```

- [ ] **Step 7: Créer `services/core/src/main/resources/application-dev.yml`**

```yaml
logging:
  level:
    com.accenture.nexcharge: DEBUG
    org.springframework.security: DEBUG
```

- [ ] **Step 8: Créer `services/core/src/main/java/com/accenture/nexcharge/NexchargeApplication.java`**

```java
package com.accenture.nexcharge;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class NexchargeApplication {
    public static void main(String[] args) {
        SpringApplication.run(NexchargeApplication.class, args);
    }
}
```

- [ ] **Step 9: Créer un test minimal `services/core/src/test/java/com/accenture/nexcharge/NexchargeApplicationTests.java`**

```java
package com.accenture.nexcharge;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
@ActiveProfiles("test")
class NexchargeApplicationTests {

    @Test
    void contextLoads() {
        // Smoke test: Spring context démarre sans erreur.
    }
}
```

- [ ] **Step 10: Créer `services/core/src/test/resources/application-test.yml`**

```yaml
# Test profile : DB et Redis seront fournis par Testcontainers (Task 5).
# Pour l'instant, juste un context load minimal sans Flyway/JPA.
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
      - org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
      - org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration
      - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
      - org.springframework.boot.autoconfigure.security.oauth2.resource.servlet.OAuth2ResourceServerAutoConfiguration
```

- [ ] **Step 11: Lancer le test**

Run: `cd services/core && ./gradlew test`
Expected: BUILD SUCCESSFUL, 1 test passed.

- [ ] **Step 12: Créer `services/core/Dockerfile` (multi-stage avec target dev)**

```dockerfile
# syntax=docker/dockerfile:1.7

FROM eclipse-temurin:21-jdk AS dev
WORKDIR /app
COPY gradle gradle
COPY gradlew settings.gradle.kts build.gradle.kts gradle.properties ./
RUN ./gradlew --version
COPY src src
EXPOSE 8090
CMD ["./gradlew", "bootRun", "--args=--spring.profiles.active=dev"]

FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY gradle gradle
COPY gradlew settings.gradle.kts build.gradle.kts gradle.properties ./
RUN ./gradlew --version
COPY src src
RUN ./gradlew bootJar -x test

FROM eclipse-temurin:21-jre AS runtime
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8090
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

- [ ] **Step 13: Build l'image Docker**

Run: `cd services/core && docker build -t nexcharge-core:dev --target build .`
Expected: build success, image `nexcharge-core:dev` créée.

- [ ] **Step 14: Commit**

```bash
git add services/core/
git commit -m "feat(core): bootstrap Spring Boot 3 + Gradle + Java 21"
```

---

### Task 3b: Backend Java — properties métier (BusinessProperties)

Câbler les paramètres de config métier dès le Sprint 1 (même si leur logique d'usage est implémentée plus tard) — garantit que le naming et les valeurs par défaut sont stables.

**Files:**
- Create: `services/core/src/main/java/com/accenture/nexcharge/common/config/BusinessProperties.java`
- Modify: `services/core/src/main/java/com/accenture/nexcharge/NexchargeApplication.java`
- Modify: `services/core/src/main/resources/application.yml`
- Create: `services/core/src/test/java/com/accenture/nexcharge/common/config/BusinessPropertiesTest.java`

- [ ] **Step 1: Créer `BusinessProperties.java`**

```java
package com.accenture.nexcharge.common.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;

/**
 * Paramètres métier alignés avec les guidelines Accenture pour les bornes EV
 * au NEX Tower & Nexteracom. Valeurs par défaut documentées dans la spec.
 */
@ConfigurationProperties(prefix = "nexcharge.business")
@Validated
public record BusinessProperties(
        @Min(1) int maxBookingDurationHours,
        @Min(1) int sufficientChargeThresholdPct,
        @Min(0) int autoReleaseGraceMinutes,
        @Min(0) int noShowCooldownHours,
        @Min(1) int maxBookingsPerWeek,
        double biasGiniAlertThreshold,
        @NotBlank @Email String workplaceOpsEmail) {}
```

- [ ] **Step 2: Activer la classe dans `NexchargeApplication.java`**

```java
package com.accenture.nexcharge;

import com.accenture.nexcharge.common.config.BusinessProperties;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.ConfigurationPropertiesScan;

@SpringBootApplication
@ConfigurationPropertiesScan
public class NexchargeApplication {
    public static void main(String[] args) {
        SpringApplication.run(NexchargeApplication.class, args);
    }
}
```

- [ ] **Step 3: Étendre `application.yml` avec les défauts**

Ajouter à la fin de `services/core/src/main/resources/application.yml` :

```yaml
nexcharge:
  ai:
    service-url: ${NEXCHARGE_AI_SERVICE_URL}
  business:
    max-booking-duration-hours: ${NEXCHARGE_MAX_BOOKING_DURATION_HOURS:3}
    sufficient-charge-threshold-pct: ${NEXCHARGE_SUFFICIENT_CHARGE_THRESHOLD_PCT:80}
    auto-release-grace-minutes: ${NEXCHARGE_AUTO_RELEASE_GRACE_MINUTES:15}
    no-show-cooldown-hours: ${NEXCHARGE_NO_SHOW_COOLDOWN_HOURS:48}
    max-bookings-per-week: ${NEXCHARGE_MAX_BOOKINGS_PER_WEEK:5}
    bias-gini-alert-threshold: ${NEXCHARGE_BIAS_GINI_ALERT_THRESHOLD:0.4}
    workplace-ops-email: ${NEXCHARGE_WORKPLACE_OPS_EMAIL:Workplace.Mauritius.OfficeServices@accenture.com}
```

(Remplacer le bloc existant `nexcharge: ai: ...` qui n'avait que `service-url` — la nouvelle version le contient.)

- [ ] **Step 4: Étendre `application-test.yml` avec les valeurs de test**

Ajouter à `services/core/src/test/resources/application-test.yml` :

```yaml
nexcharge:
  ai:
    service-url: http://localhost:9999/unused-in-test
  business:
    max-booking-duration-hours: 3
    sufficient-charge-threshold-pct: 80
    auto-release-grace-minutes: 15
    no-show-cooldown-hours: 48
    max-bookings-per-week: 5
    bias-gini-alert-threshold: 0.4
    workplace-ops-email: ops-test@accenture.local
```

(Remplacer la ligne `nexcharge: ai: service-url: ...` existante par ce bloc.)

- [ ] **Step 5: Écrire `BusinessPropertiesTest.java`**

```java
package com.accenture.nexcharge.common.config;

import static org.assertj.core.api.Assertions.assertThat;

import com.accenture.nexcharge.AbstractIntegrationTest;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;

class BusinessPropertiesTest extends AbstractIntegrationTest {

    @Autowired BusinessProperties props;

    @Test
    void defaults_match_spec() {
        assertThat(props.maxBookingDurationHours()).isEqualTo(3);
        assertThat(props.sufficientChargeThresholdPct()).isEqualTo(80);
        assertThat(props.autoReleaseGraceMinutes()).isEqualTo(15);
        assertThat(props.noShowCooldownHours()).isEqualTo(48);
        assertThat(props.maxBookingsPerWeek()).isEqualTo(5);
        assertThat(props.biasGiniAlertThreshold()).isEqualTo(0.4);
        assertThat(props.workplaceOpsEmail()).isEqualTo("ops-test@accenture.local");
    }
}
```

- [ ] **Step 6: Lancer les tests**

Run: `cd services/core && ./gradlew test`
Expected: BUILD SUCCESSFUL, +1 test passed.

- [ ] **Step 7: Commit**

```bash
git add services/core/
git commit -m "feat(core): BusinessProperties — config métier alignée guidelines Accenture"
```

---

### Task 4: Backend Java — schéma Postgres complet via Flyway

**Files:**
- Create: `services/core/src/main/resources/db/migration/V1__init.sql`

- [ ] **Step 1: Créer la migration V1**

Cette migration crée toutes les tables principales du modèle de données du sprint 1. C'est volumineux mais on le fait en une fois pour éviter les chevauchements.

```sql
-- V1__init.sql : schéma initial NEXCharge
-- Toutes les tables principales de la spec, prêtes pour les sprints suivants.

CREATE TYPE user_role AS ENUM ('DRIVER', 'FACILITY_MANAGER', 'SUSTAINABILITY_OFFICER', 'ADMIN');
CREATE TYPE charger_status AS ENUM ('AVAILABLE', 'OCCUPIED', 'RESERVED', 'OFFLINE', 'FAULTED');
CREATE TYPE booking_status AS ENUM ('RESERVED', 'ACTIVE', 'COMPLETED', 'CANCELLED', 'RELEASED_AUTO', 'NO_SHOW');
CREATE TYPE session_status AS ENUM ('IN_PROGRESS', 'COMPLETED', 'INTERRUPTED');

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entra_oid       VARCHAR(255) UNIQUE NOT NULL,
    email           VARCHAR(255) UNIQUE NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            user_role NOT NULL DEFAULT 'DRIVER',
    fair_share_kwh  NUMERIC(10, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_entra_oid ON users(entra_oid);
CREATE INDEX idx_users_email ON users(email);

CREATE TABLE chargers (
    id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    ocpp_id          VARCHAR(255) UNIQUE NOT NULL,
    site             VARCHAR(64) NOT NULL,           -- 'NEX_TOWER' ou 'NEXTERACOM'
    location_label   VARCHAR(255) NOT NULL,
    status           charger_status NOT NULL DEFAULT 'OFFLINE',
    max_power_kw     NUMERIC(6, 2) NOT NULL,
    connector_type   VARCHAR(64) NOT NULL,           -- 'TYPE2', 'CCS', 'CHADEMO'
    last_heartbeat   TIMESTAMPTZ,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_chargers_status ON chargers(status);
CREATE INDEX idx_chargers_site ON chargers(site);

CREATE TABLE charger_status_log (
    id              BIGSERIAL PRIMARY KEY,
    charger_id      UUID NOT NULL REFERENCES chargers(id) ON DELETE CASCADE,
    status          charger_status NOT NULL,
    event_payload   JSONB,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_charger_status_log_charger ON charger_status_log(charger_id, recorded_at DESC);

CREATE TABLE bookings (
    id                 UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id            UUID NOT NULL REFERENCES users(id),
    charger_id         UUID NOT NULL REFERENCES chargers(id),
    slot_start         TIMESTAMPTZ NOT NULL,
    slot_end           TIMESTAMPTZ NOT NULL,
    status             booking_status NOT NULL DEFAULT 'RESERVED',
    predicted_demand   NUMERIC(5, 4),
    fairness_score     NUMERIC(5, 4),
    released_reason    VARCHAR(255),
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT booking_slot_valid CHECK (slot_end > slot_start)
);

CREATE INDEX idx_bookings_user ON bookings(user_id);
CREATE INDEX idx_bookings_charger ON bookings(charger_id);
CREATE INDEX idx_bookings_slot ON bookings(slot_start, slot_end);
CREATE INDEX idx_bookings_status ON bookings(status);

-- Empêche le double-booking sur un même charger pour des slots qui se chevauchent
-- (sauf pour les bookings annulés/no-show).
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE bookings
    ADD CONSTRAINT no_overlap_active_booking
    EXCLUDE USING gist (
        charger_id WITH =,
        tstzrange(slot_start, slot_end, '[)') WITH &&
    ) WHERE (status IN ('RESERVED', 'ACTIVE'));

CREATE TABLE charging_sessions (
    id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    booking_id        UUID UNIQUE REFERENCES bookings(id),
    charger_id        UUID NOT NULL REFERENCES chargers(id),
    user_id           UUID NOT NULL REFERENCES users(id),
    started_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ended_at          TIMESTAMPTZ,
    kwh_total         NUMERIC(10, 3),
    peak_power_kw     NUMERIC(6, 2),
    co2_kg_avoided    NUMERIC(10, 3),
    status            session_status NOT NULL DEFAULT 'IN_PROGRESS',
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sessions_user ON charging_sessions(user_id);
CREATE INDEX idx_sessions_charger ON charging_sessions(charger_id);
CREATE INDEX idx_sessions_status ON charging_sessions(status);

CREATE TABLE meter_values (
    id              BIGSERIAL PRIMARY KEY,
    session_id      UUID NOT NULL REFERENCES charging_sessions(id) ON DELETE CASCADE,
    recorded_at     TIMESTAMPTZ NOT NULL,
    kwh             NUMERIC(10, 3) NOT NULL,
    power_kw        NUMERIC(6, 2),
    voltage         NUMERIC(6, 2),
    current         NUMERIC(6, 2),
    soc_percent     NUMERIC(5, 2)
);

CREATE INDEX idx_meter_values_session ON meter_values(session_id, recorded_at);

CREATE TABLE usage_quotas (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    period          VARCHAR(7) NOT NULL,             -- 'YYYY-MM'
    kwh_used        NUMERIC(10, 3) NOT NULL DEFAULT 0,
    sessions_count  INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, period)
);

CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    kind            VARCHAR(64) NOT NULL,            -- 'PRE_SESSION', 'END_SESSION', 'AUTO_RELEASE', 'ANOMALY'
    channel         VARCHAR(32) NOT NULL,            -- 'PUSH', 'EMAIL'
    payload         JSONB NOT NULL,
    sent_at         TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications(user_id);

CREATE TABLE ai_predictions (
    id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    kind             VARCHAR(64) NOT NULL,           -- 'DEMAND_FORECAST', 'ANOMALY_SCORE', 'FAIRNESS_SCORE'
    target_id        UUID,                           -- charger_id, user_id, ou booking_id selon kind
    horizon_minutes  INTEGER,
    value            NUMERIC(10, 4) NOT NULL,
    confidence       NUMERIC(5, 4),
    model_version    VARCHAR(64) NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ai_predictions_target ON ai_predictions(kind, target_id, created_at DESC);

CREATE TABLE ai_explanations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    decision_kind   VARCHAR(64) NOT NULL,            -- 'BOOKING_RANK', 'AUTO_RELEASE', 'ANOMALY_FLAG'
    decision_id     UUID,                            -- booking_id, session_id, etc.
    inputs          JSONB NOT NULL,
    output_value    JSONB,
    explanation     TEXT NOT NULL,
    model_version   VARCHAR(64),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ai_explanations_decision ON ai_explanations(decision_kind, decision_id);

-- audit_log : append-only (pas d'UPDATE/DELETE — enforce via revocation des privilèges en prod).
CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    actor_user_id   UUID REFERENCES users(id),
    action          VARCHAR(128) NOT NULL,           -- 'BOOKING_OVERRIDE', 'ROLE_CHANGE', 'AUTO_RELEASE', etc.
    target_type     VARCHAR(64),
    target_id       VARCHAR(255),
    payload         JSONB,
    reason          TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_actor ON audit_log(actor_user_id);
CREATE INDEX idx_audit_log_action ON audit_log(action);
CREATE INDEX idx_audit_log_created ON audit_log(created_at DESC);

CREATE TABLE report_exports (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    period          VARCHAR(7) NOT NULL,
    format          VARCHAR(16) NOT NULL,            -- 'PDF', 'CSV'
    minio_path      VARCHAR(512) NOT NULL,
    narration_text  TEXT,
    generated_by    UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_report_exports_period ON report_exports(period);
```

- [ ] **Step 2: Vérifier la syntaxe SQL en local (sans Spring)**

Run:
```bash
docker compose -f infra/docker-compose.yml up -d postgres
sleep 5
docker compose -f infra/docker-compose.yml exec -T postgres psql -U nexcharge -d nexcharge -f - < services/core/src/main/resources/db/migration/V1__init.sql
```

Expected: pas d'erreur SQL. Si erreur, corriger et réessayer.

Puis nettoyer :
```bash
docker compose -f infra/docker-compose.yml down -v
```

(On purge le volume car Flyway ne pourra pas appliquer V1 si les tables existent déjà — Flyway le fera proprement à la prochaine étape.)

- [ ] **Step 3: Commit**

```bash
git add services/core/src/main/resources/db/migration/V1__init.sql
git commit -m "feat(core): initial Postgres schema (Flyway V1)"
```

---

### Task 5: Backend Java — entités JPA pour User + endpoint /users/me

**Files:**
- Create: `services/core/src/main/java/com/accenture/nexcharge/users/UserRole.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/users/User.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/users/UserRepository.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/users/UserService.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/users/UserDto.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/users/UserController.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/security/SecurityConfig.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/security/CurrentUserResolver.java`
- Create: `services/core/src/test/java/com/accenture/nexcharge/AbstractIntegrationTest.java`
- Create: `services/core/src/test/java/com/accenture/nexcharge/users/UserControllerIT.java`

- [ ] **Step 1: Créer `UserRole.java` (enum mappé sur le type Postgres)**

```java
package com.accenture.nexcharge.users;

public enum UserRole {
    DRIVER,
    FACILITY_MANAGER,
    SUSTAINABILITY_OFFICER,
    ADMIN
}
```

- [ ] **Step 2: Créer `User.java` (entité JPA)**

```java
package com.accenture.nexcharge.users;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.OffsetDateTime;
import java.util.UUID;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(name = "entra_oid", unique = true, nullable = false)
    private String entraOid;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(name = "display_name", nullable = false)
    private String displayName;

    @Enumerated(EnumType.STRING)
    @JdbcTypeCode(SqlTypes.NAMED_ENUM)
    @Column(nullable = false, columnDefinition = "user_role")
    private UserRole role = UserRole.DRIVER;

    @Column(name = "fair_share_kwh")
    private BigDecimal fairShareKwh;

    @Column(name = "created_at", nullable = false, updatable = false)
    private OffsetDateTime createdAt;

    @Column(name = "updated_at", nullable = false)
    private OffsetDateTime updatedAt;

    @PrePersist
    void onCreate() {
        OffsetDateTime now = OffsetDateTime.now();
        this.createdAt = now;
        this.updatedAt = now;
    }

    @PreUpdate
    void onUpdate() {
        this.updatedAt = OffsetDateTime.now();
    }

    public User(String entraOid, String email, String displayName) {
        this.entraOid = entraOid;
        this.email = email;
        this.displayName = displayName;
    }
}
```

- [ ] **Step 3: Créer `UserRepository.java`**

```java
package com.accenture.nexcharge.users;

import java.util.Optional;
import java.util.UUID;
import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<User, UUID> {
    Optional<User> findByEntraOid(String entraOid);
}
```

- [ ] **Step 4: Créer `UserService.java` (provisioning JIT)**

```java
package com.accenture.nexcharge.users;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class UserService {

    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    /**
     * Récupère l'utilisateur correspondant au JWT, ou le crée si premier login.
     * Mapping : Entra/Keycloak `sub` → User.entraOid.
     */
    @Transactional
    public User findOrCreate(String entraOid, String email, String displayName) {
        return repository
                .findByEntraOid(entraOid)
                .orElseGet(() -> repository.save(new User(entraOid, email, displayName)));
    }
}
```

- [ ] **Step 5: Créer `UserDto.java` (DTO de réponse, pas d'expo des entités JPA)**

```java
package com.accenture.nexcharge.users;

import java.math.BigDecimal;
import java.util.UUID;

public record UserDto(
        UUID id,
        String email,
        String displayName,
        UserRole role,
        BigDecimal fairShareKwh) {

    public static UserDto from(User user) {
        return new UserDto(
                user.getId(),
                user.getEmail(),
                user.getDisplayName(),
                user.getRole(),
                user.getFairShareKwh());
    }
}
```

- [ ] **Step 6: Créer `CurrentUserResolver.java` (utilitaire pour extraire le user du JWT)**

```java
package com.accenture.nexcharge.security;

import com.accenture.nexcharge.users.User;
import com.accenture.nexcharge.users.UserService;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.stereotype.Component;

@Component
public class CurrentUserResolver {

    private final UserService userService;

    public CurrentUserResolver(UserService userService) {
        this.userService = userService;
    }

    /**
     * À appeler depuis un controller avec @AuthenticationPrincipal Jwt jwt.
     * Provisionne le User en base si premier login.
     */
    public User resolve(Jwt jwt) {
        String entraOid = jwt.getSubject();
        String email = jwt.getClaimAsString("email");
        String name = jwt.getClaimAsString("name");
        if (name == null || name.isBlank()) {
            name = jwt.getClaimAsString("preferred_username");
        }
        return userService.findOrCreate(entraOid, email, name);
    }
}
```

- [ ] **Step 7: Créer `UserController.java` avec endpoint `/users/me`**

```java
package com.accenture.nexcharge.users;

import com.accenture.nexcharge.security.CurrentUserResolver;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/users")
public class UserController {

    private final CurrentUserResolver currentUser;

    public UserController(CurrentUserResolver currentUser) {
        this.currentUser = currentUser;
    }

    @GetMapping("/me")
    public UserDto me(@AuthenticationPrincipal Jwt jwt) {
        User user = currentUser.resolve(jwt);
        return UserDto.from(user);
    }
}
```

- [ ] **Step 8: Créer `SecurityConfig.java`**

```java
package com.accenture.nexcharge.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
                .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/healthz/**", "/actuator/health", "/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html")
                        .permitAll()
                        .anyRequest().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}));
        return http.build();
    }
}
```

- [ ] **Step 9: Créer `AbstractIntegrationTest.java` (Testcontainers shared)**

```java
package com.accenture.nexcharge;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.test.context.ActiveProfiles;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@Testcontainers
public abstract class AbstractIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:16-alpine")
                    .withDatabaseName("nexcharge")
                    .withUsername("test")
                    .withPassword("test");
}
```

Mettre à jour `services/core/src/test/resources/application-test.yml` (créé en Task 3) — remplacer son contenu par :

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
    locations: classpath:db/migration
  security:
    oauth2:
      resourceserver:
        jwt:
          # JWKS mocké via spring-security-test ; aucun appel réseau
          issuer-uri: http://localhost/test-realm
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
      - org.springframework.boot.autoconfigure.security.oauth2.resource.servlet.OAuth2ResourceServerAutoConfiguration

nexcharge:
  ai:
    service-url: http://localhost:9999/unused-in-test
```

(On désactive l'auto-config OAuth2 en test pour utiliser `MockMvc` avec `@WithMockUser`/`jwt()` directement.)

- [ ] **Step 10: Écrire le test d'intégration `UserControllerIT.java`**

```java
package com.accenture.nexcharge.users;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import com.accenture.nexcharge.AbstractIntegrationTest;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.test.web.servlet.MockMvc;

@AutoConfigureMockMvc
class UserControllerIT extends AbstractIntegrationTest {

    @Autowired MockMvc mockMvc;
    @Autowired UserRepository userRepository;

    @Test
    void me_provisions_user_on_first_call() throws Exception {
        long initialCount = userRepository.count();

        mockMvc.perform(get("/api/users/me")
                        .with(jwt().jwt(jwt -> jwt
                                .subject("entra-oid-123")
                                .claim("email", "alice@accenture.com")
                                .claim("name", "Alice Test"))))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.email").value("alice@accenture.com"))
                .andExpect(jsonPath("$.displayName").value("Alice Test"))
                .andExpect(jsonPath("$.role").value("DRIVER"));

        assertThat(userRepository.count()).isEqualTo(initialCount + 1);
    }

    @Test
    void me_returns_existing_user_on_second_call() throws Exception {
        // Premier appel (provisionne)
        mockMvc.perform(get("/api/users/me")
                        .with(jwt().jwt(jwt -> jwt
                                .subject("entra-oid-456")
                                .claim("email", "bob@accenture.com")
                                .claim("name", "Bob"))))
                .andExpect(status().isOk());

        long countAfterFirst = userRepository.count();

        // Deuxième appel : ne crée pas de nouveau user
        mockMvc.perform(get("/api/users/me")
                        .with(jwt().jwt(jwt -> jwt
                                .subject("entra-oid-456")
                                .claim("email", "bob@accenture.com")
                                .claim("name", "Bob"))))
                .andExpect(status().isOk());

        assertThat(userRepository.count()).isEqualTo(countAfterFirst);
    }

    @Test
    void me_unauthenticated_returns_401() throws Exception {
        mockMvc.perform(get("/api/users/me"))
                .andExpect(status().isUnauthorized());
    }
}
```

- [ ] **Step 11: Lancer les tests**

Run: `cd services/core && ./gradlew test`
Expected: BUILD SUCCESSFUL, 4 tests passed (1 contextLoads + 3 user tests).

- [ ] **Step 12: Commit**

```bash
git add services/core/src/main services/core/src/test
git commit -m "feat(core): User entity, /users/me endpoint, OIDC security, JIT provisioning"
```

---

### Task 6: Backend Java — autres entités JPA (Charger, Booking, ChargingSession, MeterValue)

Pas de logique métier ici — seulement les entités/enums/repositories pour valider que le schéma Flyway et les mappings JPA sont cohérents (`hibernate.ddl-auto=validate` lèvera une erreur en cas de divergence).

**Files:**
- Create: `services/core/src/main/java/com/accenture/nexcharge/chargers/ChargerStatus.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/chargers/Charger.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/chargers/ChargerRepository.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/booking/BookingStatus.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/booking/Booking.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/booking/BookingRepository.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/sessions/SessionStatus.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/sessions/ChargingSession.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/sessions/MeterValue.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/sessions/SessionRepository.java`
- Create: `services/core/src/test/java/com/accenture/nexcharge/SchemaValidationIT.java`

- [ ] **Step 1: Créer les enums**

`ChargerStatus.java` :
```java
package com.accenture.nexcharge.chargers;

public enum ChargerStatus { AVAILABLE, OCCUPIED, RESERVED, OFFLINE, FAULTED }
```

`BookingStatus.java` :
```java
package com.accenture.nexcharge.booking;

public enum BookingStatus { RESERVED, ACTIVE, COMPLETED, CANCELLED, RELEASED_AUTO, NO_SHOW }
```

`SessionStatus.java` :
```java
package com.accenture.nexcharge.sessions;

public enum SessionStatus { IN_PROGRESS, COMPLETED, INTERRUPTED }
```

- [ ] **Step 2: Créer `Charger.java`**

```java
package com.accenture.nexcharge.chargers;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.OffsetDateTime;
import java.util.UUID;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

@Entity
@Table(name = "chargers")
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Charger {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(name = "ocpp_id", unique = true, nullable = false)
    private String ocppId;

    @Column(nullable = false)
    private String site;

    @Column(name = "location_label", nullable = false)
    private String locationLabel;

    @Enumerated(EnumType.STRING)
    @JdbcTypeCode(SqlTypes.NAMED_ENUM)
    @Column(nullable = false, columnDefinition = "charger_status")
    private ChargerStatus status = ChargerStatus.OFFLINE;

    @Column(name = "max_power_kw", nullable = false)
    private BigDecimal maxPowerKw;

    @Column(name = "connector_type", nullable = false)
    private String connectorType;

    @Column(name = "last_heartbeat")
    private OffsetDateTime lastHeartbeat;

    @Column(name = "created_at", nullable = false, updatable = false)
    private OffsetDateTime createdAt;

    @Column(name = "updated_at", nullable = false)
    private OffsetDateTime updatedAt;

    @PrePersist
    void onCreate() {
        OffsetDateTime now = OffsetDateTime.now();
        this.createdAt = now;
        this.updatedAt = now;
    }

    @PreUpdate
    void onUpdate() {
        this.updatedAt = OffsetDateTime.now();
    }
}
```

- [ ] **Step 3: Créer `ChargerRepository.java`**

```java
package com.accenture.nexcharge.chargers;

import java.util.Optional;
import java.util.UUID;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ChargerRepository extends JpaRepository<Charger, UUID> {
    Optional<Charger> findByOcppId(String ocppId);
}
```

- [ ] **Step 4: Créer `Booking.java`**

```java
package com.accenture.nexcharge.booking;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.OffsetDateTime;
import java.util.UUID;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

@Entity
@Table(name = "bookings")
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Booking {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(name = "user_id", nullable = false)
    private UUID userId;

    @Column(name = "charger_id", nullable = false)
    private UUID chargerId;

    @Column(name = "slot_start", nullable = false)
    private OffsetDateTime slotStart;

    @Column(name = "slot_end", nullable = false)
    private OffsetDateTime slotEnd;

    @Enumerated(EnumType.STRING)
    @JdbcTypeCode(SqlTypes.NAMED_ENUM)
    @Column(nullable = false, columnDefinition = "booking_status")
    private BookingStatus status = BookingStatus.RESERVED;

    @Column(name = "predicted_demand")
    private BigDecimal predictedDemand;

    @Column(name = "fairness_score")
    private BigDecimal fairnessScore;

    @Column(name = "released_reason")
    private String releasedReason;

    @Column(name = "created_at", nullable = false, updatable = false)
    private OffsetDateTime createdAt;

    @Column(name = "updated_at", nullable = false)
    private OffsetDateTime updatedAt;

    @PrePersist
    void onCreate() {
        OffsetDateTime now = OffsetDateTime.now();
        this.createdAt = now;
        this.updatedAt = now;
    }

    @PreUpdate
    void onUpdate() {
        this.updatedAt = OffsetDateTime.now();
    }
}
```

- [ ] **Step 5: Créer `BookingRepository.java`**

```java
package com.accenture.nexcharge.booking;

import java.util.UUID;
import org.springframework.data.jpa.repository.JpaRepository;

public interface BookingRepository extends JpaRepository<Booking, UUID> {}
```

- [ ] **Step 6: Créer `ChargingSession.java`**

```java
package com.accenture.nexcharge.sessions;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.OffsetDateTime;
import java.util.UUID;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

@Entity
@Table(name = "charging_sessions")
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class ChargingSession {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(name = "booking_id", unique = true)
    private UUID bookingId;

    @Column(name = "charger_id", nullable = false)
    private UUID chargerId;

    @Column(name = "user_id", nullable = false)
    private UUID userId;

    @Column(name = "started_at", nullable = false)
    private OffsetDateTime startedAt;

    @Column(name = "ended_at")
    private OffsetDateTime endedAt;

    @Column(name = "kwh_total")
    private BigDecimal kwhTotal;

    @Column(name = "peak_power_kw")
    private BigDecimal peakPowerKw;

    @Column(name = "co2_kg_avoided")
    private BigDecimal co2KgAvoided;

    @Enumerated(EnumType.STRING)
    @JdbcTypeCode(SqlTypes.NAMED_ENUM)
    @Column(nullable = false, columnDefinition = "session_status")
    private SessionStatus status = SessionStatus.IN_PROGRESS;

    @Column(name = "created_at", nullable = false, updatable = false)
    private OffsetDateTime createdAt;

    @PrePersist
    void onCreate() {
        this.createdAt = OffsetDateTime.now();
        if (this.startedAt == null) this.startedAt = this.createdAt;
    }
}
```

- [ ] **Step 7: Créer `MeterValue.java`**

```java
package com.accenture.nexcharge.sessions;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.OffsetDateTime;
import java.util.UUID;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "meter_values")
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class MeterValue {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "session_id", nullable = false)
    private UUID sessionId;

    @Column(name = "recorded_at", nullable = false)
    private OffsetDateTime recordedAt;

    @Column(nullable = false)
    private BigDecimal kwh;

    @Column(name = "power_kw")
    private BigDecimal powerKw;

    private BigDecimal voltage;

    private BigDecimal current;

    @Column(name = "soc_percent")
    private BigDecimal socPercent;
}
```

- [ ] **Step 8: Créer `SessionRepository.java`**

```java
package com.accenture.nexcharge.sessions;

import java.util.UUID;
import org.springframework.data.jpa.repository.JpaRepository;

public interface SessionRepository extends JpaRepository<ChargingSession, UUID> {}
```

- [ ] **Step 9: Écrire le test de validation de schéma `SchemaValidationIT.java`**

```java
package com.accenture.nexcharge;

import org.junit.jupiter.api.Test;

/**
 * Vérifie qu'au démarrage du contexte Spring, Hibernate (ddl-auto=validate)
 * confirme que toutes les entités JPA matchent le schéma Flyway.
 * Si une entité diverge, AbstractIntegrationTest.contextLoads échouera.
 */
class SchemaValidationIT extends AbstractIntegrationTest {

    @Test
    void schema_matches_flyway_migration() {
        // Aucun assert : la simple instanciation du contexte avec ddl-auto=validate suffit.
        // Hibernate lève une SchemaManagementException si une entité ne mappe pas une table.
    }
}
```

- [ ] **Step 10: Lancer tous les tests**

Run: `cd services/core && ./gradlew test`
Expected: BUILD SUCCESSFUL, 5 tests passed (contextLoads + 3 users + 1 schema validation).

- [ ] **Step 11: Commit**

```bash
git add services/core/src/main/java/com/accenture/nexcharge/{chargers,booking,sessions} services/core/src/test/java/com/accenture/nexcharge/SchemaValidationIT.java
git commit -m "feat(core): JPA entities for Charger, Booking, ChargingSession, MeterValue"
```

---

### Task 7: AI service Python — bootstrap FastAPI avec uv

**Files:**
- Create: `apps/ai/pyproject.toml`
- Create: `apps/ai/.env.example`
- Create: `apps/ai/Dockerfile`
- Create: `apps/ai/src/nexcharge_ai/__init__.py`
- Create: `apps/ai/src/nexcharge_ai/config.py`
- Create: `apps/ai/src/nexcharge_ai/main.py`
- Create: `apps/ai/src/nexcharge_ai/narrate/__init__.py`
- Create: `apps/ai/tests/__init__.py`
- Create: `apps/ai/tests/test_health.py`

- [ ] **Step 1: Créer `apps/ai/pyproject.toml`**

```toml
[project]
name = "nexcharge-ai"
version = "0.1.0"
description = "NEXCharge AI service"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "pydantic>=2.9",
    "pydantic-settings>=2.5",
    "httpx>=0.27",
]

[dependency-groups]
dev = [
    "pytest>=8",
    "pytest-asyncio>=0.24",
    "ruff>=0.6",
    "mypy>=1.11",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "W", "I", "B", "UP"]

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"

[tool.mypy]
strict = true
```

- [ ] **Step 2: Créer `apps/ai/.env.example`**

```dotenv
NEXCHARGE_AI_HOST=0.0.0.0
NEXCHARGE_AI_PORT=8000
NEXCHARGE_AI_LOG_LEVEL=INFO
ANTHROPIC_API_KEY=
```

- [ ] **Step 3: Créer `apps/ai/src/nexcharge_ai/__init__.py`**

```python
__version__ = "0.1.0"
```

- [ ] **Step 4: Créer `apps/ai/src/nexcharge_ai/config.py`**

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="NEXCHARGE_AI_",
        env_file=".env",
        extra="ignore",
    )

    host: str = "0.0.0.0"
    port: int = 8000
    log_level: str = "INFO"


def get_settings() -> Settings:
    return Settings()
```

- [ ] **Step 5: Créer `apps/ai/src/nexcharge_ai/main.py`**

```python
from fastapi import FastAPI
from pydantic import BaseModel

from nexcharge_ai import __version__

app = FastAPI(
    title="NEXCharge AI",
    version=__version__,
    description="AI service: forecasting, anomaly detection, fairness scoring, ESG narration",
)


class HealthResponse(BaseModel):
    status: str
    version: str


@app.get("/healthz", response_model=HealthResponse)
async def healthz() -> HealthResponse:
    return HealthResponse(status="ok", version=__version__)


@app.get("/version", response_model=HealthResponse)
async def version() -> HealthResponse:
    return HealthResponse(status="ok", version=__version__)
```

- [ ] **Step 6: Créer `apps/ai/src/nexcharge_ai/narrate/__init__.py`**

```python
"""Placeholder pour la narration ESG (sprint 4)."""
```

- [ ] **Step 7: Créer `apps/ai/tests/__init__.py`**

```python
```

(fichier vide)

- [ ] **Step 8: Écrire le test `apps/ai/tests/test_health.py`**

```python
from fastapi.testclient import TestClient

from nexcharge_ai.main import app

client = TestClient(app)


def test_healthz_returns_ok() -> None:
    response = client.get("/healthz")
    assert response.status_code == 200
    body = response.json()
    assert body["status"] == "ok"
    assert "version" in body


def test_version_returns_ok() -> None:
    response = client.get("/version")
    assert response.status_code == 200
    assert response.json()["status"] == "ok"
```

- [ ] **Step 9: Installer les deps et lancer les tests**

Run:
```bash
cd apps/ai
uv sync
uv run pytest -v
```
Expected: 2 passed.

- [ ] **Step 10: Créer `apps/ai/Dockerfile` (multi-stage avec target dev)**

```dockerfile
# syntax=docker/dockerfile:1.7

FROM python:3.12-slim AS dev
RUN pip install --no-cache-dir uv
WORKDIR /app
COPY pyproject.toml uv.lock* ./
RUN uv sync --frozen || uv sync
COPY src src
ENV PYTHONPATH=/app/src
EXPOSE 8000
CMD ["uv", "run", "uvicorn", "nexcharge_ai.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

FROM python:3.12-slim AS runtime
RUN pip install --no-cache-dir uv
WORKDIR /app
COPY pyproject.toml uv.lock* ./
RUN uv sync --frozen --no-dev || uv sync --no-dev
COPY src src
ENV PYTHONPATH=/app/src
EXPOSE 8000
CMD ["uv", "run", "uvicorn", "nexcharge_ai.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- [ ] **Step 11: Build l'image Docker**

Run: `cd apps/ai && docker build -t nexcharge-ai:dev --target dev .`
Expected: build success.

- [ ] **Step 12: Commit**

```bash
git add apps/ai/
git commit -m "feat(ai): bootstrap FastAPI service with /healthz and /version"
```

---

### Task 8: Backend Java — AiClient + endpoint /healthz/ai (preuve de communication core ↔ ai)

**Files:**
- Create: `services/core/src/main/java/com/accenture/nexcharge/ai/AiClient.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/ai/AiHealthDto.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/health/HealthController.java`
- Create: `services/core/src/main/java/com/accenture/nexcharge/common/config/WebClientConfig.java`
- Create: `services/core/src/test/java/com/accenture/nexcharge/health/HealthControllerIT.java`

- [ ] **Step 1: Créer `WebClientConfig.java`**

```java
package com.accenture.nexcharge.common.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class WebClientConfig {

    @Bean
    public WebClient aiWebClient(
            @Value("${nexcharge.ai.service-url}") String aiServiceUrl) {
        return WebClient.builder().baseUrl(aiServiceUrl).build();
    }
}
```

- [ ] **Step 2: Créer `AiHealthDto.java`**

```java
package com.accenture.nexcharge.ai;

public record AiHealthDto(String status, String version) {}
```

- [ ] **Step 3: Créer `AiClient.java`**

```java
package com.accenture.nexcharge.ai;

import java.time.Duration;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;

@Component
public class AiClient {

    private final WebClient aiWebClient;

    public AiClient(WebClient aiWebClient) {
        this.aiWebClient = aiWebClient;
    }

    public AiHealthDto health() {
        return aiWebClient
                .get()
                .uri("/healthz")
                .retrieve()
                .bodyToMono(AiHealthDto.class)
                .timeout(Duration.ofSeconds(5))
                .block();
    }
}
```

- [ ] **Step 4: Créer `HealthController.java`**

```java
package com.accenture.nexcharge.health;

import com.accenture.nexcharge.ai.AiClient;
import com.accenture.nexcharge.ai.AiHealthDto;
import java.util.Map;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/healthz")
public class HealthController {

    private final AiClient aiClient;

    public HealthController(AiClient aiClient) {
        this.aiClient = aiClient;
    }

    @GetMapping
    public Map<String, String> core() {
        return Map.of("status", "ok", "service", "nexcharge-core");
    }

    @GetMapping("/ai")
    public ResponseEntity<Map<String, Object>> ai() {
        try {
            AiHealthDto ai = aiClient.health();
            return ResponseEntity.ok(Map.of(
                    "status", "ok",
                    "ai_status", ai.status(),
                    "ai_version", ai.version()));
        } catch (Exception e) {
            return ResponseEntity.status(503).body(Map.of(
                    "status", "degraded",
                    "error", e.getMessage()));
        }
    }
}
```

- [ ] **Step 5: Écrire le test `HealthControllerIT.java`**

```java
package com.accenture.nexcharge.health;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import com.accenture.nexcharge.AbstractIntegrationTest;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.test.web.servlet.MockMvc;

@AutoConfigureMockMvc
class HealthControllerIT extends AbstractIntegrationTest {

    @Autowired MockMvc mockMvc;

    @Test
    void healthz_returns_ok_unauthenticated() throws Exception {
        mockMvc.perform(get("/healthz"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status").value("ok"))
                .andExpect(jsonPath("$.service").value("nexcharge-core"));
    }

    @Test
    void healthz_ai_returns_degraded_when_ai_unreachable() throws Exception {
        // Le service AI n'est pas démarré dans les tests d'intégration backend.
        // On vérifie que l'endpoint répond et indique le degraded status proprement.
        mockMvc.perform(get("/healthz/ai"))
                .andExpect(status().isServiceUnavailable())
                .andExpect(jsonPath("$.status").value("degraded"));
    }
}
```

- [ ] **Step 6: Lancer tous les tests**

Run: `cd services/core && ./gradlew test`
Expected: BUILD SUCCESSFUL, 7 tests passed.

- [ ] **Step 7: Vérifier la communication réelle core → ai (test d'intégration manuel)**

Run dans 2 terminaux :
```bash
# Terminal 1
cd apps/ai && uv run uvicorn nexcharge_ai.main:app --port 8000

# Terminal 2
cd services/core
NEXCHARGE_AI_SERVICE_URL=http://localhost:8000 \
SPRING_PROFILES_ACTIVE=dev \
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/nexcharge \
SPRING_DATASOURCE_USERNAME=nexcharge \
SPRING_DATASOURCE_PASSWORD=changeme-in-prod \
SPRING_DATA_REDIS_HOST=localhost \
SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI=http://localhost:8080/realms/nexcharge \
./gradlew bootRun
```

(Postgres + Keycloak doivent être démarrés via `make up` au préalable.)

Puis :
```bash
curl http://localhost:8090/healthz
# Expected: {"status":"ok","service":"nexcharge-core"}

curl http://localhost:8090/healthz/ai
# Expected: {"status":"ok","ai_status":"ok","ai_version":"0.1.0"}
```

Si le second appel retourne `degraded`, vérifier que le service AI tourne bien sur 8000.

- [ ] **Step 8: Commit**

```bash
git add services/core/src/main services/core/src/test/java/com/accenture/nexcharge/health
git commit -m "feat(core): AiClient + /healthz and /healthz/ai endpoints"
```

---

### Task 9: Frontend Next.js — bootstrap PWA + Auth.js OIDC

**Files:**
- Create: `apps/web/package.json`
- Create: `apps/web/tsconfig.json`
- Create: `apps/web/next.config.mjs`
- Create: `apps/web/tailwind.config.ts`
- Create: `apps/web/postcss.config.mjs`
- Create: `apps/web/Dockerfile`
- Create: `apps/web/.env.example`
- Create: `apps/web/src/app/layout.tsx`
- Create: `apps/web/src/app/page.tsx`
- Create: `apps/web/src/app/globals.css`
- Create: `apps/web/src/lib/auth.ts`
- Create: `apps/web/src/app/api/auth/[...nextauth]/route.ts`
- Create: `apps/web/src/app/(auth)/signin/page.tsx`
- Create: `apps/web/src/app/dashboard/page.tsx`
- Create: `apps/web/src/components/sign-in-button.tsx`
- Create: `apps/web/src/lib/api-client.ts`
- Create: `apps/web/src/middleware.ts`

- [ ] **Step 1: Créer `apps/web/package.json`**

```json
{
  "name": "@nexcharge/web",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev -p 3000",
    "build": "next build",
    "start": "next start -p 3000",
    "lint": "next lint",
    "test": "echo 'no tests yet'"
  },
  "dependencies": {
    "next": "15.0.3",
    "react": "19.0.0-rc-66855b96-20241106",
    "react-dom": "19.0.0-rc-66855b96-20241106",
    "next-auth": "5.0.0-beta.25",
    "@auth/core": "0.37.2"
  },
  "devDependencies": {
    "@types/node": "22.9.0",
    "@types/react": "18.3.12",
    "@types/react-dom": "18.3.1",
    "autoprefixer": "10.4.20",
    "eslint": "9.14.0",
    "eslint-config-next": "15.0.3",
    "postcss": "8.4.49",
    "tailwindcss": "3.4.14",
    "typescript": "5.6.3"
  }
}
```

- [ ] **Step 2: Créer `apps/web/tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

- [ ] **Step 3: Créer `apps/web/next.config.mjs`**

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  output: "standalone"
};

export default nextConfig;
```

- [ ] **Step 4: Créer `apps/web/tailwind.config.ts`**

```typescript
import type { Config } from "tailwindcss";

const config: Config = {
  content: ["./src/**/*.{js,ts,jsx,tsx,mdx}"],
  theme: {
    extend: {
      colors: {
        nex: {
          purple: "#5B2EFF",
          dark: "#0F0E17"
        }
      }
    }
  },
  plugins: []
};

export default config;
```

- [ ] **Step 5: Créer `apps/web/postcss.config.mjs`**

```javascript
export default {
  plugins: { tailwindcss: {}, autoprefixer: {} }
};
```

- [ ] **Step 6: Créer `apps/web/.env.example`**

```dotenv
# Auth.js
AUTH_SECRET=changeme-generate-with-openssl-rand-base64-32
NEXTAUTH_URL=http://localhost
AUTH_TRUST_HOST=true

# OIDC provider (Keycloak en dev, Entra en démo)
OIDC_ISSUER_URL=http://keycloak:8080/realms/nexcharge
OIDC_CLIENT_ID=nexcharge-web
OIDC_CLIENT_SECRET=changeme-in-prod

# Backend
NEXCHARGE_API_URL=http://core:8090
```

- [ ] **Step 7: Créer `apps/web/src/lib/auth.ts`**

```typescript
import NextAuth from "next-auth";

export const { handlers, signIn, signOut, auth } = NextAuth({
  trustHost: true,
  providers: [
    {
      id: "keycloak",
      name: "Keycloak",
      type: "oidc",
      issuer: process.env.OIDC_ISSUER_URL!,
      clientId: process.env.OIDC_CLIENT_ID!,
      clientSecret: process.env.OIDC_CLIENT_SECRET!,
      authorization: { params: { scope: "openid email profile" } }
    }
  ],
  callbacks: {
    async jwt({ token, account }) {
      if (account?.access_token) {
        token.accessToken = account.access_token;
      }
      return token;
    },
    async session({ session, token }) {
      (session as { accessToken?: string }).accessToken = token.accessToken as string | undefined;
      return session;
    }
  },
  session: { strategy: "jwt" }
});
```

- [ ] **Step 8: Créer `apps/web/src/app/api/auth/[...nextauth]/route.ts`**

```typescript
import { handlers } from "@/lib/auth";

export const { GET, POST } = handlers;
```

- [ ] **Step 9: Créer `apps/web/src/middleware.ts`**

```typescript
import { auth } from "@/lib/auth";
import { NextResponse } from "next/server";

export default auth((req) => {
  const isAuthPage = req.nextUrl.pathname.startsWith("/signin");
  const isApiAuth = req.nextUrl.pathname.startsWith("/api/auth");
  const isPublic = req.nextUrl.pathname === "/" || isAuthPage || isApiAuth;

  if (!req.auth && !isPublic) {
    return NextResponse.redirect(new URL("/signin", req.url));
  }
  return NextResponse.next();
});

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"]
};
```

- [ ] **Step 10: Créer `apps/web/src/app/globals.css`**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  color-scheme: light dark;
}

body {
  background: white;
  color: #0F0E17;
}
```

- [ ] **Step 11: Créer `apps/web/src/app/layout.tsx`**

```typescript
import "./globals.css";

export const metadata = {
  title: "NEXCharge",
  description: "Equitable EV charging at NEX Tower & NEXTERACOM"
};

export default function RootLayout({
  children
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className="min-h-screen">{children}</body>
    </html>
  );
}
```

- [ ] **Step 12: Créer `apps/web/src/components/sign-in-button.tsx`**

```typescript
import { signIn, signOut, auth } from "@/lib/auth";

export async function SignInButton() {
  const session = await auth();
  if (session) {
    return (
      <form
        action={async () => {
          "use server";
          await signOut({ redirectTo: "/" });
        }}
      >
        <button
          type="submit"
          className="rounded bg-nex-purple px-4 py-2 text-white"
        >
          Sign out ({session.user?.email})
        </button>
      </form>
    );
  }
  return (
    <form
      action={async () => {
        "use server";
        await signIn("keycloak", { redirectTo: "/dashboard" });
      }}
    >
      <button
        type="submit"
        className="rounded bg-nex-purple px-4 py-2 text-white"
      >
        Sign in
      </button>
    </form>
  );
}
```

- [ ] **Step 13: Créer `apps/web/src/app/page.tsx` (landing)**

```typescript
import { SignInButton } from "@/components/sign-in-button";

export default function HomePage() {
  return (
    <main className="mx-auto flex max-w-2xl flex-col gap-6 p-8">
      <h1 className="text-4xl font-bold">NEXCharge</h1>
      <p className="text-lg">
        Equitable, transparent and predictive EV charging at NEX Tower & NEXTERACOM.
      </p>
      <SignInButton />
    </main>
  );
}
```

- [ ] **Step 14: Créer `apps/web/src/app/(auth)/signin/page.tsx`**

```typescript
import { SignInButton } from "@/components/sign-in-button";

export default function SignInPage() {
  return (
    <main className="mx-auto flex max-w-md flex-col gap-6 p-8">
      <h1 className="text-2xl font-semibold">Sign in to NEXCharge</h1>
      <SignInButton />
    </main>
  );
}
```

- [ ] **Step 15: Créer `apps/web/src/lib/api-client.ts`**

```typescript
import { auth } from "@/lib/auth";

export type UserDto = {
  id: string;
  email: string;
  displayName: string;
  role: "DRIVER" | "FACILITY_MANAGER" | "SUSTAINABILITY_OFFICER" | "ADMIN";
  fairShareKwh: number | null;
};

export async function fetchMe(): Promise<UserDto> {
  const session = await auth();
  const token = (session as { accessToken?: string } | null)?.accessToken;
  if (!token) {
    throw new Error("Not authenticated");
  }
  const res = await fetch(`${process.env.NEXCHARGE_API_URL}/api/users/me`, {
    headers: { Authorization: `Bearer ${token}` },
    cache: "no-store"
  });
  if (!res.ok) {
    throw new Error(`API ${res.status}: ${await res.text()}`);
  }
  return res.json();
}
```

- [ ] **Step 16: Créer `apps/web/src/app/dashboard/page.tsx`**

```typescript
import { auth } from "@/lib/auth";
import { fetchMe } from "@/lib/api-client";
import { SignInButton } from "@/components/sign-in-button";
import { redirect } from "next/navigation";

export default async function DashboardPage() {
  const session = await auth();
  if (!session) redirect("/signin");

  let userInfo;
  try {
    userInfo = await fetchMe();
  } catch (e) {
    return (
      <main className="mx-auto max-w-2xl p-8">
        <h1 className="text-2xl font-bold">Dashboard</h1>
        <p className="mt-4 text-red-600">
          Could not reach backend: {(e as Error).message}
        </p>
        <div className="mt-6">
          <SignInButton />
        </div>
      </main>
    );
  }

  return (
    <main className="mx-auto max-w-2xl p-8">
      <h1 className="text-3xl font-bold">Welcome, {userInfo.displayName}</h1>
      <dl className="mt-6 grid grid-cols-2 gap-4">
        <dt className="font-semibold">Email</dt>
        <dd>{userInfo.email}</dd>
        <dt className="font-semibold">Role</dt>
        <dd>{userInfo.role}</dd>
      </dl>
      <div className="mt-6">
        <SignInButton />
      </div>
    </main>
  );
}
```

- [ ] **Step 17: Installer les deps front**

Run: `cd apps/web && pnpm install`
Expected: install success.

- [ ] **Step 18: Lancer le build**

Run: `cd apps/web && pnpm build`
Expected: build success, output `.next/standalone`.

- [ ] **Step 19: Créer `apps/web/Dockerfile` (multi-stage avec target dev)**

```dockerfile
# syntax=docker/dockerfile:1.7

FROM node:20-alpine AS base
RUN corepack enable && corepack prepare pnpm@9.12.0 --activate
WORKDIR /app

FROM base AS dev
COPY package.json pnpm-lock.yaml* ./
RUN pnpm install
COPY . .
EXPOSE 3000
CMD ["pnpm", "dev"]

FROM base AS deps
COPY package.json pnpm-lock.yaml* ./
RUN pnpm install --frozen-lockfile

FROM base AS build
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build

FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

- [ ] **Step 20: Build Docker**

Run: `cd apps/web && docker build -t nexcharge-web:dev --target dev .`
Expected: build success.

- [ ] **Step 21: Commit**

```bash
git add apps/web/
git commit -m "feat(web): bootstrap Next.js 15 PWA with Auth.js OIDC + dashboard"
```

---

### Task 10: End-to-end smoke — la stack démarre, login fonctionne, dashboard appelle le backend

**Files:**
- Create: `apps/web/tests/auth-flow.spec.ts`
- Create: `apps/web/playwright.config.ts`
- Modify: `apps/web/package.json` (ajouter scripts/deps Playwright)

- [ ] **Step 1: Démarrer toute la stack**

Run:
```bash
make up
docker compose -f infra/docker-compose.yml ps
```

Expected: 8 services en `Up`. Patienter ~60s pour que tout soit healthy.

- [ ] **Step 2: Vérifier manuellement les endpoints**

```bash
# Frontend (page landing publique)
curl -L http://localhost
# Expected: HTML avec "NEXCharge"

# Backend healthz (pas d'auth requise)
curl http://localhost/api/healthz
# Expected: {"status":"ok","service":"nexcharge-core"}

# Backend → AI healthz
curl http://localhost/api/healthz/ai
# Expected: {"status":"ok","ai_status":"ok","ai_version":"0.1.0"}

# AI direct
curl http://localhost/ai/healthz
# Expected: {"status":"ok","version":"0.1.0"}
```

- [ ] **Step 3: Test manuel du flow OIDC dans un navigateur**

Ouvrir http://localhost dans un navigateur :
1. Cliquer "Sign in" → redirige vers Keycloak.
2. Se connecter avec `driver` / `driver`.
3. Retour sur `/dashboard` → afficher "Welcome, Demo Driver" + email + role DRIVER.

Si erreur :
- Vérifier que `OIDC_CLIENT_SECRET` est cohérent entre Keycloak realm export et `.env`.
- Vérifier que les `redirectUris` du client Keycloak matchent `http://localhost/api/auth/callback/keycloak`.

- [ ] **Step 4: Installer Playwright dans `apps/web`**

```bash
cd apps/web
pnpm add -D @playwright/test
pnpm exec playwright install chromium
```

- [ ] **Step 5: Créer `apps/web/playwright.config.ts`**

```typescript
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./tests",
  timeout: 60_000,
  use: {
    baseURL: process.env.PLAYWRIGHT_BASE_URL ?? "http://localhost",
    trace: "on-first-retry"
  }
});
```

- [ ] **Step 6: Créer `apps/web/tests/auth-flow.spec.ts`**

```typescript
import { test, expect } from "@playwright/test";

test("landing page shows sign in button", async ({ page }) => {
  await page.goto("/");
  await expect(page.getByText("NEXCharge")).toBeVisible();
  await expect(page.getByRole("button", { name: /sign in/i })).toBeVisible();
});

test("sign in with keycloak driver succeeds and dashboard shows user info", async ({
  page
}) => {
  await page.goto("/");
  await page.getByRole("button", { name: /sign in/i }).click();

  // Keycloak login form
  await page.locator("#username").fill("driver");
  await page.locator("#password").fill("driver");
  await page.locator("#kc-login").click();

  // Back on /dashboard, JIT-provisioned user data from backend
  await expect(page.getByText(/welcome, demo driver/i)).toBeVisible({
    timeout: 15_000
  });
  await expect(page.getByText("driver@accenture.local")).toBeVisible();
  await expect(page.getByText("DRIVER")).toBeVisible();
});
```

- [ ] **Step 7: Ajouter le script Playwright dans `apps/web/package.json`**

Modifier le `scripts` block pour ajouter :
```json
"test:e2e": "playwright test"
```

- [ ] **Step 8: Lancer les tests E2E**

Run:
```bash
cd apps/web
pnpm test:e2e
```

Expected: 2 passed.

Si échec : vérifier que la stack tourne (`make ps`) et que la page Keycloak utilise bien les ID `#username`/`#password`/`#kc-login` (selectors par défaut de Keycloak).

- [ ] **Step 9: Vérifier que les tests Java passent toujours**

Run: `cd services/core && ./gradlew test`
Expected: BUILD SUCCESSFUL.

- [ ] **Step 10: Vérifier que les tests Python passent toujours**

Run: `cd apps/ai && uv run pytest -v`
Expected: 2 passed.

- [ ] **Step 11: Arrêter la stack**

Run: `make down`

- [ ] **Step 12: Commit**

```bash
git add apps/web/playwright.config.ts apps/web/tests/ apps/web/package.json apps/web/pnpm-lock.yaml
git commit -m "test(web): Playwright E2E for landing + Keycloak login + dashboard"
```

---

### Task 11: CI GitHub Actions

**Files:**
- Create: `.github/workflows/ci.yml`

- [ ] **Step 1: Créer `.github/workflows/ci.yml`**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  core:
    name: Backend Java
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
          cache: gradle
      - name: Run tests
        working-directory: services/core
        run: ./gradlew spotlessCheck test
      - name: Build jar
        working-directory: services/core
        run: ./gradlew bootJar -x test

  ai:
    name: AI Python
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - name: Install uv
        run: pip install uv
      - name: Install deps
        working-directory: apps/ai
        run: uv sync
      - name: Lint
        working-directory: apps/ai
        run: uv run ruff check .
      - name: Test
        working-directory: apps/ai
        run: uv run pytest

  web:
    name: Frontend
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9.12.0 }
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
          cache-dependency-path: apps/web/pnpm-lock.yaml
      - name: Install
        working-directory: apps/web
        run: pnpm install --frozen-lockfile
      - name: Lint
        working-directory: apps/web
        run: pnpm lint
      - name: Build
        working-directory: apps/web
        run: pnpm build
        env:
          AUTH_SECRET: ci-dummy-secret-32-chars-min-padding
          OIDC_ISSUER_URL: http://localhost/dummy
          OIDC_CLIENT_ID: dummy
          OIDC_CLIENT_SECRET: dummy
          NEXCHARGE_API_URL: http://localhost/dummy
```

- [ ] **Step 2: Vérifier syntaxe YAML**

Run: `cat .github/workflows/ci.yml | docker run --rm -i mikefarah/yq:latest e -`
Expected: pas d'erreur.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: GitHub Actions for core / ai / web"
```

---

### Task 12: Documentation finale Sprint 1

**Files:**
- Modify: `README.md` (ajouter section "Status Sprint 1")
- Create: `docs/superpowers/plans/2026-05-21-sprint-1-fondations-DONE.md` (snapshot, optionnel)

- [ ] **Step 1: Mettre à jour `README.md`**

Ajouter à la fin du README :

```markdown

## Status

### Sprint 1 — Fondations ✅
- [x] Monorepo bootstrap (pnpm + Gradle + uv)
- [x] docker-compose stack (Postgres, Redis, MinIO, Keycloak, Traefik)
- [x] Backend Java + Spring Boot 3 + schema Postgres complet (Flyway V1)
- [x] AI service FastAPI minimal (`/healthz`)
- [x] Auth OIDC end-to-end (Keycloak local, prêt pour Entra)
- [x] Frontend Next.js 15 PWA + Auth.js
- [x] Communication core ↔ ai validée
- [x] Provisioning JIT du User au premier login
- [x] CI GitHub Actions (3 jobs : core / ai / web)
- [x] Tests E2E Playwright (login flow)

### Demo Sprint 1
1. `make up`
2. Ouvrir http://localhost
3. Sign in avec `driver` / `driver`
4. Voir le dashboard "Welcome, Demo Driver" avec données depuis le backend Java
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: sprint 1 status in README"
```

---

## Self-Review

Spec sections × tasks coverage :

| Spec section | Tasks |
|---|---|
| §1 Vision (incl. baseline guidelines Accenture) | T1 (README), T3b (BusinessProperties câble les valeurs de config alignées guidelines) |
| §2 Architecture (containers, comms) | T1, T2 (docker-compose), T3 (core), T7 (ai), T9 (web), T8 (core↔ai) |
| §3 Modèle de données | T4 (V1 init.sql), T5 (User entity), T6 (autres entités) |
| §4 Modules | différé S2/S3/S4 (Sprint 1 = fondations) |
| §4 Module 7 Issue Reporting | différé S4 (UI + email Workplace) — config câblée en T3b |
| §5 Sécurité OIDC + RBAC | T5 (SecurityConfig + JIT) |
| §5 RBAC complet | différé S2 (Spring Security `@PreAuthorize` ajouté quand BookingController existe) |
| §5 Configuration métier admin-tunable | T3b (BusinessProperties + defaults validés) |
| §6 Tests, monorepo, CI | T1 (monorepo), T5/T6/T8 (Testcontainers), T10 (Playwright), T11 (CI) |
| §6 Sprint 1 demo (login) | T10 |

Aucune section S1 du spec n'est sans task.

Placeholder scan : aucun TBD/TODO dans le plan. Tout code est complet et prêt à coller.

Type consistency :
- `UserDto` (record) défini en T5, utilisé en T9 (`api-client.ts`) — champs alignés (`id`, `email`, `displayName`, `role`, `fairShareKwh`).
- `AiHealthDto` (record) défini en T8, sérialisé via Jackson, désérialisé en `HealthResponse` côté Python (`status`, `version`) — aligné.
- `UserRole` enum cohérent entre Java (T5) et TS (T9) : `DRIVER | FACILITY_MANAGER | SUSTAINABILITY_OFFICER | ADMIN`.
- Migration V1 et entités JPA mappent les mêmes types (`user_role`, `booking_status`, `charger_status`, `session_status`).

Plan auto-cohérent. Prêt pour exécution.

---

## Sprints suivants (à planifier après S1)

À la fin du Sprint 1, écrire les plans suivants au fur et à mesure (un plan = un sprint = ~1 semaine d'implémentation, plus court à écrire et plus facile à suivre que de tout figer maintenant) :

- **Sprint 2** : OCPP server intégré, simulateur OCPP, CRUD bookings + algo fairness rule-based **avec enforcement durée 3h** (`MAX_BOOKING_DURATION_HOURS`), Live Map WebSocket STOMP.
- **Sprint 3** : Service AI (forecasting Prophet, fairness scoring, anomaly detection), explainability UI ("Why this slot?"), reminders & auto-release **dont SoC-based "sufficient charge" alert** (`SUFFICIENT_CHARGE_THRESHOLD_PCT`), PWA offline, push notifications.
- **Sprint 4** : Dashboards facility manager + sustainability officer, rapport ESG mensuel narré (Claude → PDF), bias audit Gini, **Module 7 Issue Reporting** (UI + email vers `WORKPLACE_OPS_EMAIL`), polish UI/UX, tests E2E complets, préparation pitch.
