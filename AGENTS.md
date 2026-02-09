# Agenten-Definitionen

Dieses Projekt nutzt ein Multi-Agenten-System. Jeder Agent hat eine klar definierte Rolle, einen Trigger und eine Konfiguration.

---

## Übersicht

```
Push auf Branch ──► PR + Auto-Merge ──► CI prüft ──► GitHub merged ──► Pages live
     [1]                [4]               [5]        (Branch Protection)   (auto)
  Claude Code      create-pr.yml        ci.yml       gh pr merge --auto  Deploy from branch
```

Alle Schritte laufen **vollautomatisch** - kein Human-in-the-Loop.
Auto-Merge respektiert **Branch Protection Rules** - GitHub merged erst wenn alle Regeln erfüllt sind.

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

## Agent 4: PR- und Auto-Merge-Agent (GitHub Actions)

| Eigenschaft | Wert |
|-------------|------|
| **Plattform** | GitHub Actions |
| **Trigger** | `push` auf `claude/**` oder `dependabot/**` Branches |
| **Aufgabe** | PR erstellen + Auto-Merge aktivieren |
| **Konfiguration** | `.github/workflows/create-pr.yml` |
| **Merge-Methode** | Squash Merge |

### Was er macht
1. Erkennt Push auf einen `claude/*` oder `dependabot/*` Branch
2. Prüft ob bereits ein offener PR für diesen Branch existiert
3. Falls nein: Sammelt alle Commits seit `master` und erstellt PR
4. Aktiviert **`gh pr merge --auto --squash`** auf dem PR
5. GitHub merged den PR **automatisch** sobald alle Branch Protection Rules erfüllt sind

### Warum `gh pr merge --auto`?
Im Gegensatz zu einem direkten Merge (`pulls.merge` API) respektiert `--auto` die **Branch Protection Rules**:
- Wartet auf required Status Checks (CI muss grün sein)
- Wartet auf required Reviews (falls konfiguriert)
- Wartet auf alle weiteren Branch Protection Regeln
- Merged erst wenn **alles** erfüllt ist

### Wo konfiguriert
```
.github/workflows/create-pr.yml
```

### So inspizierst du die Einstellungen
```bash
cat .github/workflows/create-pr.yml

# Auf GitHub: Actions Tab → "Auto PR erstellen + Auto-Merge aktivieren"
# Pull Requests Tab → "Auto-merge" Badge zeigt Status
```

### Permissions
```yaml
permissions:
  contents: write        # für Auto-Merge
  pull-requests: write   # für PR-Erstellung
```

---

## Agent 5: CI-Agent (GitHub Actions)

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

### Zusammenspiel mit Branch Protection
Wenn in den Branch Protection Rules `validate` und/oder `review` als **required Status Checks** konfiguriert sind, wartet der Auto-Merge-Agent (Agent 4) automatisch bis diese Jobs bestanden haben.

### Wo konfiguriert
```
.github/workflows/ci.yml
```

### So inspizierst du die Einstellungen
```bash
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

## GitHub Pages (kein Agent - eingebaute GitHub-Funktion)

| Eigenschaft | Wert |
|-------------|------|
| **Plattform** | GitHub Pages (nativ) |
| **Trigger** | Push auf `master` |
| **Aufgabe** | Statische HTML-Dateien direkt vom Branch servieren |
| **Konfiguration** | Settings > Pages > "Deploy from a branch" |

### Warum kein Workflow?
Für reine HTML/CSS/JS-Dateien (wie Reveal.js Präsentationen) braucht man keinen Build-Schritt.
GitHub Pages kann die Dateien direkt vom Branch servieren - einfacher, schneller, weniger Fehlerquellen.

---

## Optionale Agenten (externe Dienste)

### CodeRabbit (Code-Review)
| Eigenschaft | Wert |
|-------------|------|
| **Plattform** | coderabbit.ai (GitHub App) |
| **Trigger** | Neuer PR oder Push auf PR-Branch |
| **Aufgabe** | Tiefgehendes KI-Code-Review mit Inline-Kommentaren |
| **Aktivierung** | [coderabbit.ai](https://coderabbit.ai) installieren und Repo auswählen |
| **Konfiguration** | Optional: `.coderabbit.yaml` im Repo-Root |

---

## Branch Protection Setup

Damit die Pipeline korrekt funktioniert, muss **Branch Protection** auf `master` konfiguriert sein.

### Einrichtung: Settings > Branches > Branch protection rules > Add rule

| Einstellung | Wert | Warum |
|-------------|------|-------|
| **Branch name pattern** | `master` | Schützt den Hauptbranch |
| **Require a pull request before merging** | Aktiviert | Kein direkter Push auf master |
| **Require status checks to pass** | Aktiviert | CI muss grün sein vor Merge |
| **Status checks that are required** | `HTML Validierung`, `Automatisches Review` | Die Job-Namen aus `ci.yml` |
| **Require branches to be up to date** | Optional | Stellt sicher, dass Branch aktuell ist |
| **Do not allow bypassing** | Optional | Gilt auch für Admins |

### Einrichtung: Settings > General

| Einstellung | Wert | Warum |
|-------------|------|-------|
| **Allow auto-merge** | Aktiviert | Damit `gh pr merge --auto` funktioniert |

### Einrichtung: Settings > Actions > General

| Einstellung | Wert | Warum |
|-------------|------|-------|
| **Workflow permissions** | Read and write permissions | Workflows können PRs erstellen und mergen |
| **Allow GitHub Actions to create and approve pull requests** | Aktiviert | PR-Agent kann PRs erstellen |

### Einrichtung: Settings > Pages

| Einstellung | Wert | Warum |
|-------------|------|-------|
| **Source** | Deploy from a branch | Statische Dateien direkt von `master` servieren |
| **Branch** | `master` / `/ (root)` | Gesamtes Repo wird als Website bereitgestellt |

### So prüfst du die Branch Protection
```bash
# Auf GitHub: Settings → Branches → Branch protection rules
# Dort siehst du alle aktiven Regeln für master

# Oder per API:
gh api repos/{owner}/{repo}/branches/master/protection
```

---

## Gesamtarchitektur

```
Du (Prompt)
  │
  ▼
┌──────────────────────────────────┐
│  Claude Code (Entwickler-Agent)  │  Agent 1
│  ├── Review-Agent (Sub-Agent)    │  Agent 2
│  └── Deployment-Agent (Sub-Agent)│  Agent 3
└──────────┬───────────────────────┘
           │ git push claude/*
           ▼
┌──────────────────────────────────────────────┐
│  GitHub Actions                              │
│  ├── PR + Auto-Merge (create-pr.yml)         │  Agent 4
│  │     └── gh pr merge --auto --squash       │
│  └── CI-Agent (ci.yml)                       │  Agent 5
│        ├── HTML Validierung                  │
│        └── Link-Check                        │
└──────────┬───────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────┐
│  Branch Protection (GitHub)                  │
│  ├── Required status checks müssen bestehen  │
│  ├── Auto-Merge wartet auf grünes Licht      │
│  └── Merge → Push auf master                 │
└──────────┬───────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────┐
│  GitHub Pages (Deploy from branch)           │
│  └── Serviert statische Dateien von master   │
└──────────────────────────────────────────────┘
```

**Kein Human-in-the-Loop** - der gesamte Flow von Push bis Live-Seite ist vollautomatisch.
Branch Protection stellt sicher, dass nur geprüfter Code auf master landet.
GitHub Pages braucht keinen Workflow - es serviert die Dateien direkt vom Branch.

---

## Alle Konfigurationsdateien auf einen Blick

| Datei | Agent | Beschreibung |
|-------|-------|-------------|
| `.github/workflows/create-pr.yml` | PR + Auto-Merge (#4) | PR erstellen + Auto-Merge aktivieren |
| `.github/workflows/ci.yml` | CI-Agent (#5) | HTML-Validierung + Link-Check |
| `CLAUDE.md` (optional) | Entwickler-Agent (#1) | Projektspezifische Anweisungen für Claude |
| `.coderabbit.yaml` (optional) | CodeRabbit | Review-Konfiguration |
