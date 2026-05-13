# HTML → PDF Rendering global einrichten

Wenn du HTML-Inhalte als PDF oder PNG exportierst — egal ob **Slide-Deck (16:9, 4:3, 9:16), A4-Report, A5-Flyer, Visitenkarte, Newsletter oder LinkedIn-Visual** — und die Ergebnisse nicht sauber formatieren (graue Ränder, fehlende Hintergrundfarben, falsche Page-Breaks, Fonts laden zu spät), liegt das fast immer an denselben fünf Chromium-Bugs. Dieses Toolkit installiert eine globale Rule in deinem Claude Code Setup, die diese Bugs präventiv löst — für **jedes** Projekt, automatisch.

## Was wird installiert?

- Eine neue Datei `~/.claude/rules/html-to-pdf.md` mit Pflicht-CSS, Anti-Patterns, Pipeline-Konvention
- Eine Zeile in deiner globalen `~/.claude/CLAUDE.md`, die Claude auf die Rule verweist

Danach kennt jede neue Claude-Session in jedem Projekt die HTML→PDF-Regeln. Die Rule ergänzt den bestehenden **designer-Skill** (`~/.claude/skills/designer/`) — sie überschreibt nichts, sondern hebt die wichtigsten Präventionsregeln in den globalen Auto-Load-Layer, während die Detail-Diagnose (`pdf-gotchas.md`) on-demand im Skill bleibt.

## Installation: Prompt in Claude Code reinposten

Kopier diesen Block, öffne Claude Code (Terminal oder Cursor) und paste ihn als ersten Prompt in eine neue Session:

```
Bitte richte mir die HTML→PDF-Rule aus dem AI OS Community Repo global ein:

1. Lade den Inhalt von https://raw.githubusercontent.com/affm777/ai-os-community/main/toolkit/html-to-pdf/rule.md
2. Schreib ihn nach ~/.claude/rules/html-to-pdf.md (Datei neu anlegen)
3. Öffne ~/.claude/CLAUDE.md, finde die Tabelle unter "## Regeln (Details)" und ergänze als letzte Zeile:
   | HTML → PDF Rendering | `~/.claude/rules/html-to-pdf.md` |
4. Zeig mir am Ende den Diff der CLAUDE.md-Änderung zur Bestätigung.
```

Claude erledigt den Rest. Nach Abschluss: einmal neue Session starten, fertig.

## Verifikation

Nach der Installation, in einem beliebigen Projekt mit HTML-Inhalt:

1. Neue Claude-Session
2. Sag: "Render mir aus dieser HTML-Datei ein PDF"
3. Claude sollte den **designer-Skill** nutzen (Screenshot-Assembly), nicht direktes `page.pdf()`
4. Hintergrundfarben sind im PDF da, keine grauen Rahmen um border-radius-Elemente, Page-Breaks sitzen, Fonts korrekt geladen

## Bei Problemen

Falls trotz Rule ein PDF-Bug auftritt: sag Claude **"PDF rendert mit <Symptom>"** — Claude lädt automatisch die detaillierte Bug-Liste aus `~/.claude/skills/designer/references/pdf-gotchas.md` und prüft die acht bekannten Patterns (Border-Radius-Artefakt, Print-Emulation-Overflow, Position-Override, FOUT/FOIT, Gradient-Banding, SVG-Race, Window-Size-Overhead, DPI-Mismatch).

## Voraussetzung

Du hast den **designer-Skill** installiert (kommt automatisch über das `ai-os-starter`-Bootstrap mit). Prüfen: `ls ~/.claude/skills/designer/SKILL.md` — wenn da, alles gut. Falls fehlt: erst Bootstrap nachziehen, dann diese Rule installieren.
