# Fathom-Sync als Claude-Code-Routine einrichten

Migration des Fathom-Meeting-Sync-Workflows von **Scheduled Tasks in Claude Cowork** auf **Routines im Claude Code** (Reiter in der Claude-Desktop-App).

## Warum migrieren?

**Claude Cowork und Claude Code haben getrennte Skill-Bibliotheken.**

Alles, was du als Skill in `~/.claude/skills/` bzw. `~/.claude/commands/` angelegt hast (z.B. `brain:sync-meetings`), ist erreichbar ueber:

- Claude Code im **Terminal**
- Claude Code in **Cursor**
- Claude Code als **Reiter in der Claude-Desktop-App**

In **Claude Cowork** (wo Scheduled Tasks bisher liefen) ist diese Skill-Bibliothek **nicht verfuegbar**. Was bisher passiert ist: der Scheduled Task hat einen Prompt-Text wie „bitte fuehre brain:sync-meetings aus" bekommen, Claude in Cowork hat den Text **interpretiert** und oft das Richtige getan — aber nicht den tatsaechlichen Skill aufgerufen. Das ist zufaellig und fragil.

**Die Loesung:** Routines im Claude Code nutzen. Dort wird der Skill **deterministisch** aufgerufen, weil die Skill-Bibliothek im selben Kontext liegt.

**Was du brauchst:**

- macOS oder Windows mit Claude-Desktop-App installiert (Reiter „Claude Code")
- Fathom-Connector aktiviert und in Claude Code als MCP-Server verbunden
- Vault unter `~/Documents/Second-Brain/` (Standard aus dem Bootstrap)
- Skill `brain:sync-meetings` in `~/.claude/commands/brain/sync-meetings.md` (kommt aus dem Bootstrap)

## Schritt 1: Settings.json um Permissions ergaenzen

Damit die Routine ohne Permission-Prompts durchlaeuft, muessen alle Tools, die der Skill verwendet, in `~/.claude/settings.json` explizit erlaubt werden.

Oeffne `~/.claude/settings.json` und ergaenze im `permissions.allow`-Array folgende Eintraege:

```json
{
  "permissions": {
    "allow": [
      "Read(~/Documents/Second-Brain/**)",
      "Glob(~/Documents/Second-Brain/**)",
      "Grep(~/Documents/Second-Brain/**)",
      "Bash(ls ~/Documents/Second-Brain/**)",
      "Bash(find ~/Documents/Second-Brain/**)",
      "Bash(wc -l ~/Documents/Second-Brain/**)",
      "Bash(tail:*)",
      "Bash(head:*)",
      "Bash(awk:*)",

      "Read(~/.claude/project-repos.yaml)",

      "Write(~/Documents/Second-Brain/01_Inbox/**)",
      "Edit(~/Documents/Second-Brain/01_Inbox/**)",
      "Edit(~/Documents/Second-Brain/00_Meta/system/.last-fathom-sync)",
      "Edit(~/Documents/Second-Brain/00_Meta/system/vault-log.md)",
      "Edit(~/Documents/Second-Brain/00_Meta/system/vault-index.md)",

      "Bash(mkdir:*)",
      "Bash(date:*)",
      "Bash(grep:*)",

      "mcp__claude_ai_Fathom__list_meetings",
      "mcp__claude_ai_Fathom__get_identity",
      "mcp__claude_ai_Fathom__get_meeting_summary",
      "mcp__claude_ai_Fathom__get_meeting_transcript",
      "mcp__claude_ai_Fathom__get_recording_by_url",
      "mcp__claude_ai_Fathom__get_recording_by_call_id",
      "mcp__claude_ai_Fathom__search_meetings",
      "mcp__claude_ai_Fathom__find_person"
    ]
  }
}
```

**Was jeder Block bewirkt:**

| Block | Zweck |
|---|---|
| `Read/Glob/Grep` auf Vault | Skill liest Templates, bestehende Meetings, Sync-Timestamp |
| `Bash ls/find/wc/tail/head/awk` | Skill scannt Inbox und prueft TSV-Index |
| `Read(~/.claude/project-repos.yaml)` | Mapping Projekt-Slug → Repo-Pfad fuer Meeting-Propagation |
| `Write(01_Inbox/**)` | Neue Meeting-Files in der Inbox anlegen |
| `Edit(01_Inbox/**)` | Bestehende Inbox-Files updaten (Dedup-Warning) |
| `Edit(.last-fathom-sync)` | Inkrementeller Timestamp-Write nach jedem Meeting |
| `Edit(vault-log.md` + `vault-index.md)` | Sweep-Logs ohne Prompt |
| `Bash mkdir/date/grep` | Sub-Ordner anlegen, ISO-Timestamps, Pipe-Greps |
| `mcp__claude_ai_Fathom__*` | Alle 8 Read-Tools des Fathom-Connectors (alle read-only) |

**Bewusst NICHT erlaubt:** Writes ausserhalb von Inbox + system-Files. Wenn der Skill versucht, direkt in `02_Projects/` zu schreiben, bleibt das geblockt — gewollt, weil neue Files erst durch `/brain:sort-inbox` einsortiert werden sollen.

## Schritt 2: JSON-Validitaet pruefen

Im Terminal:

```bash
jq . ~/.claude/settings.json > /dev/null && echo "JSON OK"
```

Wenn keine Ausgabe `JSON OK`: JSON kaputt, Syntax pruefen (vergessenes Komma o.ae.).

## Schritt 3: Claude Code komplett neu starten

**Wichtig:** Settings.json wird nur beim Prozess-Start gelesen, nicht pro Chat.

1. Alle offenen Claude-Code-Chats schliessen
2. **Claude-Desktop-App komplett beenden** (macOS: `Cmd+Q`, nicht nur Fenster schliessen)
3. App neu oeffnen

Nach dem Neustart sind die neuen Permissions aktiv.

## Schritt 4: Alte Scheduled Task in Claude Cowork loeschen

1. claude.ai im Browser oeffnen, in den Bereich **Claude Cowork** wechseln
2. Zu **Scheduled Tasks** navigieren
3. Den Task fuer Fathom-Sync identifizieren (typisch: cron-Schedule mit Aufruf von `brain:sync-meetings` oder aehnlichem Prompt)
4. **Loeschen**

Damit verhindert, dass spaeter zwei parallele Syncs laufen und Duplikate erzeugen.

## Schritt 5: Routine im Claude Code anlegen

1. In der Claude-Desktop-App in den Reiter **Claude Code** wechseln, dann zu **Routines**
2. **New Routine** klicken
3. Konfiguration:
   - **Name:** `fathom-sync-daily` (oder eigener Name)
   - **Schedule:** Cron-Pattern, z.B. `0 9 * * *` fuer taeglich 9:00 Uhr, oder `0 */4 * * *` fuer alle 4 Stunden
   - **Prompt:** `/brain:sync-meetings`
   - **Permission Mode:** auf `auto` / `acceptEdits` setzen (UI-Bezeichnung kann variieren)
4. Speichern

**Cron-Beispiele:**

| Pattern | Bedeutung |
|---|---|
| `0 9 * * *` | Taeglich 9:00 |
| `0 9 * * 1-5` | Mo-Fr 9:00 |
| `0 */4 * * *` | Alle 4 Stunden |
| `0 9,17 * * *` | Taeglich 9:00 und 17:00 |

## Schritt 6: Test-Run

1. In der Routines-Uebersicht den neuen Task auswaehlen
2. **Run now** ausfuehren
3. Den Chat-Verlauf beobachten:
   - Sollte ohne Permission-Prompts durchlaufen
   - Output am Ende: `Fathom-Sync abgeschlossen — <Datum>` mit Liste der neuen Meetings
   - Neue Meetings landen in `~/Documents/Second-Brain/01_Inbox/`

**Wenn doch noch ein Prompt kommt:** Tool-Name aus dem Prompt notieren und in `settings.json` ergaenzen. Typische Nachzuegler:

- `Bash(cat:*)` falls der Skill `cat` statt `Read` nutzt
- Andere MCP-Server-IDs falls Fathom-Connector unter anderem Namen registriert ist

## Schritt 7: Wenn trotz Auto-Mode noch Permission-Prompts kommen

Bei einigen TN tauchen auch nach Schritt 1-6 vereinzelt noch Permission-Prompts auf, obwohl der Auto-Mode aktiv ist. Grund: das UI-Toggle „Auto-Mode" ist pro Chat-Session, die `allow`-Liste in `settings.json` greift erst, wenn alle Tools dort exakt gelistet sind.

**Fallback:** Statt selbst in der `settings.json` zu suchen, gibst du Claude Code den folgenden Prompt — Claude vergleicht die Permissions mit dem, was der Skill braucht, und ergaenzt fehlende Eintraege automatisch:

```
Du bist mein Setup-Assistent. Bitte fuehre folgenden Plan aus:

1. Lies ~/.claude/settings.json
2. Lies ~/.claude/commands/brain/sync-meetings.md
3. Identifiziere alle Tools (Read, Write, Edit, Glob, Grep, Bash-Commands, MCP-Tools), die der Skill verwendet
4. Vergleiche mit dem permissions.allow-Array in settings.json
5. Ergaenze fehlende Eintraege so, dass der Skill ohne Prompts durchlaufen kann
6. Validiere die JSON-Struktur mit `jq . ~/.claude/settings.json > /dev/null`
7. Sag mir, was du ergaenzt hast und ob ein App-Neustart noetig ist
```

Diesen Block kopierst du 1:1 in einen frischen Claude-Code-Chat. Nach Abschluss: Claude Code komplett neu starten (Cmd+Q, dann oeffnen), dann Test-Run erneut ausfuehren.

## Schritt 8: Verifikation nach 1-2 Tagen

Nach 1-2 Routine-Laeufen pruefen:

```bash
# Anzahl neue Meeting-Files in Inbox
ls ~/Documents/Second-Brain/01_Inbox/*meeting*.md | wc -l

# Letzter Sync-Timestamp
cat ~/Documents/Second-Brain/00_Meta/system/.last-fathom-sync

# Sweep-Log Eintraege
tail -10 ~/Documents/Second-Brain/00_Meta/system/vault-log.md
```

Wenn neue Meetings in Inbox auftauchen und Timestamp aktualisiert ist: **Migration erfolgreich**.

## Troubleshooting

**Trotz Settings noch Permission-Prompts**

- Claude Code komplett neu gestartet? (Cmd+Q, nicht nur Fenster)
- Ist `jq . ~/.claude/settings.json > /dev/null` ohne Fehler durchgelaufen?
- Tool-Name im Prompt exakt mit `allow`-Eintrag abgeglichen? (Case-sensitive)

**Routine laeuft, aber keine neuen Meetings**

- Fathom-Connector in claude.ai aktiv? Pruefen via `/connectors` in claude.ai
- Letzter Sync-Timestamp aktuell? Falls zu weit in der Zukunft → `.last-fathom-sync` auf ein Datum vor letztem Meeting setzen
- Pruefe Output des Routine-Chats auf Fehlermeldungen

**Neue Meetings landen direkt in `02_Projects/`, nicht in Inbox**

- Skill-Protokoll-Verletzung — gehoert in Inbox
- Files manuell zurueck nach `~/Documents/Second-Brain/01_Inbox/` verschieben
- Sicherstellen, dass `Write(~/Documents/Second-Brain/01_Inbox/**)` in `allow` steht und KEINE breitere `Write(~/Documents/Second-Brain/**)`-Regel die Doktrin aushebelt

**Permission-Mode-Toggle im Chat steht nach Routine-Run nicht auf Auto**

- Bekannter UX-Bug: UI-Toggle ist pro Chat-Session, nicht global
- Manuelles Re-Toggeln nicht noetig — die allow-Liste in `settings.json` greift unabhaengig vom UI-Toggle
- Falls trotzdem Prompts kommen: liegt am UI, nicht an den Settings, einmal auf Auto klicken reicht fuer den jeweiligen Chat

## Zusaetzlicher Hinweis fuer `/brain:sort-inbox`

Wenn du den Sync produktiv nutzt, sammeln sich Meeting-Files in der Inbox. Einsortieren passiert NICHT durch die Routine, sondern interaktiv via `/brain:sort-inbox` (typisch woechentlich). Die hier gesetzten Permissions decken die Read-Seite ab; interaktiver Einsortier-Lauf prompted bewusst fuer Moves nach `02_Projects/` und Hub-Updates — diese Prompts sind gewollt, weil semantische Entscheidungen anstehen.
