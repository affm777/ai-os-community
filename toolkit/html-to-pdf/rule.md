# HTML → PDF Rendering

Globale Konvention fuer reproduzierbare PDF-Outputs aus HTML — gilt fuer Slide-Decks, A4-Reports, Flyer, Social-Media-Visuals, alle Print-Deliverables.

## Pipeline (Pflicht)

PDFs aus HTML werden ueber den **designer-Skill** gerendert (Bootstrap-Standard, `~/.claude/skills/designer/`). Der Skill nutzt **Screenshot-Assembly** (Playwright → PNG → pdf-lib), nicht Chromium `page.pdf()`.

**Grund:** `page.pdf()` produziert Artefakte bei border-radius + Hintergrund, ignoriert Hintergrundfarben by default, repaginatiert unter dem Hub bei `position: fixed`. Screenshot-Assembly rasterisiert pro Slide pixel-genau und assembliert deterministisch.

**Standard-Command (16:9 Slides, 144dpi):**

```bash
node ~/.claude/skills/designer/scripts/assemble-pdf.mjs \
  --inputs "./build/<name>-*.png" \
  --output "./out/<name>.pdf" \
  --format 16-9
```

Vorgelagert: Playwright screenshot pro `.designer-slide` (oder `.slide`, `.designer-page`, `.page`). Unterstuetzte Formate im `--format`-Flag: `a4`, `letter`, `a3`, `a5`, `16-9`, `4-3`, `9-16`, `dl`, `visitenkarte`, `postkarte`.

## Workflow-Doktrin

1. **HTML im Browser iterieren** — live-reload via lokalem HTTP-Server (`python3 -m http.server 8765`, nie `file://` direkt, blockiert Font/Image-Loads). Visuelle Verifikation per Playwright open + screenshot.
2. **PDF wird nur auf Ansage gerendert** — nicht nach jeder HTML-Aenderung. Spart Zeit, schuetzt vor Pipeline-Drift.
3. **Bei PDF-Bug oder visuellem Glitch:** zuerst `~/.claude/skills/designer/references/pdf-gotchas.md` lesen, dort 8 bekannte Bug-Patterns mit Fixes.

## Pflicht-CSS fuer HTML-Folien

In `<style>` oder externem CSS einbauen, bevor Layout-Spezifika kommen. Loest 80% der typischen PDF-Bugs praeventiv.

```css
@page { size: 1920px 1080px; margin: 0; }   /* exakte Pixel, NIEMALS transform: scale() */

* { box-sizing: border-box; }

.slide, .page, .designer-slide, .designer-page {
  position: relative;                       /* Anker fuer absolut positionierte Ornamente */
  overflow: hidden;                         /* haelt Glow/Arc-Elemente in der Frame */
  background-clip: padding-box;
  -webkit-background-clip: padding-box;     /* Fix Chromium-Grey-Border-Bug bei border-radius */
  transform: translateZ(0);                 /* GPU-Layer, stabilisiert Gradients in PDF */
  page-break-after: always;
  break-after: page;
  -webkit-print-color-adjust: exact;
  print-color-adjust: exact;                /* zwingt Chromium Backgrounds zu drucken */
}

.slide:last-child, .page:last-child,
.designer-slide:last-child, .designer-page:last-child {
  page-break-after: auto;
  break-after: auto;
}

* { -webkit-font-smoothing: antialiased; }
```

Fuer A4-Reports: `@page { size: A4; margin: 0; }` und Container-Dimension `794px × 1123px` bei 96dpi, oder `2480px × 3508px` bei 300dpi (Template-Pixel 3× skalieren).

## Anti-Patterns (NICHT machen)

| Falsch | Richtig | Warum |
|---|---|---|
| `transform: scale(0.5)` zum Skalieren | exakte Pixel in `@page` setzen | scale aendert nur die Darstellung, nicht die Layout-Box → Browser paginatiert weiter als A4 |
| `background-attachment: fixed` | `background-attachment: scroll` oder weglassen | Chromium clipt fixed-Backgrounds im PDF |
| `<img src="logo.svg">` | Inline-SVG direkt im HTML einbetten | File-Loading-Race, SVG fehlt manchmal im PDF |
| JPEG-Logo auf Farbflaeche | PNG mit Transparenz + base64-data-URL | JPEG-Kompression erzeugt sichtbare Rahmen |
| `page.pdf()` direkt | Screenshot-Assembly via `assemble-pdf.mjs` | Chromium `page.pdf()` hat Border-Radius-Bug und Background-Color-Default-Off |
| `<tr style="page-break-inside: avoid">` allein | `tr { page-break-inside: avoid }` + `tbody { page-break-inside: auto }` | sonst wird ganze Tabelle auf naechste Seite geschoben |

## Font-Handling

- Google Fonts via `<link>` im `<head>` laden
- Pre-Screenshot: `document.fonts.ready` checken, plus 800-1000ms sleep als Sicherheits-Puffer gegen FOUT/FOIT
- In Playwright: `await page.waitForLoadState('networkidle'); await page.waitForTimeout(1500)` vor dem Screenshot

## Bei Problemen

Sag der KI: **"PDF rendert mit <Symptom>"** — sie laedt `~/.claude/skills/designer/references/pdf-gotchas.md` und prueft das passende der 8 dokumentierten Patterns:

1. Grey-Border-Artefakt (border-radius + background)
2. Backgrounds werden nicht gedruckt
3. Ornamente shiften (absolute Positionierung)
4. FOUT/FOIT (Font-Fallback im ersten PNG)
5. Gradient-Banding
6. SVG fehlt (File-Race)
7. Window-Size-Viewport-Overhead (~88px Cutoff)
8. DPI-Mismatch (blurry print)

## Source of Truth

- Skill: `~/.claude/skills/designer/SKILL.md`
- Bug-Patterns: `~/.claude/skills/designer/references/pdf-gotchas.md`
- Format-Katalog: `~/.claude/skills/designer/references/format-catalog.md`
- Assembly-Script: `~/.claude/skills/designer/scripts/assemble-pdf.mjs`
