# SwimSpot — Spécifications Techniques et Fonctionnelles

**Version:** 5.0  
**Date:** Janvier 2026  
**Application:** PWA de navigation pour nageurs en eau libre  

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Architecture technique](#2-architecture-technique)
3. [Data Schema](#3-data-schema)
4. [Règles métier](#4-règles-métier)
5. [Écrans et interfaces](#5-écrans-et-interfaces)
6. [Composants UI](#6-composants-ui)
7. [Fonctionnalités détaillées](#7-fonctionnalités-détaillées)
8. [API et intégrations](#8-api-et-intégrations)
9. [Variables et constantes](#9-variables-et-constantes)
10. [Guide d'implémentation](#10-guide-dimplémentation)

---

## 1. Vue d'ensemble

### 1.1 Description produit

SwimSpot est une application mobile (PWA) communautaire permettant aux nageurs en eau libre de :
- Découvrir des spots de baignade avec conditions en temps réel
- Consulter et contribuer des informations locales
- Créer et partager des parcours de nage
- Signaler des dangers

### 1.2 Cible utilisateur

- Nageurs en eau libre (débutants à confirmés)
- ~400 000 pratiquants en France
- Marchés cibles : France, Espagne, Italie

### 1.3 Stack technique recommandé

| Couche | Technologie |
|--------|-------------|
| Frontend | React Native / Flutter / PWA |
| Backend | Node.js / Python (FastAPI) |
| Base de données | PostgreSQL + PostGIS |
| Cache | Redis |
| Cartographie | Mapbox GL JS / Leaflet |
| Météo | Météo-France API, SHOM |
| Auth | Firebase Auth / Auth0 |
| Storage | AWS S3 / Cloudflare R2 |

---

## 2. Architecture technique

### 2.1 Architecture globale

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT (PWA)                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │ Explore │  │   Add   │  │ Profile │  │  Spot   │        │
│  │  Screen │  │  Screen │  │  Screen │  │  Detail │        │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘        │
│                      │                                      │
│              ┌───────┴───────┐                              │
│              │   State Mgmt  │                              │
│              │ (Redux/Zustand)│                              │
│              └───────────────┘                              │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                       API GATEWAY                           │
├─────────────────────────────────────────────────────────────┤
│  /spots  │  /routes  │  /infos  │  /users  │  /conditions  │
└─────────────────────────────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │  PostgreSQL │  │    Redis    │  │   External  │
   │  + PostGIS  │  │   (Cache)   │  │    APIs     │
   └─────────────┘  └─────────────┘  └─────────────┘
                                           │
                    ┌──────────────────────┼──────────────┐
                    ▼                      ▼              ▼
             ┌───────────┐          ┌───────────┐  ┌───────────┐
             │Météo-France│          │   SHOM    │  │  Mapbox   │
             └───────────┘          └───────────┘  └───────────┘
```

### 2.2 Navigation (écrans)

```
┌─────────────────────────────────────────┐
│              NAVIGATION                 │
├─────────────────────────────────────────┤
│                                         │
│  screen-explore ←→ screen-spot          │
│       │               │                 │
│       │               └→ screen-route   │
│       │                                 │
│       ↓                                 │
│  screen-create (Add)                    │
│       │                                 │
│       ↓                                 │
│  screen-profile                         │
│                                         │
└─────────────────────────────────────────┘
```

---

## 3. Data Schema

### 3.1 Entité : Spot

```typescript
interface Spot {
  id: string;                    // Unique identifier (slug ou UUID)
  name: string;                  // Nom affiché
  region: string;                // Région/ville
  lat: number;                   // Latitude GPS (WGS84)
  lng: number;                   // Longitude GPS (WGS84)
  type?: SpotType;               // Type de spot
  access?: AccessLevel;          // Niveau d'accès
  description?: string;          // Description
  createdBy?: string;            // User ID créateur
  createdAt?: Date;              // Date création
  conditions: Condition[];       // Prévisions (6 créneaux)
}

type SpotType = 'plage' | 'crique' | 'port' | 'lac' | 'riviere';
type AccessLevel = 'facile' | 'modere' | 'difficile';
```

### 3.2 Entité : Condition

```typescript
interface Condition {
  status: ConditionStatus;       // État global
  temp: number;                  // Température eau (°C)
  wind: number;                  // Vitesse vent (km/h)
  waves: number;                 // Hauteur vagues (m)
  tide?: TideStatus;             // État marée
  jellyfish?: RiskLevel;         // Risque méduses
  currents?: CurrentLevel;       // Intensité courants
  timestamp?: Date;              // Horodatage prévision
}

type ConditionStatus = 'favorable' | 'technique' | 'exigeant';
type TideStatus = 'montante' | 'descendante' | 'etale_haute' | 'etale_basse';
type RiskLevel = 'faible' | 'possible' | 'probable' | 'certain';
type CurrentLevel = 'faibles' | 'moderes' | 'forts';
```

### 3.3 Entité : Route (Parcours)

```typescript
interface Route {
  id: string;                    // Unique identifier
  spotId: string;                // Spot parent
  name: string;                  // Nom du parcours
  distance: number;              // Distance en km
  level: RouteLevel;             // Niveau requis
  points: LatLng[];              // Points GPS (toujours boucle)
  description?: string;          // Description
  swims: number;                 // Nombre de nages comptabilisées
  createdBy: string;             // User ID créateur
  createdAt: Date;               // Date création
}

type RouteLevel = 'debutant' | 'intermediaire' | 'avance';

interface LatLng {
  lat: number;
  lng: number;
}
```

### 3.4 Entité : LocalInfo

```typescript
interface LocalInfo {
  id: string;                    // Unique identifier
  spotId: string;                // Spot parent
  type: InfoType;                // Type d'info
  title: string;                 // Titre
  description: string;           // Description
  verifications: number;         // Nombre de vérifications
  createdBy: string;             // User ID créateur
  createdAt: Date;               // Date création
  lastVerifiedAt?: Date;         // Dernière vérification
}

type InfoType = 'tip' | 'warning' | 'danger';
```

### 3.5 Entité : User

```typescript
interface User {
  id: string;                    // Unique identifier
  username: string;              // @pseudo
  email: string;                 // Email
  karma: number;                 // Points karma
  region?: string;               // Région
  createdAt: Date;               // Date inscription
  stats: UserStats;              // Statistiques
}

interface UserStats {
  spotsCreated: number;          // Spots créés
  routesCreated: number;         // Parcours créés
  infosCreated: number;          // Infos créées
  verificationsGiven: number;    // Vérifications données
}
```

### 3.6 Schéma relationnel

```
┌─────────────┐       ┌─────────────┐
│    USERS    │       │    SPOTS    │
├─────────────┤       ├─────────────┤
│ id (PK)     │       │ id (PK)     │
│ username    │◄──────│ created_by  │
│ email       │       │ name        │
│ karma       │       │ region      │
│ region      │       │ lat         │
│ created_at  │       │ lng         │
└─────────────┘       │ type        │
       │              │ access      │
       │              │ description │
       │              │ created_at  │
       │              └──────┬──────┘
       │                     │
       │              ┌──────┴──────┐
       │              │             │
       │      ┌───────▼───┐  ┌──────▼──────┐
       │      │  ROUTES   │  │ LOCAL_INFOS │
       │      ├───────────┤  ├─────────────┤
       │      │ id (PK)   │  │ id (PK)     │
       └──────│ spot_id   │  │ spot_id     │
              │ created_by│  │ created_by  │──►
              │ name      │  │ type        │
              │ distance  │  │ title       │
              │ level     │  │ description │
              │ points[]  │  │ verif_count │
              │ swims     │  │ created_at  │
              │ created_at│  └─────────────┘
              └───────────┘

┌─────────────────┐
│   CONDITIONS    │
├─────────────────┤
│ id (PK)         │
│ spot_id (FK)    │
│ timestamp       │
│ status          │
│ temp            │
│ wind            │
│ waves           │
│ tide            │
│ jellyfish       │
│ currents        │
└─────────────────┘
```

---

## 4. Règles métier

### 4.1 Système Karma

| Action | Points | Condition |
|--------|--------|-----------|
| Créer un spot | +20 ⭐ | Karma ≥ 50 requis |
| Créer un parcours | +25 ⭐ | Spot existant requis |
| Ajouter une info | +10 ⭐ | Spot existant requis |
| Signaler un danger | +15 ⭐ | Spot existant requis |
| Vérifier une info | +5 ⭐ | Aucune |

### 4.2 Conditions requises pour créer

| Type de contenu | Karma minimum | Prérequis |
|-----------------|---------------|-----------|
| Nouveau spot | **50 ⭐** | Clic sur carte (position GPS) |
| Info locale | 0 | Spot existant sélectionné |
| Danger | 0 | Spot existant sélectionné |
| Parcours | 0 | Spot existant sélectionné |

### 4.3 Classification des conditions

#### Statut global

| Statut | Couleur | Description |
|--------|---------|-------------|
| `favorable` | 🟢 Vert (#16a34a) | Idéal pour tous niveaux |
| `technique` | 🟡 Orange (#f59e0b) | Nageurs expérimentés recommandés |
| `exigeant` | 🔴 Rouge (#ef4444) | Fortement déconseillé |

#### Règles de calcul du statut

```javascript
// Logique de calcul (exemple)
function calculateStatus(wind, waves, temp) {
  if (wind > 25 || waves > 1.0) return 'exigeant';
  if (wind > 18 || waves > 0.6 || temp < 14) return 'technique';
  return 'favorable';
}
```

#### Seuils d'alerte par paramètre

| Paramètre | Normal | Warning (⚠️) | Danger (🔴) |
|-----------|--------|--------------|-------------|
| Température eau | ≥ 15°C | < 15°C | < 12°C |
| Vent | ≤ 15 km/h | 15-20 km/h | > 20 km/h |
| Vagues | ≤ 0.5 m | 0.5-0.8 m | > 0.8 m |
| Méduses | Faible | Possible | Probable |
| Courants | Faibles | Modérés | Forts |

### 4.4 Règles parcours

1. **Boucle obligatoire** : Tous les parcours doivent revenir au point de départ
2. **Minimum 2 points** : Un parcours doit avoir au moins 2 waypoints
3. **Sur l'eau uniquement** : Les points doivent être tracés sur l'eau, pas sur terre
4. **Nom obligatoire** : Le parcours doit avoir un nom

### 4.5 Types d'informations locales

| Type | Icône | Usage |
|------|-------|-------|
| `tip` (Conseil) | 💡 | Bon à savoir, astuces |
| `warning` (Attention) | ⚠️ | Prudence recommandée |
| `danger` | 🚨 | Risque sérieux, danger immédiat |

### 4.6 Créneaux horaires (prévisions)

```javascript
const timeOffsets = [0, 2, 4, 6, 8, 24]; // heures depuis maintenant

// Index 0: Maintenant
// Index 1: +2h
// Index 2: +4h
// Index 3: +6h
// Index 4: +8h
// Index 5: +24h (demain)
```

---

## 5. Écrans et interfaces

### 5.1 Écran Explore (Carte)

```
┌─────────────────────────────────────┐
│ ┌─────┐                    ┌─────┐  │
│ │  ≡  │  SwimSpot          │  ⚙  │  │
│ └─────┘                    └─────┘  │
├─────────────────────────────────────┤
│ [🔍 Rechercher un spot...]          │
├─────────────────────────────────────┤
│ [Marseille ▼]  [Wimereux ▼]         │
├─────────────────────────────────────┤
│                                     │
│              [CARTE]                │
│                                     │
│    🟢 Catalans    🟡 Pointe Rouge   │
│           🟢 Prado                  │
│                                     │
├─────────────────────────────────────┤
│ Mer 15 · 14:30           Prévisions │
│ ○──────●──────○──────○──────○       │
│ Maint. +2h   +4h   +6h   +8h  +24h  │
├─────────────────────────────────────┤
│  🗺️        ➕         👤            │
│ Explore    Add      Profile         │
└─────────────────────────────────────┘
```

**Fonctionnalités :**
- Carte interactive Leaflet/Mapbox
- Marqueurs colorés selon conditions
- Slider temporel 6 créneaux
- Filtres par région
- Recherche textuelle

### 5.2 Écran Spot (Fiche détail)

```
┌─────────────────────────────────────┐
│ ←  Catalans                         │
│    Marseille · Mer 15 14:30         │
│    ────────────────────────────     │
│    🟢 Favorables    🌡️ 18°C        │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │
│ │  ✓  Conditions Favorables       │ │
│ │     Idéal pour tous niveaux     │ │
│ └─────────────────────────────────┘ │
│ ⚠️ L'évaluation appartient au nageur│
├─────────────────────────────────────┤
│ ┌─────┐ ┌─────┐ ┌─────┐            │
│ │🌡️   │ │💨   │ │🌊   │            │
│ │ Eau │ │Vent │ │Vague│            │
│ │18°C │ │10   │ │0.2m │            │
│ └─────┘ └─────┘ └─────┘            │
│ ┌─────┐ ┌─────┐ ┌─────┐            │
│ │🌙   │ │🎐   │ │🌀   │            │
│ │Marée│ │Médus│ │Cour.│            │
│ │ ↗   │ │Faib │ │Faib │            │
│ └─────┘ └─────┘ └─────┘            │
├─────────────────────────────────────┤
│ 🗣️ Infos locales (3)    [+ Ajouter]│
│ ┌─────────────────────────────────┐ │
│ │ 💡 Meilleur créneau   Vérifié×12│ │
│ │ Tôt le matin (6h-8h)            │ │
│ │ @MarineLuc 127⭐    [✓ Vérifier]│ │
│ └─────────────────────────────────┘ │
├─────────────────────────────────────┤
│ 🧭 Parcours (2)           [+ Créer]│
│ ┌─────────────────────────────────┐ │
│ │ 0.8 │ Triangle des bouées    › │ │
│ │ km  │ Débutant · 45 nages       │ │
│ └─────────────────────────────────┘ │
├─────────────────────────────────────┤
│  🗺️        ➕         👤            │
└─────────────────────────────────────┘
```

### 5.3 Écran Route (Fiche parcours)

```
┌─────────────────────────────────────┐
│ ←  Triangle des bouées              │
│    Catalans · 0.8 km · 45 nages     │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │
│ │      [CARTE AVEC TRACÉ]         │ │
│ │    🟢──────🔵                   │ │
│ │        └────🔵                  │ │
│ │  Départ/Arrivée = 🟢 vert       │ │
│ │  Waypoints = 🔵 bleu            │ │
│ └─────────────────────────────────┘ │
├─────────────────────────────────────┤
│ ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│ │ ~16 min │ │   ⭐    │ │  @Luc   │ │
│ │  Durée  │ │Débutant │ │Créateur │ │
│ └─────────┘ └─────────┘ └─────────┘ │
├─────────────────────────────────────┤
│ ⚠️ Vérifiez les conditions avant.   │
├─────────────────────────────────────┤
│ 📝 Description                      │
│ ┌─────────────────────────────────┐ │
│ │ Parcours triangulaire balisé    │ │
│ │ par 3 bouées jaunes.            │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
```

### 5.4 Écran Add (Création)

```
┌─────────────────────────────────────┐
│ ←  Ajouter    [🔍 Rechercher...]    │
├─────────────────────────────────────┤
│                                     │
│           [CARTE AVEC SPOTS]        │
│                                     │
│     🏊 Catalans    🏊 Prado         │
│                                     │
│   👆 Sélectionnez un spot ou        │
│      cliquez pour en créer un       │
│                                     │
├─────────────────────────────────────┤
│ Que souhaitez-vous ajouter ?        │
│ ┌────┐ ┌────┐ ┌────┐ ┌────┐        │
│ │ 📍 │ │ 💡 │ │ 🚨 │ │ 🧭 │        │
│ │Spot│ │Info│ │Dang│ │Parc│        │
│ │+20⭐│ │+10⭐│ │+15⭐│ │+25⭐│       │
│ └────┘ └────┘ └────┘ └────┘        │
│ 📍 Cliquez sur un spot existant     │
├─────────────────────────────────────┤
│  🗺️        ➕         👤            │
└─────────────────────────────────────┘
```

**États des boutons :**

| État | Spot | Info | Danger | Parcours |
|------|------|------|--------|----------|
| Initial (rien sélectionné) | Grisé | Grisé | Grisé | Grisé |
| Clic sur carte (nouveau) | ✅ Actif | Grisé | Grisé | Grisé |
| Spot existant sélectionné | Grisé | ✅ Actif | ✅ Actif | ✅ Actif |

### 5.5 Écran Profile

```
┌─────────────────────────────────────┐
│            👤 Mon Profil            │
├─────────────────────────────────────┤
│                                     │
│               🏊                    │
│          @SwimmerLuc                │
│             France                  │
│                                     │
│     ┌───────┬───────┬───────┐      │
│     │  127  │   5   │  12   │      │
│     │Karma⭐│Parcour│ Infos │      │
│     └───────┴───────┴───────┘      │
│                                     │
├─────────────────────────────────────┤
│  🗺️        ➕         👤            │
└─────────────────────────────────────┘
```

---

## 6. Composants UI

### 6.1 Design tokens (CSS Variables)

```css
:root {
  /* Couleurs principales */
  --primary: #0369a1;         /* Bleu océan */
  --bg: #f8fafc;              /* Fond */
  --white: #ffffff;           /* Blanc */
  --text: #1e293b;            /* Texte principal */
  --text-muted: #64748b;      /* Texte secondaire */
  --border: #e2e8f0;          /* Bordures */
  
  /* Couleurs de statut */
  --favorable: #16a34a;       /* Vert */
  --favorable-bg: rgba(22, 163, 74, 0.1);
  --technique: #f59e0b;       /* Orange */
  --technique-bg: rgba(245, 158, 11, 0.1);
  --exigeant: #ef4444;        /* Rouge */
  --exigeant-bg: rgba(239, 68, 68, 0.1);
}
```

### 6.2 Marqueur carte

```html
<div class="custom-marker">
  <div class="marker-temp">18°C</div>
  <div class="marker-pin favorable"></div>
  <div class="marker-name">Catalans</div>
</div>
```

```css
.custom-marker {
  text-align: center;
  cursor: pointer;
}
.marker-temp {
  background: white;
  padding: 2px 6px;
  border-radius: 8px;
  font-size: 10px;
  font-weight: 700;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}
.marker-pin {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  border: 3px solid white;
  margin: 4px auto;
  box-shadow: 0 2px 4px rgba(0,0,0,0.3);
}
.marker-pin.favorable { background: var(--favorable); }
.marker-pin.technique { background: var(--technique); }
.marker-pin.exigeant { background: var(--exigeant); }
```

### 6.3 Carte condition (grille 3×2)

```html
<div class="conditions-grid">
  <div class="condition-card">
    <div class="condition-label">🌡️ Eau</div>
    <div class="condition-value">18°C</div>
    <div class="condition-desc">Confort</div>
  </div>
  <!-- 5 autres cartes... -->
</div>
```

```css
.conditions-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
  margin-bottom: 16px;
}
.condition-card {
  background: var(--white);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 10px;
  text-align: center;
}
.condition-card.warning {
  background: var(--technique-bg);
  border-color: rgba(245,158,11,0.3);
}
.condition-card.danger {
  background: var(--exigeant-bg);
  border-color: rgba(239,68,68,0.3);
}
```

### 6.4 Bloc Hero statut

```html
<div class="status-hero favorable">
  <div class="status-hero-icon">✓</div>
  <div class="status-hero-content">
    <div class="status-hero-title">Conditions Favorables</div>
    <div class="status-hero-subtitle">Idéal pour tous niveaux</div>
  </div>
</div>
```

### 6.5 Carte info locale

```html
<div class="local-card">
  <div class="local-card-header">
    <div class="local-card-icon tip">💡</div>
    <div class="local-card-title">Meilleur créneau</div>
    <div class="local-card-badge">Vérifié ×12</div>
  </div>
  <div class="local-card-desc">Tôt le matin (6h-8h).</div>
  <div class="local-card-footer">
    <div class="local-card-karma">
      Par <strong>@MarineLuc</strong> 
      <span class="karma-badge">127 ⭐</span>
    </div>
    <button class="verify-btn">✓ Vérifier</button>
  </div>
</div>
```

### 6.6 Carte parcours

```html
<div class="route-card">
  <div class="route-card-header">
    <div class="route-distance">
      <div class="route-distance-value">0.8</div>
      <div class="route-distance-unit">km</div>
    </div>
    <div class="route-info">
      <div class="route-name">Triangle des bouées</div>
      <div class="route-meta">Débutant · 45 nages</div>
    </div>
    <span class="route-arrow">›</span>
  </div>
</div>
```

### 6.7 Barre de navigation

```html
<div class="nav-bar">
  <button class="nav-item active">
    <span class="nav-icon">🗺️</span>
    <span class="nav-label">Explore</span>
  </button>
  <button class="nav-item">
    <span class="nav-icon">➕</span>
    <span class="nav-label">Add</span>
  </button>
  <button class="nav-item">
    <span class="nav-icon">👤</span>
    <span class="nav-label">Profile</span>
  </button>
</div>
```

### 6.8 Toast notification

```html
<div class="toast" id="toast">Spot créé ! +20 karma ⭐</div>
```

```javascript
function showToast(message) {
  const toast = document.getElementById('toast');
  toast.textContent = message;
  toast.classList.add('show');
  setTimeout(() => toast.classList.remove('show'), 2500);
}
```

---

## 7. Fonctionnalités détaillées

### 7.1 Slider temporel

**Comportement :**
1. 6 créneaux : Maintenant, +2h, +4h, +6h, +8h, +24h
2. Mise à jour des marqueurs carte en temps réel
3. Mise à jour de la fiche spot si ouverte

```javascript
const timeOffsets = [0, 2, 4, 6, 8, 24];

function getTimeLabel(offsetIndex) {
  const now = new Date();
  const offset = timeOffsets[offsetIndex];
  const futureTime = new Date(now.getTime() + offset * 60 * 60 * 1000);
  
  const days = ['Dim', 'Lun', 'Mar', 'Mer', 'Jeu', 'Ven', 'Sam'];
  const day = days[futureTime.getDay()];
  const date = futureTime.getDate();
  const hours = futureTime.getHours().toString().padStart(2, '0');
  const minutes = futureTime.getMinutes().toString().padStart(2, '0');
  
  return `${day} ${date} · ${hours}:${minutes}`;
}
```

### 7.2 Calcul des risques dérivés

```javascript
function calculateDerivedRisks(condition) {
  const { temp, wind, waves } = condition;
  
  // Risque méduses : eau chaude + calme
  const jellyfishRisk = temp > 17 && waves < 0.5 ? 'Possible' : 'Faible';
  
  // Intensité courants : vent fort ou vagues hautes
  let currentsRisk;
  if (wind > 18 || waves > 0.7) {
    currentsRisk = 'Forts';
  } else if (wind > 12) {
    currentsRisk = 'Modérés';
  } else {
    currentsRisk = 'Faibles';
  }
  
  return { jellyfishRisk, currentsRisk };
}
```

### 7.3 Création de spot

```javascript
async function createSpot(data) {
  // Validation
  if (userKarma < KARMA_REQUIRED_SPOT) {
    throw new Error('Karma insuffisant');
  }
  if (!data.name) {
    throw new Error('Nom requis');
  }
  if (!data.lat || !data.lng) {
    throw new Error('Position GPS requise');
  }
  
  // Création
  const newSpot = {
    id: generateId(),
    name: data.name,
    region: 'Communauté',
    lat: data.lat,
    lng: data.lng,
    type: data.type || 'plage',
    access: data.access || 'facile',
    description: data.description || '',
    createdBy: currentUser.id,
    createdAt: new Date(),
    conditions: generateDefaultConditions()
  };
  
  // Sauvegarder
  await api.spots.create(newSpot);
  
  // Mettre à jour karma
  await updateUserKarma(currentUser.id, 20);
  
  return newSpot;
}
```

### 7.4 Tracé de parcours

**Algorithme de fermeture de boucle :**

```javascript
function calculateRouteDistance(points, isLoopClosed) {
  let distance = 0;
  
  for (let i = 1; i < points.length; i++) {
    distance += calculateHaversineDistance(points[i-1], points[i]);
  }
  
  // Ajouter retour au départ si boucle fermée
  if (isLoopClosed && points.length >= 2) {
    distance += calculateHaversineDistance(
      points[points.length - 1], 
      points[0]
    );
  }
  
  return distance / 1000; // Convertir en km
}

function calculateHaversineDistance(p1, p2) {
  const R = 6371000; // Rayon Terre en mètres
  const φ1 = p1.lat * Math.PI / 180;
  const φ2 = p2.lat * Math.PI / 180;
  const Δφ = (p2.lat - p1.lat) * Math.PI / 180;
  const Δλ = (p2.lng - p1.lng) * Math.PI / 180;

  const a = Math.sin(Δφ/2) ** 2 +
            Math.cos(φ1) * Math.cos(φ2) *
            Math.sin(Δλ/2) ** 2;
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));

  return R * c;
}
```

### 7.5 Calcul durée estimée

```javascript
function estimateSwimDuration(distanceKm) {
  const averageSpeedKmH = 3; // Vitesse moyenne nage eau libre
  const durationMinutes = Math.round(distanceKm / averageSpeedKmH * 60);
  
  if (durationMinutes >= 60) {
    const hours = Math.floor(durationMinutes / 60);
    const mins = durationMinutes % 60;
    return `~${hours}h${mins > 0 ? mins : ''}`;
  }
  return `~${durationMinutes} min`;
}
```

---

## 8. API et intégrations

### 8.1 Endpoints REST

```yaml
# Spots
GET    /api/spots                    # Liste tous les spots
GET    /api/spots/:id                # Détail d'un spot
POST   /api/spots                    # Créer un spot
PUT    /api/spots/:id                # Modifier un spot
DELETE /api/spots/:id                # Supprimer un spot

# Routes
GET    /api/spots/:spotId/routes     # Liste parcours d'un spot
GET    /api/routes/:id               # Détail d'un parcours
POST   /api/spots/:spotId/routes     # Créer un parcours
DELETE /api/routes/:id               # Supprimer un parcours

# Infos locales
GET    /api/spots/:spotId/infos      # Liste infos d'un spot
POST   /api/spots/:spotId/infos      # Créer une info
POST   /api/infos/:id/verify         # Vérifier une info
DELETE /api/infos/:id                # Supprimer une info

# Conditions
GET    /api/spots/:id/conditions     # Conditions actuelles + prévisions

# Users
GET    /api/users/me                 # Profil courant
PUT    /api/users/me                 # Modifier profil
GET    /api/users/:id/karma          # Karma d'un utilisateur
```

### 8.2 Intégrations externes

#### Météo-France API

```javascript
const METEO_FRANCE_API = 'https://api.meteo-france.com/v1';

async function fetchMarineConditions(lat, lng) {
  const response = await fetch(
    `${METEO_FRANCE_API}/marine?lat=${lat}&lng=${lng}`,
    { headers: { 'Authorization': `Bearer ${METEO_API_KEY}` } }
  );
  return response.json();
}
```

#### SHOM (Service Hydrographique)

```javascript
const SHOM_API = 'https://services.shom.fr/api';

async function fetchTideData(portId) {
  const response = await fetch(
    `${SHOM_API}/maree/${portId}/predictions`,
    { headers: { 'X-API-Key': SHOM_API_KEY } }
  );
  return response.json();
}
```

#### Mapbox/Leaflet

```javascript
// Initialisation carte Leaflet
function initMap(containerId, center, zoom) {
  const map = L.map(containerId, {
    center: [center.lat, center.lng],
    zoom: zoom,
    zoomControl: false
  });
  
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap'
  }).addTo(map);
  
  return map;
}
```

---

## 9. Variables et constantes

### 9.1 Constantes globales

```javascript
// Système Karma
const KARMA_REQUIRED_SPOT = 50;
const KARMA_REWARD_SPOT = 20;
const KARMA_REWARD_ROUTE = 25;
const KARMA_REWARD_INFO = 10;
const KARMA_REWARD_DANGER = 15;
const KARMA_REWARD_VERIFY = 5;

// Créneaux horaires (heures)
const TIME_OFFSETS = [0, 2, 4, 6, 8, 24];

// Vitesse moyenne nage (km/h)
const AVERAGE_SWIM_SPEED = 3;

// Seuils conditions
const THRESHOLDS = {
  temp: { warning: 15, danger: 12 },
  wind: { warning: 15, danger: 20 },
  waves: { warning: 0.5, danger: 0.8 }
};

// Régions prédéfinies
const REGIONS = {
  marseille: { lat: 43.25, lng: 5.40, zoom: 11 },
  wimereux: { lat: 50.75, lng: 1.65, zoom: 10 }
};
```

### 9.2 Types d'icônes

```javascript
const ICONS = {
  info: {
    tip: '💡',
    warning: '⚠️',
    danger: '🚨'
  },
  actions: {
    spot: '📍',
    info: '💡',
    danger: '🚨',
    route: '🧭'
  },
  conditions: {
    temp: '🌡️',
    wind: '💨',
    waves: '🌊',
    tide: '🌙',
    jellyfish: '🎐',
    currents: '🌀'
  },
  status: {
    favorable: '✓',
    technique: '!',
    exigeant: '✕'
  }
};
```

---

## 10. Guide d'implémentation

### 10.1 Phase 1 : MVP (4-6 semaines)

**Objectifs :**
- Carte interactive avec spots
- Fiche spot avec conditions
- Système utilisateur basique

**Tâches :**
1. Setup projet (React Native / Flutter)
2. Intégration Mapbox/Leaflet
3. API spots (CRUD)
4. Écran Explore + Spot detail
5. Authentification basique

### 10.2 Phase 2 : Communauté (4-6 semaines)

**Objectifs :**
- Système karma complet
- Création de contenu
- Infos locales + vérification

**Tâches :**
1. Système karma (calcul, stockage)
2. Création spot avec validation
3. Infos locales (CRUD + vérification)
4. Écran Add complet
5. Profil utilisateur

### 10.3 Phase 3 : Parcours (3-4 semaines)

**Objectifs :**
- Tracé de parcours interactif
- Fiche parcours avec carte
- Statistiques parcours

**Tâches :**
1. Éditeur de tracé (Leaflet draw)
2. Calcul distance automatique
3. Validation boucle
4. Fiche parcours avec tracé
5. Compteur de nages

### 10.4 Phase 4 : Données temps réel (4-6 semaines)

**Objectifs :**
- Intégration Météo-France
- Prévisions multi-créneaux
- Alertes automatiques

**Tâches :**
1. Intégration API Météo-France
2. Intégration SHOM (marées)
3. Algorithme calcul conditions
4. Slider temporel fonctionnel
5. Notifications push

### 10.5 Checklist technique

```markdown
## Backend
- [ ] PostgreSQL + PostGIS configuré
- [ ] Modèles de données créés
- [ ] API REST implémentée
- [ ] Authentification JWT
- [ ] Rate limiting
- [ ] Logging

## Frontend
- [ ] Navigation entre écrans
- [ ] Carte interactive
- [ ] Formulaires création
- [ ] Gestion d'état (Redux/Zustand)
- [ ] Offline support (cache)
- [ ] Tests unitaires

## Intégrations
- [ ] Météo-France API connectée
- [ ] SHOM API connectée
- [ ] Push notifications configurées
- [ ] Analytics (Mixpanel/Amplitude)

## DevOps
- [ ] CI/CD pipeline
- [ ] Staging environment
- [ ] Production environment
- [ ] Monitoring (Sentry)
- [ ] Backup automatique
```

---

## Annexes

### A. Coordonnées spots initiaux

| Spot | Latitude | Longitude | Région |
|------|----------|-----------|--------|
| Catalans | 43.2905 | 5.3545 | Marseille |
| Plages du Prado | 43.2610 | 5.3750 | Marseille |
| Pointe Rouge | 43.2450 | 5.3680 | Marseille |
| Calanque Sormiou | 43.2100 | 5.4180 | Marseille |
| Calanque En-Vau | 43.2020 | 5.4950 | Marseille |
| Cassis | 43.2140 | 5.5370 | Marseille |
| Îles du Frioul | 43.2795 | 5.3100 | Marseille |
| Plage de Wimereux | 50.7680 | 1.6130 | Wimereux |
| Boulogne-sur-Mer | 50.7260 | 1.5980 | Wimereux |
| Cap Blanc-Nez | 50.9290 | 1.7070 | Wimereux |
| Le Portel | 50.7080 | 1.5720 | Wimereux |
| Ambleteuse | 50.8050 | 1.6080 | Wimereux |

### B. Exemples de conditions

```json
{
  "spotId": "catalans",
  "conditions": [
    { "status": "favorable", "temp": 18, "wind": 10, "waves": 0.2 },
    { "status": "favorable", "temp": 18, "wind": 12, "waves": 0.3 },
    { "status": "favorable", "temp": 18, "wind": 14, "waves": 0.4 },
    { "status": "technique", "temp": 17, "wind": 18, "waves": 0.5 },
    { "status": "favorable", "temp": 17, "wind": 8, "waves": 0.2 },
    { "status": "favorable", "temp": 18, "wind": 6, "waves": 0.1 }
  ]
}
```

### C. Messages UI

```javascript
const MESSAGES = {
  errors: {
    karmaInsufficient: 'Karma insuffisant (min. 50 ⭐)',
    spotNameRequired: 'Donnez un nom au spot',
    routeNameRequired: 'Donnez un nom au parcours',
    routeMinPoints: 'Tracez au moins 2 points',
    routeNotClosed: 'Cliquez "Boucler" pour fermer le parcours',
    infoTitleRequired: 'Donnez un titre à l\'info',
    selectSpotFirst: 'Sélectionnez d\'abord un spot'
  },
  success: {
    spotCreated: 'Spot créé ! +20 karma ⭐',
    routeCreated: 'Parcours créé ! +25 karma ⭐',
    infoPublished: 'Info publiée ! +10 karma ⭐',
    infoVerified: 'Merci ! +5 karma ⭐',
    loopClosed: 'Boucle fermée !'
  },
  hints: {
    selectSpot: '👆 Sélectionnez un spot ou cliquez pour en créer un',
    drawRoute: '👆 Cliquez sur la carte pour tracer',
    continueDrawing: '👆 Continuez le tracé...'
  }
};
```

---

**Document généré le :** Janvier 2026  
**Version prototype :** SwimSpot V5.0  
**Auteur :** Claude (Anthropic) pour AquaNav SAS
