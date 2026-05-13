# HTML → PDF Rendering

Globale Konvention für reproduzierbare PDF- und PNG-Outputs aus HTML. Gilt für **alle Formate**: Slide-Decks (16:9, 4:3, 9:16), Print-Dokumente (A4, A5, Letter, A3, Flyer, Visitenkarten, Postkarten), Long-Form (Newsletter, Whitepaper, Angebote, Rechnungen) und Social-Visuals (LinkedIn-Post, Instagram-Story, X-Card, Reel).

## Pipeline-Entscheidung (zwei Pfade)

Es gibt **zwei legitime Render-Pfade**. Welcher passt, hängt vom Inhalt ab. Beide Pfade nutzen das Pflicht-CSS weiter unten — das ist die Grundlage in beiden Fällen.

### Entscheidungsbaum

| Render-Inhalt | Pfad | Warum |
|---|---|---|
| Slide-Deck mit Gradients, Ornamenten, komplexen Backgrounds, border-radius mit Hintergrund | **Pfad A: Screenshot-Assembly** | Rastert pixelgenau, vermeidet Chromium-Rendering-Bugs |
| Text-lastiger A4-Report, Whitepaper, Newsletter, Angebot, Rechnung, Long-Form | **Pfad B: `page.pdf()` (Vektor)** | Text bleibt vektoriell und damit gestochen scharf, kleinere Dateigröße, suchbar |
| Print-Flyer, Visitenkarte, Postkarte mit Hochaufflösungs-Anspruch | **Pfad A mit Template × 3** | Echtes 300 dpi für Druckerei |
| Social-Visual (LinkedIn-Post, Instagram, X-Card) als PNG | **Pfad A `--scale 2`** | PNG-Output, kein PDF nötig |

**Faustregel:** Inhalt überwiegend Text und einfaches Layout → Pfad B. Inhalt visuell komplex (Slides, Print-Visuals) → Pfad A.

### Pfad A: Screenshot-Assembly (visuell-komplex)

Playwright Screenshot pro Frame → PNG → pdf-lib. Über den **designer-Skill** (`~/.claude/skills/designer/`).

**Standard-Command:**

```bash
node ~/.claude/skills/designer/scripts/assemble-pdf.mjs \
  --inputs "./build/<name>-*.png" \
  --output "./out/<name>.pdf" \
  --format 16-9
```

**Scale-Wahl entscheidet über Schärfe:**

- `--scale 2` (144 dpi) → Screen, WhatsApp, Web-Preview
- `--scale 3` (216 dpi) → Print-tauglich (Default für gedruckte Slides)
- Template-Pixel × 3 + `--scale 1` → echtes 300 dpi für Druckerei

Selektoren: `.designer-slide`, `.designer-page`, `.designer-canvas` oder `.slide` / `.page`. Unterstützte `--format`-Werte: `a4`, `letter`, `a3`, `a5`, `16-9`, `4-3`, `9-16`, `dl`, `visitenkarte`, `postkarte`. Format-Katalog: `~/.claude/skills/designer/references/format-catalog.md`.

### Pfad B: `page.pdf()` (text-lastig, Vektor)

Direkter Chromium-PDF-Export. Output bleibt vektoriell — Text ist bei jeder Vergrößerung scharf, Datei klein, Inhalt suchbar und kopierbar.

**Voraussetzung:** Das Pflicht-CSS weiter unten ist im HTML aktiv. Ohne das CSS treten die 5 Killer-Gotchas auf (graue Ränder, fehlende Hintergrundfarben, etc.).

**Standard-Command via Playwright-Script:**

```bash
node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto('http://localhost:8765/report.html', { waitUntil: 'networkidle' });
  await page.waitForTimeout(1500);
  await page.pdf({
    path: 'out/report.pdf',
    printBackground: true,        // PFLICHT — Hintergründe drucken
    preferCSSPageSize: true,      // @page-Regeln respektieren
    margin: { top: 0, bottom: 0, left: 0, right: 0 }
  });
  await browser.close();
})();
"
```

**Pflicht-Flags:** `printBackground: true` (sonst druckt Chromium keine Backgrounds) und `preferCSSPageSize: true` (sonst ignoriert es dein `@page { size: ... }`).

## Workflow-Doktrin

1. **HTML im Browser iterieren** über einen lokalen HTTP-Server (`python3 -m http.server 8765`), nie direkt über `file://` — das blockiert Font- und Image-Loads. Visuelle Verifikation via Playwright open + screenshot.
2. **PDF wird nur auf Ansage gerendert** — nicht nach jeder HTML-Änderung. Spart Zeit, schützt vor Pipeline-Drift.
3. **Bei PDF-Bug oder visuellem Glitch:** zuerst `~/.claude/skills/designer/references/pdf-gotchas.md` lesen, dort 8 dokumentierte Bug-Patterns mit Fixes (Border-Radius-Artefakt, Print-Emulation-Overflow, Position-Override, FOUT/FOIT, Gradient-Banding, SVG-Race, Window-Size-Overhead, DPI-Mismatch).

## Pflicht-CSS für jedes HTML-Render-Template

In `<style>` oder externem CSS einbauen, bevor Layout-Spezifika kommen. Verhindert 80 % der typischen PDF-Bugs präventiv. Format-spezifisch: `@page`-Größe an Ziel-Format anpassen.

```css
/* Format-Beispiele — eins davon wählen, nicht alle gleichzeitig */
@page { size: 1920px 1080px; margin: 0; }  /* 16:9 Slide-Deck */
/* @page { size: A4; margin: 0; }            A4-Report / Flyer */
/* @page { size: 1080px 1920px; margin: 0; } 9:16 Reel / Story */
/* @page { size: 1080px 1080px; margin: 0; } 1:1 Social-Post */

* { box-sizing: border-box; }

.slide, .page, .designer-slide, .designer-page, .designer-canvas {
  position: relative;                       /* Anker für absolut positionierte Ornamente */
  overflow: hidden;                         /* hält Glow- und Arc-Elemente im Frame */
  background-clip: padding-box;
  -webkit-background-clip: padding-box;     /* Fix Chromium-Grey-Border-Bug bei border-radius */
  transform: translateZ(0);                 /* GPU-Layer, stabilisiert Gradients in PDF */
  page-break-after: always;
  break-after: page;
  -webkit-print-color-adjust: exact;
  print-color-adjust: exact;                /* zwingt Chromium Hintergründe zu drucken */
}

.slide:last-child, .page:last-child,
.designer-slide:last-child, .designer-page:last-child {
  page-break-after: auto;
  break-after: auto;
}

* { -webkit-font-smoothing: antialiased; }
```

**DPI-Skalierung:** Für 300-dpi-Print Template-Pixel × 3 (A4 wird dann `2480 × 3508` statt `794 × 1123`).

## Anti-Patterns (NICHT machen)

| Falsch | Richtig | Warum |
|---|---|---|
| `transform: scale(0.5)` zum Skalieren | exakte Pixel in `@page` setzen | scale ändert nur die Darstellung, nicht die Layout-Box → Browser paginiert weiter im alten Format |
| `background-attachment: fixed` | `background-attachment: scroll` oder weglassen | Chromium clippt fixed-Backgrounds im PDF |
| `<img src="logo.svg">` | Inline-SVG direkt im HTML einbetten | File-Loading-Race, SVG fehlt manchmal im PDF |
| JPEG-Logo auf Farbfläche | PNG mit Transparenz oder base64-data-URL | JPEG-Kompression erzeugt sichtbare Rahmen |
| `page.pdf()` **ohne** Pflicht-CSS und ohne `printBackground: true` | mit Pflicht-CSS plus `printBackground: true, preferCSSPageSize: true` | Sonst graue Ränder bei border-radius und fehlende Hintergrundfarben |
| `page.pdf()` für visuell komplexe Slides mit Gradients/Ornamenten | Screenshot-Assembly (Pfad A) | Vektor-Export kann Gradients fehlerhaft renderieren, Screenshot rastert deterministisch |
| nur `tr { page-break-inside: avoid }` | zusätzlich `tbody { page-break-inside: auto }` | sonst wird ganze Tabelle auf nächste Seite geschoben |

## Font-Handling

- Google Fonts via `<link>` im `<head>` laden
- Im Template: `document.fonts.ready.then(() => document.body.classList.add('fonts-loaded'))`
- Vor Screenshot: Playwright `eval` auf `document.body.classList.contains('fonts-loaded')` plus 800-1000 ms Sicherheits-Puffer gegen FOUT/FOIT
- In Playwright-Script: `await page.waitForLoadState('networkidle'); await page.waitForTimeout(1500)` vor dem Screenshot

## Bei Problemen

Sag der KI: **"PDF rendert mit <Symptom>"** — sie lädt `~/.claude/skills/designer/references/pdf-gotchas.md` und prüft das passende der 8 dokumentierten Patterns.

## Source of Truth

- Skill: `~/.claude/skills/designer/SKILL.md`
- Bug-Patterns (on-demand): `~/.claude/skills/designer/references/pdf-gotchas.md`
- Format-Katalog: `~/.claude/skills/designer/references/format-catalog.md`
- Style-Guide-Builder: `~/.claude/skills/designer/references/style-guide-builder.md`
- Assembly-Script: `~/.claude/skills/designer/scripts/assemble-pdf.mjs`
