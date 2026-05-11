# Fathom-Sync als Claude-Code-Routine einrichten

Migration der automatisierten Fathom-Meeting-Sync von Cloud-Scheduled-Tasks (claude.ai) auf Routines in der Claude-Code-Desktop-App.

## Warum migrieren?

**Cloud-Scheduled-Tasks laufen in der Cloud** und haben **keinen Zugriff auf deine lokalen Claude-Code-Skills** (z.B. `brain:sync-meetings`). Routinen in der Claude-Code-Desktop-App laufen lokal auf deinem Rechner und greifen damit auf deine komplette Skill-Bibliothek zu.

**Was du brauchst:**

- macOS oder Windows mit Claude-Code-Desktop-App installiert
- Fathom-Connector in claude.ai aktiviert und im Claude Code als MCP-Server verbunden
- Vault unter `~/Documents/Second-Brain/` (Standard aus `ai-os-starter`)
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
2. **Claude-Code-Desktop-App komplett beenden** (macOS: `Cmd+Q`, nicht nur Fenster schliessen)
3. App neu oeffnen

Nach dem Neustart sind die neuen Permissions aktiv.

## Schritt 4: Alte Cloud-Scheduled-Task loeschen

1. claude.ai im Browser oeffnen
2. Zu Scheduled Tasks navigieren
3. Den Task fuer Fathom-Sync identifizieren (typisch: cron-Schedule mit Aufruf von `brain:sync-meetings` oder aehnlichem Prompt)
4. **Loeschen**

Damit verhindert, dass spaeter zwei parallele Syncs laufen und Duplikate erzeugen.

## Schritt 5: Routine in der Desktop App anlegen

1. In der Claude-Code-Desktop-App: rechts oben auf den Settings/Profil-Bereich, dann zu **Routines** wechseln
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

## Schritt 7: Verifikation nach 1-2 Tagen

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
