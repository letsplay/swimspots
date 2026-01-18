# 🏊 SwimSpot

**Application communautaire de navigation pour nageurs en eau libre**

[![License](https://img.shields.io/badge/license-Proprietary-red.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-5.0-blue.svg)](CHANGELOG.md)
[![Status](https://img.shields.io/badge/status-Prototype-orange.svg)]()

---

## 🌊 À propos

SwimSpot est une PWA permettant aux nageurs en eau libre de :

- 🗺️ **Découvrir** des spots de baignade avec conditions en temps réel
- 📍 **Contribuer** des informations locales et signaler des dangers
- 🧭 **Créer** et partager des parcours de nage
- ⭐ **Gagner du karma** en aidant la communauté

## 📱 Wireframe interactif

👉 **[Ouvrir le prototype V5.0](https://letsplay.github.io/swimspots/swimspot-v5.0-complete.html)**

Le wireframe est entièrement fonctionnel et démontre :
- Navigation entre écrans
- Carte interactive (Leaflet)
- Slider de prévisions temporelles
- Création de spots (avec système karma)
- Création de parcours (tracé sur carte)
- Fiches détaillées spots et parcours

## 🎯 Fonctionnalités clés

### Carte Explore
| Fonctionnalité | Description |
|----------------|-------------|
| Marqueurs colorés | 🟢 Favorable · 🟡 Technique · 🔴 Exigeant |
| Slider temporel | 6 créneaux : Maintenant → +24h |
| Filtres régions | Marseille, Wimereux, etc. |

### Système Karma
| Action | Points |
|--------|--------|
| Créer un spot | +20 ⭐ (min. 50 requis) |
| Créer un parcours | +25 ⭐ |
| Ajouter une info | +10 ⭐ |
| Vérifier une info | +5 ⭐ |

### Types d'informations
| Type | Icône | Usage |
|------|-------|-------|
| Conseil | 💡 | Bons plans, astuces |
| Attention | ⚠️ | Prudence recommandée |
| Danger | 🚨 | Risque sérieux |

## 📂 Structure du projet

```
swimspots/
├── swimspot-v5.0-complete.html    # Wireframe interactif
├── README.md                       # Ce fichier
└── docs/
    ├── specifications-techniques.md
    └── specifications-techniques.docx
```

## 🛠️ Stack technique (cible)

| Couche | Technologie |
|--------|-------------|
| Frontend | React Native / Flutter / PWA |
| Backend | Node.js / Python (FastAPI) |
| Database | PostgreSQL + PostGIS |
| Maps | Mapbox GL JS / Leaflet |
| Weather | Météo-France API |
| Tides | SHOM API |

## 📊 Data Schema

### Spot
```typescript
interface Spot {
  id: string;
  name: string;
  region: string;
  lat: number;
  lng: number;
  type: 'plage' | 'crique' | 'port' | 'lac';
  conditions: Condition[];
}
```

### Condition
```typescript
interface Condition {
  status: 'favorable' | 'technique' | 'exigeant';
  temp: number;      // °C
  wind: number;      // km/h
  waves: number;     // m
}
```

### Route (Parcours)
```typescript
interface Route {
  id: string;
  spotId: string;
  name: string;
  distance: number;  // km
  level: 'debutant' | 'intermediaire' | 'avance';
  points: LatLng[];  // GPS coordinates (loop)
}
```

## 🚀 Roadmap

- [x] Wireframe V5.0 - Prototype interactif complet
- [ ] Phase 1 : MVP (carte + spots)
- [ ] Phase 2 : Système communautaire (karma, infos)
- [ ] Phase 3 : Parcours interactifs
- [ ] Phase 4 : Données temps réel (météo, marées)

## 📄 Documentation

- [Spécifications techniques (MD)](docs/specifications-techniques.md)
- [Spécifications techniques (DOCX)](docs/specifications-techniques.docx)

## 👥 Équipe

**AquaNav SAS** - Projet ClipHUD / SwimSpot

## 📝 License

Proprietary - © 2026 AquaNav SAS. All rights reserved.

---

<p align="center">
  <strong>🌊 Nagez en sécurité, nagez connecté 🌊</strong>
</p>
