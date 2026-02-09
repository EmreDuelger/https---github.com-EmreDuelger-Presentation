# Emre's Präsentationen

Reveal.js Präsentationen mit vollautomatischer CI/CD Pipeline - gebaut mit einem Multi-Agenten-System.

## Präsentationen

| Präsentation | Thema | Datei |
|-------------|-------|-------|
| **DOM - Document Object Model** | W3C Standard, DOM-Baumstruktur, JavaScript DOM-API, interaktive Demo | `EmresPräsentationMitReveal.js/index.html` |
| **KI-Agenten** | Was sind Agenten, Agent-Loop, Typen, Multi-Agenten-Systeme, Frameworks | `EmresPräsentationMitReveal.js/agenten.html` |

## Projektstruktur

```
.
├── index.html                              # Landing Page (Links zu beiden Präsentationen)
├── README.md                               # Diese Datei
├── AGENTS.md                               # Agenten-Definitionen und -Konfiguration
├── .github/
│   └── workflows/
│       ├── ci.yml                          # HTML-Validierung + Link-Check
│       ├── auto-merge.yml                  # Auto-Merge nach bestandenen Checks
│       └── deploy-pages.yml                # GitHub Pages Deployment
└── EmresPräsentationMitReveal.js/
    ├── index.html                          # DOM-Präsentation
    ├── agenten.html                        # Agenten-Präsentation
    ├── css/                                # Reveal.js Styles + Themes
    ├── js/reveal.js                        # Reveal.js Framework
    ├── lib/                                # Support-Libraries + Fonts
    └── plugin/                             # Reveal.js Plugins
```

## Multi-Agenten-System

Dieses Projekt wird von **6 Agenten vollautomatisch** betrieben - **kein Human-in-the-Loop**:

```
Claude Code ──► git push ──► PR + Auto-Merge ──► CI prüft ──► GitHub merged ──► Pages Deploy
  [1-3]              [4]                          [5]       (Branch Protection)       [6]
```

| # | Agent | Plattform | Trigger | Konfiguration |
|---|-------|-----------|---------|---------------|
| 1 | Entwickler-Agent | Claude Code | Benutzer-Prompt | Per Prompt |
| 2 | Review-Agent | Claude Code (Sub-Agent) | Vom Entwickler-Agent | Per Prompt |
| 3 | Deployment-Agent | Claude Code (Sub-Agent) | Vom Entwickler-Agent | Per Prompt |
| 4 | PR + Auto-Merge-Agent | GitHub Actions | Push auf `claude/*` | `.github/workflows/create-pr.yml` |
| 5 | CI-Agent | GitHub Actions | PR geöffnet | `.github/workflows/ci.yml` |
| 6 | Pages-Deploy-Agent | GitHub Actions | Push auf master | `.github/workflows/deploy-pages.yml` |

Auto-Merge nutzt `gh pr merge --auto` und respektiert **Branch Protection Rules** - siehe [AGENTS.md](AGENTS.md).

Details zu jedem Agenten, ihren Triggern, Permissions und Inspektionsmöglichkeiten: siehe [AGENTS.md](AGENTS.md).

## CI/CD Pipeline

### Bei jedem Pull Request (`ci.yml`)
- HTML-Validierung aller `.html`-Dateien
- Link-Check: Prüft alle externen URLs auf Erreichbarkeit
- Dateigrößen-Report

### Nach bestandenen Checks (`auto-merge.yml`)
- Automatischer Squash-Merge von `claude/*` und `dependabot/*` Branches
- Manuelle PRs werden **nicht** auto-gemerged

### Nach Merge in master (`deploy-pages.yml`)
- Automatisches Deployment auf GitHub Pages

## Einrichtung GitHub Pages

1. **Settings > Pages > Build and deployment > Source**: "GitHub Actions" auswählen
2. Fertig - der `deploy-pages.yml` Workflow macht den Rest

## Lokal starten

```bash
cd EmresPräsentationMitReveal.js
# Option 1: Python
python -m http.server 8000
# Option 2: Node.js
npx serve .
# Dann öffnen: http://localhost:8000
```

## Technologie

- **Reveal.js** - Präsentations-Framework
- **GitHub Actions** - CI/CD Pipeline
- **GitHub Pages** - Hosting
- **Claude Code** - KI-gestützte Entwicklung
