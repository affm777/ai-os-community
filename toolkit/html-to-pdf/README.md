# HTML → PDF Rendering global einrichten

Wenn du HTML-Folien, A4-Reports oder Flyer als PDF exportierst und die Ergebnisse formatieren nicht sauber (graue Raender, Hintergrundfarben fehlen, Page-Breaks an falschen Stellen, Fonts laden zu spaet), liegt das fast immer an denselben 5 Chromium-Bugs. Dieses Toolkit installiert eine globale Rule in deinem Claude Code Setup, die diese Bugs praeventiv loest — fuer **jedes** Projekt, automatisch.

## Was wird installiert?

- Eine neue Datei `~/.claude/rules/html-to-pdf.md` mit Pflicht-CSS, Anti-Patterns, Pipeline-Konvention
- Eine Zeile in deiner globalen `~/.claude/CLAUDE.md`, die Claude auf die Rule verweist

Danach kennt jede neue Claude-Session in jedem Projekt die HTML→PDF-Regeln.

## Installation: Prompt in Claude Code reinposten

Kopier diesen Block, oeffne Claude Code (Terminal oder Cursor) und paste ihn als ersten Prompt in eine neue Session:

```
Bitte richte mir die HTML→PDF-Rule aus dem AI OS Community Repo global ein:

1. Lade den Inhalt von https://raw.githubusercontent.com/affm777/ai-os-community/main/toolkit/html-to-pdf/rule.md
2. Schreib ihn nach ~/.claude/rules/html-to-pdf.md (Datei neu anlegen)
3. Oeffne ~/.claude/CLAUDE.md, finde die Tabelle unter "## Regeln (Details)" und ergaenze als letzte Zeile:
   | HTML → PDF Rendering | `~/.claude/rules/html-to-pdf.md` |
4. Zeig mir am Ende den Diff der CLAUDE.md-Aenderung zur Bestaetigung.
```

Claude erledigt den Rest. Nach Abschluss: einmal neue Session starten, fertig.

## Verifikation

Nach der Installation, in einem beliebigen Projekt mit HTML-Folien:

1. Neue Claude-Session
2. Sag: "Render mir aus dieser HTML-Datei ein PDF"
3. Claude sollte den **designer-Skill** nutzen (Screenshot-Assembly), nicht direktes `page.pdf()`
4. Hintergrundfarben sind im PDF da, keine grauen Rahmen um border-radius-Elemente, Page-Breaks sitzen

## Bei Problemen

Falls trotz Rule ein PDF-Bug auftritt: sag Claude **"PDF rendert mit <Symptom>"** — Claude laedt automatisch die Detail-Bug-Liste aus `~/.claude/skills/designer/references/pdf-gotchas.md` und prueft die 8 bekannten Patterns.

## Voraussetzung

Du hast den **designer-Skill** installiert (kommt automatisch ueber das `ai-os-starter`-Bootstrap mit). Pruefen: `ls ~/.claude/skills/designer/SKILL.md` — wenn da ist, alles gut.
