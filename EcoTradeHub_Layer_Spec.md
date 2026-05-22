# EcoTrade Hub — Couche de Différenciation

> **Contexte** : Ce document décrit la couche de différenciation "EcoTrade Hub" à intégrer par-dessus le MVP du Hackathon 2026 (app de réservation EV / BRD). L'objectif est de transformer un simple système de booking en une plateforme d'intelligence énergétique, en réutilisant les concepts validés lors du GTIC Season 7.

---

## 1. Vue d'ensemble

Le MVP couvre les requirements BR001–BR013 du BRD : booking, OCPP, notifications, RBAC, tracking.

La couche EcoTrade Hub ajoute **trois modules** :

| Module | Ce qu'il fait | Lien avec le GTIC |
|--------|--------------|-------------------|
| **Green Score** | Convertit chaque session de charge en score carbone par employé | Verification & Standardization d'EcoTrade Hub |
| **Predictive Dashboard** | Prévoit la demande et projette les économies CO₂ | Data-Driven Insights & Predictive Analytics |
| **Audit Trail Hashé** | Chaîne de hashes immuable sur les données de consommation | Blockchain-based immutable records |

---

## 2. Module 1 — Green Score

### Concept

Chaque session de charge OCPP génère des données de consommation (kWh, timestamp, durée). On enrichit ces données avec le **facteur d'émission du grid** de Maurice à l'heure de la charge pour calculer le CO₂ évité par rapport à un véhicule thermique équivalent.

### Données d'entrée

- `session.energyConsumed` (kWh) — vient de l'OCPP MeterValues
- `session.startTime` / `session.endTime` — timestamps de la session
- `vehicle.make` / `vehicle.model` — du profil utilisateur (BR005)
- `gridCarbonIntensity` — facteur d'émission en gCO₂/kWh selon l'heure (données statiques ou API si disponible)

### Calcul

```
CO₂ évité (g) = distanceÉquivalente(kWh, efficacitéEV)
                × émissionThermique(gCO₂/km)
                - (kWh × gridCarbonIntensity)

greenScore = Σ CO₂ évité sur toutes les sessions de l'utilisateur
```

**Valeurs de référence pour Maurice** :
- Efficacité EV moyenne : ~6 km/kWh
- Émission véhicule thermique moyen : ~120 gCO₂/km
- Intensité carbone grid Maurice : ~600–800 gCO₂/kWh (mix fossile dominant)
  - Heures creuses (nuit) : ~550 gCO₂/kWh (moins de demand = moins de peakers)
  - Heures de pointe : ~850 gCO₂/kWh

### Schéma de données (Prisma)

```prisma
model GreenScore {
  id          String   @id @default(cuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id])
  sessionId   String   @unique
  session     ChargingSession @relation(fields: [sessionId], references: [id])

  energyKwh           Float
  gridCarbonIntensity Float    // gCO2/kWh au moment de la charge
  co2AvoidedGrams     Float
  greenPoints         Int      // score simplifié pour le leaderboard

  calculatedAt DateTime @default(now())
}
```

### UI

- **Profil utilisateur** : badge circulaire avec le green score cumulé, rang parmi les employés, équivalent visuel ("Vous avez économisé l'équivalent de X arbres plantés")
- **Leaderboard** (optionnel) : classement anonymisé ou par initiales pour encourager la charge aux heures creuses
- **Admin dashboard** : green score agrégé par site (NEX Tower vs NEXTERACOM)

### Implémentation avec Claude Code

```
Prompt suggéré :
"Crée un service `lib/ecotrade/greenScore.ts` qui :
1. Prend en entrée une ChargingSession (energyKwh, startTime, endTime)
   et un Vehicle (make, model)
2. Détermine le gridCarbonIntensity basé sur l'heure (utilise une lookup
   table statique pour les heures creuses/pointe de Maurice)
3. Calcule le CO₂ évité vs un véhicule thermique équivalent
4. Retourne { co2AvoidedGrams, greenPoints }
5. Inclus les tests unitaires"
```

---

## 3. Module 2 — Predictive Dashboard

### Concept

Au lieu de simplement montrer l'historique d'usage (ce que le BRD demande), on ajoute une couche prédictive qui anticipe la demande de charge et projette les économies carbone futures.

### Fonctionnalités

**3a. Prédiction de demande**
- Input : historique des réservations sur les 30 derniers jours
- Output : heatmap de demande prédite par jour de la semaine × créneau horaire
- Méthode : moyenne pondérée glissante (pas besoin de ML complexe pour le hackathon — une bonne heuristique suffit)

**3b. Projection carbone**
- Input : green scores cumulés + tendance de croissance du nombre d'utilisateurs EV
- Output : courbe de projection CO₂ évité sur 3/6/12 mois
- Afficher en parallèle les cibles gouvernementales 2026 pour montrer l'alignement

**3c. Détection de patterns**
- Identifier les heures de sous-utilisation des bornes
- Suggérer des créneaux optimaux aux utilisateurs (quand le grid est le plus "vert")
- Alerter l'admin quand un site approche la saturation récurrente

### Schéma de données

```prisma
model DemandForecast {
  id          String   @id @default(cuid())
  siteId      String
  site        Site     @relation(fields: [siteId], references: [id])
  dayOfWeek   Int      // 0=Lundi ... 6=Dimanche
  hourSlot    Int      // 0-23
  predictedDemand Float // nombre de sessions prédites
  confidence  Float    // 0-1
  generatedAt DateTime @default(now())
}

model CarbonProjection {
  id              String   @id @default(cuid())
  siteId          String?
  month           DateTime // premier jour du mois projeté
  projectedCo2Avoided Float // en kg
  actualCo2Avoided    Float? // rempli a posteriori
  governmentTarget    Float? // cible réglementaire si connue
  generatedAt     DateTime @default(now())
}
```

### UI

- **Heatmap de demande** : grille jour × heure avec code couleur (vert = libre, rouge = saturé, bleu = prédiction)
- **Courbe de projection** : line chart Recharts avec zone de confiance, superposée aux cibles gouvernementales
- **Recommendations panel** : cards avec suggestions ("Les bornes NEXTERACOM sont sous-utilisées le mardi matin — proposer des incitations ?")

### Implémentation avec Claude Code

```
Prompt suggéré :
"Crée un service `lib/ecotrade/forecast.ts` qui :
1. Prend l'historique des ChargingSessions des 30 derniers jours
2. Calcule la demande moyenne par (dayOfWeek, hourSlot, siteId)
   avec pondération exponentielle (sessions récentes = plus de poids)
3. Retourne un tableau de DemandForecast
4. Inclus une fonction projectCarbonSavings(months: number) qui
   extrapole les green scores cumulés sur N mois à venir

Ensuite crée un composant React `components/ecotrade/DemandHeatmap.tsx`
qui affiche la heatmap avec Recharts ou une grille CSS custom.
Et un composant `components/ecotrade/CarbonProjectionChart.tsx`
avec un line chart montrant projection vs cibles."
```

---

## 4. Module 3 — Audit Trail Hashé

### Concept

Chaque session de charge terminée produit un enregistrement immuable. On chaîne les hashes pour créer un mini-ledger vérifiable — inspiré directement de l'approche blockchain d'EcoTrade Hub, mais sans la complexité d'une vraie blockchain.

### Fonctionnement

```
Pour chaque session terminée :
1. Sérialiser les données clés : userId, sessionId, energyKwh, startTime, endTime, siteId
2. Récupérer le hash du dernier enregistrement (previousHash)
3. Calculer : hash = SHA-256(previousHash + données sérialisées + timestamp)
4. Stocker le record avec le hash
```

Cela garantit que :
- Toute modification d'un record passé casse la chaîne
- L'audit trail est vérifiable de bout en bout
- Les données de consommation soumises au gouvernement sont prouvablement intègres

### Schéma de données

```prisma
model AuditRecord {
  id            String   @id @default(cuid())
  sessionId     String   @unique
  session       ChargingSession @relation(fields: [sessionId], references: [id])

  dataPayload   String   // JSON sérialisé des données de la session
  previousHash  String   // hash du record précédent ("genesis" pour le premier)
  hash          String   @unique // SHA-256 du record courant
  timestamp     DateTime @default(now())

  @@index([timestamp])
}
```

### API de vérification

```typescript
// GET /api/ecotrade/audit/verify
// Parcourt la chaîne et vérifie l'intégrité

interface VerificationResult {
  totalRecords: number;
  verified: number;
  broken: boolean;
  brokenAtIndex?: number; // si la chaîne est cassée
  lastVerifiedHash: string;
}
```

### UI

- **Admin panel** : section "Audit & Compliance" avec un indicateur vert/rouge de l'intégrité de la chaîne
- **Bouton "Vérifier l'intégrité"** qui parcourt la chaîne et affiche le résultat
- **Export** : générer un rapport d'audit PDF-ready pour la soumission réglementaire
- **Visualisation** : timeline des records avec le hash tronqué visible, style blockchain explorer simplifié

### Implémentation avec Claude Code

```
Prompt suggéré :
"Crée un service `lib/ecotrade/auditTrail.ts` qui :
1. Expose une fonction `createAuditRecord(session: ChargingSession)`
   qui sérialise les données, récupère le previousHash du dernier record,
   calcule le SHA-256 de la concaténation, et sauvegarde en DB via Prisma
2. Expose une fonction `verifyChain()` qui parcourt tous les AuditRecords
   chronologiquement et vérifie que chaque hash correspond
3. Retourne un VerificationResult
4. Utilise la lib crypto native de Node.js
5. Inclus les tests"
```

---

## 5. Données de démo (Seed)

Pour que la démo soit visuellement impressionnante, il faut des données réalistes sur ~30 jours.

```
Prompt pour Claude Code :
"Crée un script `prisma/seed-ecotrade.ts` qui génère :
- 50 utilisateurs avec des véhicules variés (Tesla Model 3, Nissan Leaf,
  MG4, BYD Atto 3, Hyundai Ioniq 5)
- ~500 sessions de charge réparties sur 30 jours avec des patterns
  réalistes (pic le matin 8h-10h, pic en début d'après-midi 13h-14h,
  creux le weekend)
- Les GreenScores calculés pour chaque session
- Les AuditRecords chaînés pour chaque session
- Les DemandForecasts pré-calculés
- Les CarbonProjections sur 6 mois
Répartis entre NEX Tower (60%) et NEXTERACOM (40%)."
```

---

## 6. Structure de fichiers

```
lib/ecotrade/
├── greenScore.ts          # Calcul du green score par session
├── greenScore.test.ts     # Tests unitaires
├── forecast.ts            # Prédiction de demande + projection carbone
├── forecast.test.ts
├── auditTrail.ts          # Création et vérification de la chaîne de hashes
├── auditTrail.test.ts
└── constants.ts           # Facteurs d'émission, config Maurice

components/ecotrade/
├── GreenScoreBadge.tsx    # Badge circulaire sur le profil user
├── GreenLeaderboard.tsx   # Classement optionnel
├── DemandHeatmap.tsx      # Heatmap jour × heure
├── CarbonProjectionChart.tsx  # Line chart projection vs cibles
├── AuditVerifier.tsx      # Panel de vérification admin
└── AuditTimeline.tsx      # Visualisation de la chaîne

app/api/ecotrade/
├── green-score/route.ts   # GET score d'un user, POST recalcul
├── forecast/route.ts      # GET prédictions, POST régénérer
└── audit/
    ├── route.ts           # GET liste des records
    └── verify/route.ts    # GET vérification de la chaîne
```

---

## 7. Pitch narratif (pour la démo)

> "Le BRD nous demandait un système de booking avec du tracking. On l'a construit. Mais on est allés plus loin.
>
> Chaque kWh chargé sur nos bornes est maintenant converti en un green score traçable — on sait exactement combien de CO₂ chaque employé évite, en fonction du mix énergétique du grid au moment de sa charge.
>
> Notre dashboard ne montre pas juste l'historique — il prédit la demande, identifie les créneaux sous-utilisés, et projette nos économies carbone sur les 12 prochains mois, alignées avec les cibles gouvernementales 2026.
>
> Et chaque session est scellée dans un audit trail hashé, inspiré de notre travail sur EcoTrade Hub au GTIC. Les données de consommation qu'on soumettra au régulateur sont prouvablement intègres — pas juste un log dans une base de données, mais une chaîne vérifiable.
>
> Le charger est le point de départ. La plateforme d'intelligence énergétique est la destination."

---

## 8. Priorité d'implémentation

Si le temps est court pendant le hackathon :

1. **Green Score** (impact visuel fort, facile à implémenter, ~2-3h)
2. **Audit Trail** (impressionne les juges sur la compliance, ~2h)
3. **Predictive Dashboard** (le plus complexe côté UI, ~3-4h)

Le Green Score seul suffit déjà à différencier. Les trois ensemble, c'est la victoire.
