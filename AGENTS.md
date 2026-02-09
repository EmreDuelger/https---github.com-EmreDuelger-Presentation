# Agenten-Definitionen

Dieses Projekt nutzt ein Multi-Agenten-System. Jeder Agent hat eine klar definierte Rolle, einen Trigger und eine Konfiguration.

---

## Übersicht

```
Push auf Branch ──► PR erstellt ──► CI prüft ──► Auto-Merge ──► Pages Deploy
     [1]              [2]           [3+4]          [5]            [6]
```

---

## Agent 1: Entwickler-Agent (Claude Code)

| Eigenschaft | Wert |
|-------------|------|
| **Plattform** | Claude Code (CLI / Agentic SDK) |
| **Trigger** | Manuell durch Benutzer-Prompt |
| **Aufgabe** | Code schreiben, Dateien erstellen/bearbeiten, Git-Operationen |
| **Tools** | Read, Write, Edit, Bash, Grep, Glob, Task (Sub-Agenten) |
| **Branch-Konvention** | `claude/*` |

### Was er macht
- Erstellt und bearbeitet HTML/CSS/JS-Dateien
- Erstellt Reveal.js Präsentationen
- Repariert kaputte Links und Bilder
- Pusht Änderungen auf Feature-Branches

### Wo konfiguriert
- Keine dauerhafte Konfigurationsdatei - wird per Prompt gesteuert
- Optional: `CLAUDE.md` im Repo-Root für projektspezifische Anweisungen

### So inspizierst du die Einstellungen
```bash
# Claude Code Konfiguration anzeigen
claude config list

# Projekt-spezifische Einstellungen
cat CLAUDE.md  # falls vorhanden
```

---

## Agent 2: Review-Agent (Claude Code Sub-Agent)

| Eigenschaft | Wert |
|-------------|------|
| **Plattform** | Claude Code (als Sub-Agent via `Task` Tool) |
| **Trigger** | Vom Entwickler-Agent gestartet (parallel) |
| **Aufgabe** | Code-Review: HTML-Validität, Inhalt, Konsistenz, Responsive prüfen |
| **Tools** | Read, Grep, Glob (nur lesend) |
| **Output** | Review-Bericht als Text (keine Dateiänderungen) |

### Was er macht
- Liest alle relevanten Dateien
- Prüft HTML-Struktur, CSS-Konsistenz, inhaltliche Korrektheit
- Gibt eine Bewertung (1-10) pro Kategorie
- Schlägt Verbesserungen vor

### Wo konfiguriert
- Wird inline im `Task`-Aufruf definiert (Prompt = Konfiguration)
- Kein eigener Workflow - lebt innerhalb der Claude Code Session

---

## Agent 3: Deployment-Agent (Claude Code Sub-Agent)

| Eigenschaft | Wert |
|-------------|------|
| **Plattform** | Claude Code (als Sub-Agent via `Task` Tool) |
| **Trigger** | Vom Entwickler-Agent gestartet (parallel mit Review-Agent) |
| **Aufgabe** | Erstellt Deployment-Artefakte (Landing Page, Konfiguration) |
| **Tools** | Read, Write, Edit, Bash |

### Was er macht
- Erstellt die Landing Page (`index.html` im Root)
- Konfiguriert Deployment-relevante Dateien

### Wo konfiguriert
- Wird inline im `Task`-Aufruf definiert

---

## Agent 4: CI-Agent (GitHub Actions)

| Eigenschaft | Wert |
|-------------|------|
| **Plattform** | GitHub Actions |
| **Trigger** | `pull_request` auf `master` oder `main` |
| **Aufgabe** | HTML-Validierung + Link-Check + Dateigrößen-Report |
| **Konfiguration** | `.github/workflows/ci.yml` |

### Was er macht

**Job 1: `validate` - HTML Validierung**
- Installiert `html-validate` via npm
- Prüft jede `.html`-Datei im Repo
- Schreibt Ergebnisse in die GitHub Step Summary

**Job 2: `review` - Automatisches Review**
- Extrahiert alle URLs aus HTML-Dateien
- Prüft jeden Link per `curl` (HTTP Status)
- Meldet kaputte Links (nicht blockierend)
- Listet Dateigrößen aller HTML-Dateien auf

### Wo konfiguriert
```
.github/workflows/ci.yml
```

### So inspizierst du die Einstellungen
```bash
# Workflow-Datei lesen
cat .github/workflows/ci.yml

# Auf GitHub: Actions Tab → "CI - HTML Validierung & Review"
# Dort siehst du alle Runs mit Step Summary
```

### Permissions
```yaml
permissions:
  contents: read
  pull-requests: write
```

---

## Agent 5: Auto-Merge-Agent (GitHub Actions)

| Eigenschaft | Wert |
|-------------|------|
| **Plattform** | GitHub Actions |
| **Trigger** | `check_suite` completed + `pull_request_review` submitted |
| **Aufgabe** | PRs automatisch mergen wenn alle Checks bestanden |
| **Konfiguration** | `.github/workflows/auto-merge.yml` |
| **Merge-Methode** | Squash Merge |

### Was er macht
1. Wartet bis eine `check_suite` mit `conclusion: success` eintrifft
2. Listet alle offenen PRs auf
3. Filtert: Nur PRs von `claude/*` oder `dependabot/*` Branches
4. Prüft: Sind alle Check-Runs `success` oder `skipped`?
5. Wenn ja: Squash-Merge in master

### Sicherheitsmechanismen
- Merged **nur** PRs von Bot-Branches (`claude/*`, `dependabot/*`)
- Manuelle PRs (z.B. `feature/xyz`) werden **nicht** auto-gemerged
- Alle CI-Checks müssen bestanden sein

### Wo konfiguriert
```
.github/workflows/auto-merge.yml
```

### So inspizierst du die Einstellungen
```bash
# Workflow-Datei lesen
cat .github/workflows/auto-merge.yml

# Auf GitHub: Actions Tab → "Auto-Merge nach CI"
# Dort siehst du welche PRs gemerged/übersprungen wurden
```

### Permissions
```yaml
permissions:
  contents: write       # zum Mergen
  pull-requests: write  # zum Lesen der PRs
```

---

## Agent 6: Pages-Deploy-Agent (GitHub Actions)

| Eigenschaft | Wert |
|-------------|------|
| **Plattform** | GitHub Actions |
| **Trigger** | `push` auf `master` oder `main` |
| **Aufgabe** | Automatisches Deployment auf GitHub Pages |
| **Konfiguration** | `.github/workflows/deploy-pages.yml` |

### Was er macht
1. Checkt den Code aus
2. Konfiguriert GitHub Pages
3. Lädt das gesamte Repo als Artifact hoch
4. Deployt auf GitHub Pages
5. Schreibt die Live-URLs in die Step Summary

### Wo konfiguriert
```
.github/workflows/deploy-pages.yml
```

### So inspizierst du die Einstellungen
```bash
# Workflow-Datei lesen
cat .github/workflows/deploy-pages.yml

# Auf GitHub: Actions Tab → "Deploy GitHub Pages"
# Environment: Settings → Environments → "github-pages"
```

### Permissions
```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

### Voraussetzung
In den GitHub Repo Settings muss aktiviert sein:
- **Settings > Pages > Source**: "GitHub Actions" (nicht "Deploy from a branch")

---

## Optionale Agenten (externe Dienste)

### GitHub Copilot (PR-Beschreibung)
| Eigenschaft | Wert |
|-------------|------|
| **Plattform** | GitHub.com |
| **Trigger** | PR-Erstellung |
| **Aufgabe** | Schlägt automatisch PR-Titel und -Beschreibung vor |
| **Aktivierung** | Wird von GitHub automatisch angeboten wenn Copilot aktiv ist |

### CodeRabbit (Code-Review)
| Eigenschaft | Wert |
|-------------|------|
| **Plattform** | coderabbit.ai (GitHub App) |
| **Trigger** | Neuer PR oder Push auf PR-Branch |
| **Aufgabe** | Tiefgehendes KI-Code-Review mit Inline-Kommentaren |
| **Aktivierung** | [coderabbit.ai](https://coderabbit.ai) installieren und Repo auswählen |
| **Konfiguration** | Optional: `.coderabbit.yaml` im Repo-Root |

---

## Gesamtarchitektur

```
Du (Prompt)
  │
  ▼
┌──────────────────────────────────┐
│  Claude Code (Entwickler-Agent)  │
│  ├── Review-Agent (Sub-Agent)    │
│  └── Deployment-Agent (Sub-Agent)│
└──────────┬───────────────────────┘
           │ git push claude/*
           ▼
┌──────────────────────────────────┐
│  GitHub Repository               │
│  ├── PR erstellt (Copilot)       │
│  ├── CodeRabbit reviewt          │
│  ├── CI-Agent prüft (.yml)       │
│  ├── Auto-Merge-Agent (.yml)     │
│  └── Pages-Deploy-Agent (.yml)   │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  GitHub Pages (Live)             │
│  └── Präsentationen online       │
└──────────────────────────────────┘
```

---

## Alle Konfigurationsdateien auf einen Blick

| Datei | Agent | Beschreibung |
|-------|-------|-------------|
| `.github/workflows/ci.yml` | CI-Agent | HTML-Validierung + Link-Check |
| `.github/workflows/auto-merge.yml` | Auto-Merge-Agent | Squash-Merge nach bestandenen Checks |
| `.github/workflows/deploy-pages.yml` | Pages-Deploy-Agent | Deployment auf GitHub Pages |
| `CLAUDE.md` (optional) | Entwickler-Agent | Projektspezifische Anweisungen für Claude |
| `.coderabbit.yaml` (optional) | CodeRabbit | Review-Konfiguration |
