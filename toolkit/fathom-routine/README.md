# Fathom-Sync als Routine im Claude Code einrichten

Du hattest deinen Fathom-Sync bisher als **Scheduled Task in Claude Cowork** laufen. Diese Anleitung migriert ihn auf eine **Routine im Claude Code** (Reiter in der Claude-Desktop-App).

## Warum migrieren?

Claude Cowork und Claude Code haben **getrennte Skill-Bibliotheken**.

Alles, was du als Skill in `~/.claude/` angelegt hast (z.B. `brain:sync-meetings`), ist erreichbar ueber Claude Code — also im Terminal, in Cursor, und im Reiter „Code" der Claude-Desktop-App. In **Claude Cowork** ist diese Skill-Bibliothek **nicht verfuegbar**.

Was bisher passiert ist: der Scheduled Task hat den Prompt-Text „bitte fuehre brain:sync-meetings aus" bekommen, Claude in Cowork hat den Text **interpretiert** und oft das Richtige getan, aber nicht den tatsaechlichen Skill aufgerufen. Funktioniert zufaellig, nicht zuverlaessig.

**Loesung:** Routine im Claude Code. Dort wird der Skill **deterministisch** aufgerufen, weil die Skill-Bibliothek im selben Kontext liegt.

## Schritt 1: Alte Scheduled Task in Claude Cowork loeschen

1. claude.ai im Browser oeffnen, in den Bereich **Cowork** wechseln
2. Zu **Scheduled Tasks** navigieren
3. Den Fathom-Sync-Task finden
4. **Loeschen**

Damit verhindert, dass spaeter zwei Syncs parallel laufen und Duplikate erzeugen.

## Schritt 2: Routine im Claude Code anlegen

1. Claude-Desktop-App oeffnen
2. Oben auf den Reiter **Code** wechseln (neben „Chat" und „Cowork")
3. Links in der Seitenleiste auf **Routines**
4. Oben rechts auf **Neue Routine** → **Lokal**

![Routinen-Uebersicht mit Neue-Routine-Dropdown](screenshot-1-routines.png)

## Schritt 3: Routine ausfuellen

Es oeffnet sich ein Dialog „Routine bearbeiten". Folgende Felder ausfuellen:

![Routine-bearbeiten-Dialog](screenshot-2-routine-form.png)

| Feld | Wert |
|---|---|
| **Name** | `fathom-sync` (oder ein Name deiner Wahl) |
| **Beschreibung** | `Taeglicher autonomer Fathom-Sync: neue Meetings aus Fathom MCP nach Obsidian-Vault 01_Inbox/, mit Personen-Cross-Linking und Projekt-Matching.` |
| **Anweisungen** | `/brain:sync-meetings scheduled` |
| **Auto-Modus** | aktivieren |
| **Verzeichnis** | `~/Documents/Second-Brain` auswaehlen |
| **Modell** | Sonnet 4.6 |
| **Zeitplan** | **Taeglich**, z.B. um **10:30** (waehle eine Zeit, zu der dein Rechner laeuft) |

Auf **Speichern** klicken.

## Schritt 4: Testlauf

1. Die neu angelegte Routine in der Liste anklicken
2. **Jetzt ausfuehren** (oben rechts)
3. Den Chat-Verlauf beobachten:
   - Sollte ohne Permission-Prompts durchlaufen
   - Am Ende: Liste der neu synchronisierten Meetings
   - Neue Meetings landen in `~/Documents/Second-Brain/01_Inbox/`

Wenn alles glatt durchgelaufen ist: **Migration erfolgreich**. Du bist fertig.

## Schritt 5 (Optional): Permission-Fallback

Manchmal fragt Claude trotz aktiviertem Auto-Modus nach Berechtigungen fuer einzelne Tools. Falls dir das im Testlauf passiert: kopiere den folgenden Block 1:1 in einen frischen Claude-Code-Chat. Claude ergaenzt die fehlenden Permissions selbst in deiner `~/.claude/settings.json`, ohne dass du selbst editieren musst.

````markdown
Ich brauche deine Hilfe, meine Claude-Code-Settings sauber zu erweitern.

Bitte:

1. Lies `~/.claude/settings.json`.
2. Pruefe das `permissions.allow`-Array.
3. Ergaenze die folgenden Eintraege darin. Wichtig: **keine Doppelungen erzeugen, keine bestehenden Eintraege ueberschreiben, keine Kollisionen verursachen**. Wenn ein Eintrag schon da ist, ueberspringen.

```json
[
  "Read(~/Documents/Second-Brain/**)",
  "Glob(~/Documents/Second-Brain/**)",
  "Grep(~/Documents/Second-Brain/**)",
  "Bash(ls ~/Documents/Second-Brain/**)",
  "Bash(find ~/Documents/Second-Brain/**)",
  "Bash(wc -l ~/Documents/Second-Brain/**)",
  "Bash(tail:*)",
  "Bash(head:*)",
  "Bash(awk:*)",
  "Bash(mkdir:*)",
  "Bash(date:*)",
  "Bash(grep:*)",
  "Read(~/.claude/project-repos.yaml)",
  "Write(~/Documents/Second-Brain/01_Inbox/**)",
  "Edit(~/Documents/Second-Brain/01_Inbox/**)",
  "Edit(~/Documents/Second-Brain/00_Meta/system/.last-fathom-sync)",
  "Edit(~/Documents/Second-Brain/00_Meta/system/vault-log.md)",
  "Edit(~/Documents/Second-Brain/00_Meta/system/vault-index.md)",
  "mcp__claude_ai_Fathom__list_meetings",
  "mcp__claude_ai_Fathom__get_identity",
  "mcp__claude_ai_Fathom__get_meeting_summary",
  "mcp__claude_ai_Fathom__get_meeting_transcript",
  "mcp__claude_ai_Fathom__get_recording_by_url",
  "mcp__claude_ai_Fathom__get_recording_by_call_id",
  "mcp__claude_ai_Fathom__search_meetings",
  "mcp__claude_ai_Fathom__find_person"
]
```

4. Validiere die Datei danach mit `jq . ~/.claude/settings.json > /dev/null`. Wenn die Validierung fehlschlaegt, korrigiere den Syntaxfehler.
5. Gib mir am Ende eine kurze Liste: welche Eintraege hast du neu ergaenzt, welche waren schon da.

Danach beende ich die Claude-Desktop-App komplett (Cmd+Q) und starte sie neu, damit die Settings geladen werden. Dann teste ich die Routine erneut.
````

Nach Abschluss: Claude-Desktop-App **komplett beenden** (macOS: `Cmd+Q`, nicht nur Fenster schliessen) und neu oeffnen. Dann Testlauf aus Schritt 4 wiederholen.

## Wenn etwas nicht klappt

**Keine neuen Meetings im Inbox**
- Fathom-Connector aktiv? In claude.ai unter `/connectors` pruefen
- Im Routine-Chat-Verlauf nach Fehlermeldungen suchen

**Neue Meetings landen nicht in `01_Inbox/`, sondern in `02_Projects/`**
- Datei manuell zurueck nach `~/Documents/Second-Brain/01_Inbox/` verschieben
- Einsortieren in den richtigen Projekt-Ordner macht spaeter `/brain:sort-inbox`, nicht die Sync-Routine

**Trotz Schritt 5 immer noch Permission-Prompts**
- Hast du die App **komplett beendet** (Cmd+Q) und neu gestartet? Settings werden nur beim App-Start geladen.
