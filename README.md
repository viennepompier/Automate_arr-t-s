# Dashboard Vienne — Arrêtés de circulation

Dashboard plein écran pour la ville de Vienne (38) affichant en temps réel les **arrêtés de circulation impliquant un blocage total de voie**, le débit du Rhône et la météo locale.

---

## Architecture

```
Gmail (arrêtés PDF)
        ↓
  Google Apps Script (Code.gs)
    ├── Gemini AI → analyse le PDF
    ├── Google Sheets → stocke les données
    └── API JSON publique
              ↓
        Dashboard HTML (index.html)
          ├── Arrêtés EN COURS / À VENIR
          ├── Débit Rhône (Hub'Eau)
          └── Météo (Open-Meteo)
```

---

## Fonctionnement global

### 1. Réception et analyse des arrêtés (Code.gs)

Les arrêtés de circulation arrivent par email au format PDF. Le script Apps Script tourne **toutes les heures** (déclencheur automatique) et pour chaque email non lu :

1. Extrait le PDF en pièce jointe
2. L'envoie à **Gemini AI** qui l'analyse et retourne un JSON structuré :
   - `lieu` — rue ou zone concernée
   - `date_debut` / `date_fin` — au format JJ/MM/AAAA
   - `motif` — description courte des travaux
   - `blocage_total` — `true` ou `false`
3. **Seuls les arrêtés avec `blocage_total: true` sont insérés dans le tableau** (fermeture complète de voie, accès interdit à tous). Les arrêtés de circulation alternée, stationnement interdit, voie rétrécie ou trottoir fermé sont classés dans un label séparé mais n'apparaissent pas sur le dashboard.
4. L'email est archivé et labelisé :
   - `Arrêtés/Blocage` → blocage total confirmé
   - `Arrêtés/Sans blocage` → arrêté sans fermeture totale
   - `Autres mails` → pas un arrêté de circulation

### 2. Gestion des dates

Les dates sont extraites par Gemini et normalisées en `JJ/MM/AAAA`.

**Cas particulier — arrêtés sans date de fin :**

Certains arrêtés ne précisent pas de date de fin (travaux à durée indéterminée, autorisations permanentes…). Dans ce cas :
- Si la **date de début est connue** → date de fin calculée = date début + 1 mois
- Si la **date de début est aussi inconnue** → date de fin = date de réception du mail + 1 mois

Cela garantit que ces arrêtés s'affichent sur le dashboard pendant une durée raisonnable puis disparaissent automatiquement, sans intervention manuelle.

### 3. Nettoyage automatique

À chaque cycle, le script :
- **Supprime** les lignes expirées (date fin dépassée) et envoie le mail à la corbeille
- **Met à jour** le statut de « À venir » à « En cours » quand la date de début est atteinte

### 4. Affichage (index.html)

Le dashboard interroge l'API Apps Script et affiche :

| Section | Contenu |
|---|---|
| **EN COURS** | Arrêtés actifs (défilement vertical auto si plusieurs) |
| **À VENIR** | Arrêtés futurs (slider horizontal) |
| **Rhône · Ternay** | Débit en m³/s — rouge à partir de 2 500 m³/s |
| **Météo · Vienne** | Température et conditions actuelles |

Le dashboard gère aussi les anciens arrêtés stockés avec le statut `Sans date` : il recalcule côté client la date d'expiration (date début ou réception + 1 mois) et les affiche dans la section EN COURS tant qu'ils ne sont pas expirés.

---

## Structure du Google Sheets

| Date Réception | Lieu | Date Début | Date Fin | Motif | Statut | ID Mail |
|---|---|---|---|---|---|---|
| Date du mail | Rue/lieu | JJ/MM/AAAA | JJ/MM/AAAA | Description | En cours / À venir | ID Gmail |

---

## Configuration

### Variables à modifier dans `Code.gs`

```javascript
const API_KEY    = 'VOTRE_CLE_GEMINI';  // Clé API Google AI Studio
const BATCH_SIZE = 10;                   // Mails traités par exécution (augmenter si pas de risque de timeout)
```

### Variable à modifier dans `index.html`

```javascript
const APPS_SCRIPT_URL = 'https://script.google.com/macros/s/VOTRE_ID/exec';
```

---

## Déploiement

### Première mise en service

1. Créer un projet Google Apps Script lié à un Google Sheets
2. Coller le contenu de `Code.gs`
3. Créer la ligne d'en-tête dans le Sheets : `Date Réception | Lieu | Date Début | Date Fin | Motif | Statut | ID Mail`
4. **Déployer** → Nouvelle déploiement → Application web → Accès : Tout le monde
5. Copier l'URL dans `index.html`
6. Configurer un **déclencheur horaire** sur `automateQuotidien()`

### Après chaque modification du script

```
Déployer → Gérer les déploiements → ✏️ Modifier → Nouvelle version → Déployer
```

> ⚠️ Sans nouvelle version, les modifications ne sont pas prises en compte par l'URL publique.

### Retraitement de tous les mails

Lancer `retraiterTousLesMails()` depuis Apps Script. Si timeout (> 6 min), relancer `automateQuotidien()` plusieurs fois jusqu'à voir `Mails restants : 0` dans les journaux.

---

## APIs utilisées

| Service | URL | Usage |
|---|---|---|
| Gemini AI | `generativelanguage.googleapis.com` | Analyse des PDFs |
| Hub'Eau | `hubeau.eaufrance.fr` | Débit du Rhône (station Ternay) |
| Open-Meteo | `api.open-meteo.com` | Météo Vienne |

---

## Fichiers

```
├── index.html                              # Dashboard HTML (standalone)
├── Code.gs                                 # Google Apps Script
└── Documentation_Dashboard_Vienne.docx    # Documentation technique complète
```
