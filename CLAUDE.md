# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the App

There is no build step. Serve the project with any static HTTP server:

```bash
python3 -m http.server 8000
# then open http://localhost:8000/Bowen_HydroClean_App_v2.html
```

The Service Worker (`sw.js`) only registers on `localhost`, so PWA features require HTTP rather than a `file://` URL. `index.html` is a meta-refresh redirect to the main file.

There are no tests, no linter, and no package manager.

## Architecture

The entire application is a single self-contained file: **`Bowen_HydroClean_App_v2.html`** (~5 300 lines). It contains inline CSS (`<style>`), inline JavaScript (`<script>`), and references three CDN libraries:

| Library | Version | Purpose |
|---------|---------|---------|
| jsPDF | 2.5.1 | PDF generation for invoices and reports |
| Chart.js | 4.4.0 | Revenue/activity charts on the dashboard |
| qrcodejs | 1.0.0 | QR codes on job-site briefing documents |

Google Fonts (Syne + DM Sans) are loaded from `fonts.googleapis.com`.

### State and Storage

All runtime data lives in a single global `let state = { … }` object (line 326). Persistence is browser `localStorage` under the `bhc-*` namespace:

| Key | Content |
|-----|---------|
| `bhc-clients` | Client records |
| `bhc-chantiers` | Job sites / projects |
| `bhc-monthly` | Monthly revenue summaries |
| `bhc-employes` | Employees |
| `bhc-heures` | Time-tracking entries |
| `bhc-devislog` | Quote history |
| `bhc-factures` | Invoices |
| `bhc-settings` | Company banking/TVA settings |
| `bhc-campagnes` | Marketing campaigns |
| `bhc-prospection` | Prospecting data |

`loadStorage()` hydrates all fields from localStorage into `state` on startup. `saveStorage(field)` writes a single field name back. Call it after mutating that slice of state (e.g. `saveStorage('clients')` after adding a client).

### Render Loop

The UI is entirely string-based (`innerHTML`):

```
navigate(page) → state.page = page → render()
render()       → calls the matching renderXxx() function → sets #content.innerHTML
```

Every user action that mutates state must either call `render()` (to refresh the page) or directly patch the DOM. There is no reactive framework.

### Pages and Sub-tabs

| `state.page` | Render function | Sub-tab state key |
|---|---|---|
| `dashboard` | `renderDashboard()` | — |
| `clients` | `renderClients()` | — |
| `equipe` | `renderEquipe()` → `renderEquipeEmployes/Heures/Planning()` | `state.equipeTab` |
| `chantiers` | `renderChantiers()` | — |
| `calendrier` | `renderCalendrier()` | — |
| `devis` | `renderDevis()` | `state.devisTab` (`nouveau` / `envoyés`) |
| `factures` | `renderFactures()` | — |
| `rapport` | `renderRapport()` | — |
| `marketing` | `renderMarketing()` | `state.marketingTab` (`campagnes` / `flyer` / `prospection`) |
| `emails` | `renderEmails()` | — |

### Key Constants

Defined at module level; change these to adjust business logic:

```js
const PRIX = { 'Terrasse & allée': 7, 'Façade & mur': 10, 'Toiture & gouttières': 12 };
const PRIX_OPTIONS = { 'Sablage entre dalles': 4, 'Désherbage': 5, … };
const VITESSE_M2H = { 'Terrasse & allée': 15, 'Façade & mur': 10, 'Toiture & gouttières': 8 };
const BASE_ADDRESS = 'Route du Jordil 25, Domdidier, Suisse';
```

TVA is 8.1 % (Swiss rate). The locale throughout is `fr-CH`.

### Emoji Handling

Emojis are never written as literal characters in the source. They are built from code points via the `EM` constant (line 386) to prevent file-encoding issues:

```js
const EM = { briefing: String.fromCodePoint(0x1F4CB), wave: String.fromCodePoint(0x1F44B), … };
```

Follow the same pattern when adding new emojis.

### AI Features

Several functions call an AI backend for suggestions (e.g. `aiPrixSuggest()`, `aiChanEmail()`, `aiClientEmail()`, `genererMessageFlyer()`). These are wired to buttons labelled **✨ IA** in the UI. The AI endpoint configuration is inside those functions.

### ID Counters

Auto-increment IDs are tracked in state:

```js
state.nextClientId, state.nextChantierId, state.nextEmployeId,
state.nextHeureId, state.nextDevisId, state.nextFactureId
```

Increment the relevant counter and persist with `saveStorage` whenever you insert a new record.
