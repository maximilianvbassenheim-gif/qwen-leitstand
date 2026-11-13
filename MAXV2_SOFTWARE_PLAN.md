# MAX V2 Software-Plan: Qwen-Leitstand

Stand: 2026-11-13, Europe/Berlin  
Produzent: engineering-lead  
Status: nur Plan (PAPER_ONLY). Kein Branch, kein PR, kein Commit, kein Deploy, kein Merge.

## Ziel

Ein schlanker, ausschließlich lokaler Dienst stellt den „Qwen-Leitstand“ bereit: Qwen (über Ollama) plant und steuert als Chef, Codex CLI arbeitet als rechte Hand, Claude Code als linke Hand. Beide Hände werden nur als Werkzeuge aufgerufen. Docker ist die Werkbank für Läufe. Handlungen mit Geld, Senden oder Löschen sind hart gesperrt und laufen erst nach ausdrücklicher menschlicher Bestätigung.

## Annahmen

Die folgenden Punkte sind Annahmen, keine festgestellten Fakten:

- Annahme: Qwen ist über Ollama auf demselben Rechner erreichbar.
- Annahme: Codex CLI und Claude Code sind als Kommandozeilenwerkzeuge installierbar und per Subprozess aufrufbar.
- Annahme: Der Dienst bindet im MVP nur an `127.0.0.1` und braucht keinen Fernzugriff.
- Annahme: Ein späterer Draft-PR erfolgt erst nach einem ausdrücklichen Max-Ja und auf einem noch festzulegenden öffentlichen Repo.
- Annahme: Die beiden Hände werden im MVP nacheinander gerufen, nicht parallel.

## Architektur

Der Leitstand ist ein kleiner Python-Dienst (FastAPI oder vergleichbarer lokaler Daemon). Er lauscht nur auf `127.0.0.1`, nicht auf `0.0.0.0`.

Ablauf einer Aufgabe:

1. Ein lokaler Client sendet die Aufgabe an `POST /task`.
2. Der Orchestrator übergibt Ziel und Kontext an Qwen/Ollama. Qwen liefert einen Plan und wählt Werkzeuge, führt aber selbst nichts aus.
3. Erlaubte Werkzeugaufrufe gehen an Adapter: Codex CLI (rechte Hand) oder Claude Code (linke Hand). Aufruf nur per Subprozess, mit Timeout, ohne Shell-Interpolation, gegen eine Befehls-Allowlist.
4. Läufe, die Code oder Befehle ausführen, starten in Docker-Containern (Werkbank): Netzwerk standardmäßig aus, keine Host-Mounts für Secrets oder Keychain, begrenzte Rechte, befristete Arbeitsverzeichnisse.
5. Aktionen der Klassen Geld, Senden oder Löschen werden nicht ausgeführt, sondern in eine Freigabewarteschlange gelegt. Erst `POST /approve/{id}` (menschliche Bestätigung) darf sie freigeben. Ohne Freigabe bleibt die Aktion blockiert.
6. Jeder relevante Schritt wird append-only ins Audit-Log geschrieben.

## Komponenten und Ordner

```
qwen-leitstand/
  app/           # Dienst, API, Orchestrator, Qwen-Client
  tools/         # Adapter für Codex CLI und Claude Code, Allowlist
  gates/         # Sperren, Freigabewarteschlange, Bestätigung
  workbench/     # Docker-Laufvorschriften und Image-Definitionen
  audit/         # JSONL-Log und Hash-Bildung (kein Secret-Inhalt)
  config/        # nicht-geheime Defaults und Allowlist
  tests/         # Einheit und Integration mit nachgestellten CLIs
```

Verantwortlichkeiten: `app` orchestriert, `tools` isoliert die CLIs, `gates` erzwingt die Sperren, `workbench` kapselt Läufe, `audit` dokumentiert.

## Minimales API-Gerüst

Nur Loopback. Keine Authentifizierung gegenüber dem Netz, weil der Dienst nicht nach außen gebunden ist; Annahme: der lokale Nutzer ist der Betreiber.

| Methode | Pfad | Zweck |
|---|---|---|
| `POST` | `/task` | Neue Aufgabe annehmen, Plan anstoßen, `task_id` zurückgeben |
| `GET` | `/status` | Übersicht offener und laufender Aufgaben |
| `GET` | `/status/{id}` | Status, Plan, wartende Freigaben einer Aufgabe |
| `POST` | `/approve/{id}` | Ausdrückliche menschliche Freigabe einer gesperrten Aktion |

Antworten bleiben sachlich (IDs, Status, Kurzgrund). Keine Secrets, keine Roh-Umgebung, keine Keychain-Werte.

## Gates, Allowlist und Audit

Hart gesperrt und nur nach ausdrücklicher menschlicher Bestätigung erlaubt:

- Geld (Zahlung, Kauf, Umbuchung und vergleichbare Auslöser)
- Senden (Nachrichten, Mails, Posts, externe Übermittlung)
- Löschen (Dateien, Repos, Remote-Objekte, destruktive Bereinigung)

Technik der Sperre: bevor ein Werkzeugstart oder ein Containerlauf beginnt, prüft der Gate-Layer die intendierte Aktion gegen Klassen und Allowlist. Ein nicht freigegebener Treffer wird abgebrochen und als `pending_approval` geführt. Die Allowlist nennt konkrete ausführbare Programme und erlaubte Unterbefehle; alles andere wird abgelehnt. Timeouts beenden hängende Subprozesse.

Audit: eine JSONL-Datei, append-only. Jede Zeile enthält mindestens Zeitstempel (Europe/Berlin), `task_id`, Akteur, Aktion, Entscheidung (erlaubt, blockiert, freigegeben, abgelehnt), Werkzeugname und einen SHA-256 über den kanonischen Zeileninhalt bzw. eine laufende Hash-Kette. Keine Geheimnisse, Tokens, Schlüssel oder vollständige Umgebungsvariablen in den Zeilen.

## Secrets

Zugangsdaten und Schlüssel liegen nur in der Umgebung oder in der Keychain des Rechners. Sie werden nicht geloggt, nicht in Audit-Zeilen, nicht in API-Antworten und nicht in Container als Host-Mount eingehängt. Benötigt ein Werkzeug im Container ein Geheimnis, gilt das als offene Frage für die Härtung; im MVP reicht die Regel „kein Secret-Mount, kein Secret-Log“.

## Phasen und Abnahmekriterien

### 1. MVP

- Dienst startet und bindet nur an `127.0.0.1`.
- `POST /task`, `GET /status` und `GET /status/{id}` funktionieren lokal.
- Qwen/Ollama liefert einen Plan (oder einen klaren Fehler, wenn Ollama fehlt).
- Mindestens ein Werkzeugadapter (Codex CLI oder Claude Code) ist per Subprozess mit Timeout angebunden.
- Geld, Senden und Löschen werden nicht ausgeführt, sondern als wartend markiert.
- Audit-JSONL entsteht ohne Secret-Inhalt.

### 2. Härtung

- Beide Hände sind angebunden; Aufrufe nur über Allowlist.
- Docker-Werkbank: Netzwerk standardmäßig aus, keine Secret-Host-Mounts.
- `POST /approve/{id}` gibt ausschließlich die bezeichnete, zuvor gesperrte Aktion frei; ohne diesen Schritt bleibt sie blockiert.
- Audit-Zeilen tragen SHA-256; nachträgliches Überschreiben ist nicht vorgesehen.

### 3. Tests

- Einheitstests: Gate-Klassen, Allowlist, Timeout-Abbruch, Freigabelogik.
- Integration mit nachgestellten CLIs und einem nachgestellten Ollama, ohne echte Fremdkonten.
- Prüfung, dass Audit-Dateien keine Secret-Muster enthalten und die Hash-Kette stimmt.
- Negativtests: Aufruf außerhalb der Allowlist, Bindung nicht an `0.0.0.0`, Lauf ohne Freigabe.

## Risiken und offene Fragen

- Öffentliches Repo für einen späteren Draft-PR: welches Repo, wer legt es an, und erst nach Max-Ja.
- Modellgröße und Hardware: welches Qwen-Modell unter Ollama auf der Zielmaschine sinnvoll läuft, ist ungeklärt; hier werden keine Leistungszahlen angenommen.
- Lizenz und Nutzungsbedingungen von Codex CLI und Claude Code: Klärung vor Nutzung und vor jedem öffentlichen Repo.
- Ausfall oder Drift der CLIs: Versionswechsel können Aufruf und Verhalten ändern; Adapter müssen eng und prüfbar bleiben.
- Zuständigkeit bei widersprüchlichen Plänen der beiden Hände: im MVP entscheidet der Chef (Qwen), Details offen.
- Ob und wie ein Container kontrolliert an ein Geheimnis kommt, ohne Host-Mount und ohne Log, ist in der Härtung zu klären.

Dieser Plan enthält keine Benchmarks und keine erfundenen Messwerte. Umsetzung, Draft-PR und Betrieb beginnen erst nach gesondertem Max-Ja.
