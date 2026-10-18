# ANSIpants File Formats

Oct 10, 2026.

What each ANSIpants file holds, field by field, read from the 16M source.

## At a glance

ANSIpants has two formats of its own, both plain JSON: `.ansipants` for drawings and `.ansipresets` for preset banks. Everything else it writes is a standard format other programs already read.

| Format | What it holds | Written by | Read by |
| --- | --- | --- | --- |
| `.ansipants` | A whole drawing: every layer, the presets, document settings, SAUCE | App, 16M | App, 16M, 16 |
| `.ansipresets` | A bank of 16 preset slots | App, 16M, 16 | App, 16M, 16 |
| `.ans` | The flattened picture as text plus escape codes, with a SAUCE record | All three | All three, and any ANSI viewer |
| `.utf8.ans` | The same picture in UTF-8 instead of CP437 | All three | Any modern terminal |
| `.png`, `.1080.png` | A rendered picture in the VGA font | All three | Anything |
| `.txt` | Characters only, no color | All three | Anything |

16 can read `.ansipants` but never writes one: its drawings have one layer, so `.ans` holds everything. The terminal editions also keep your working preset bank at `~/.config/ansipants/presets.json` (under $XDG\_CONFIG\_HOME instead when that is set), in the same shape as an `.ansipresets` file.

## .ansipants

An `.ansipants` file is one JSON object: the drawing's size, its layers, and optional settings. It is written with Swift's plain `JSONEncoder`, with no pretty-printing and keys in declaration order, matching the app byte for byte. A field that is absent or `null` is simply left out. The Swift type is `DrawingFile` in `Core/Layers.swift` (16M) and `DocumentSupport.swift` (app).

### A minimal file

A one-layer, 2×1 drawing holding a red `A`, laid out for reading (real files are one line):

```json
{
  "version": 5,
  "columns": 2,
  "rows": 1,
  "colorDepth": "c16",
  "layers": [{
    "id": "6F1C2B9E-0A4D-4E8B-9C3F-2D7A5B1E8C40",
    "name": "Layer 1",
    "visible": true,
    "cells": [[
      {"character": "A", "foreground": {"type": "ansi16", "n": 9}, "background": {"type": "ansi16", "n": 0}},
      {"character": " ", "foreground": {"type": "ansi16", "n": 7}, "background": {"type": "ansi16", "n": 0}}
    ]],
    "rowOffset": 0,
    "colOffset": 0,
    "kind": {"type": "drawing"},
    "historyEnabled": false
  }],
  "backgroundMode": "brightBackgrounds"
}
```

### Top-level fields

| Field | Type | Since | Meaning |
| --- | --- | --- | --- |
| `version` | number | v1 | Format version; writers today write 5 |
| `columns`, `rows` | number | v1 | Canvas size. Readers clamp to 1–1000 columns and 1–4000 rows |
| `cells` | grid of cells | v1 | The whole picture, before layers existed. Written only by v1; read as one layer |
| `layers` | array of layers | v2 | The layer stack, top layer first. See the next section |
| `presets` | array of 16 presets or `null` | v2 | The preset bank, in `.ansipresets` shape. Loading it is optional for the reader |
| `referenceImageData` | base64 image | v3 | The app's tracing image. With `referenceVisible`, `referenceExcludedFromExport`, `referenceOffsetX`, `referenceOffsetY`, `referenceScale` |
| `charCycleSequence` | string | v3 | The characters a character cycle steps through |
| `colorDepth` | `"mono"`, `"c16"`, `"c256"`, `"truecolor"` | v4 | Which colors the drawing allows. Absent means 16 |
| `compoMode` | boolean | v4 | The app's Classic switch: 16 colors and CP437 only. 16M reads files with it but writes it back as absent |
| `backgroundMode` | `"brightBackgrounds"` or `"blinkingText"` | v5 | What the shared DOS attribute bit means. Absent means bright backgrounds |
| `monoTint` | string | v5 | The monochrome tint: white, green, amber or inverse |
| `charset` | `"cp437"` or `"ascii"` | later | The character set. Absent means Unicode, and Unicode is never written |
| `sauceTitle`, `sauceComments`, `sauceHistory` | string, string arrays | later | The SAUCE fields you edit |
| `sourceSauce` | SAUCE record object | later | The record an imported `.ans` arrived with, so the original credit survives |
| `letterSpacing`, `aspectRatio` | numbers | later | SAUCE display hints, as numbers (the app's raw values). Strings are not accepted |
| `sauceFontName` | string | later | The font named in SAUCE, such as `IBM VGA` |

"Later" fields were added without a version bump: they are optional, and every reader treats absence as the old behavior.

## Cells, colors and layers

### Colors

A color is an object with a `type` and its values. Readers clamp out-of-range numbers instead of failing.

| Kind | JSON | Values |
| --- | --- | --- |
| 16 colors | `{"type": "ansi16", "n": 9}` | `n` 0–15, in ANSI order (1 is red, 4 is blue), 8–15 the bright versions |
| 256 colors | `{"type": "ansi256", "n": 196}` | `n` 0–255, the xterm palette |
| Truecolor | `{"type": "truecolor", "r": 255, "g": 128, "b": 0}` | `r`, `g`, `b` 0–255 |

### Cells

```json
{"character": "▓", "foreground": {…}, "background": {…}, "blink": true}
```

- `character` is a string, so glyphs built from several code points fit.
- `blink` appears only when it is true. It means "blink" only when the document's `backgroundMode` is `"blinkingText"`.
- Missing fields take defaults: a space, bright white (15) on black (0), no blink.

A layer's `cells` is an array of rows, each an array of cells, top-left first. Every row should be `columns` long and there should be `rows` rows. Readers pad short grids and crop long ones rather than fail.

### Layers

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | UUID string | Identity, kept across saves |
| `name` | string | Shown in the layers panel |
| `visible` | boolean | Hidden layers stay in the file but don't show or export |
| `cells` | grid | The layer's picture |
| `rowOffset`, `colOffset` | numbers | How far the layer is shifted from the canvas origin |
| `kind` | `{"type": "drawing"}` or `{"type": "terminal", "shellPath": …}` | Terminal layers are the app's live shell layers. Other editions skip them when loading |
| `historyEnabled` | boolean | The app's stroke-replay flag |

The first layer in the array is the **top** of the stack. A drawing has at most 7 layers.

**Transparency is a rule, not a field.** A cell is see-through when it is a space on a black (`ansi16` 0) background. Any other cell covers the layers beneath it, including a space on any other background. Exporters flatten the visible layers bottom-up with this rule.

## .ansipresets

An `.ansipresets` file is a JSON array of 16 slots. Each slot is a preset object or `null` for an empty slot. The same array is what the terminal editions keep in `presets.json`, and what an `.ansipants` file holds under `presets`.

```json
[
  {"name": "shade", "tool": "pencil.tip", "character": "▓",
   "foreground": {"type": "ansi16", "n": 11}, "background": {"type": "ansi16", "n": 0},
   "brushSize": {"type": "cells", "n": 2}},
  null,
  …
]
```

Every field in a preset is optional. A reader applies the fields that are present and leaves the rest of the current setup alone. The terminal editions write the file as `<document name>.ansipresets`, or `presets.ansipresets` for an unnamed drawing, and read both `.ansipresets` and `.json`.

### Preset fields, by area

| Area | Keys |
| --- | --- |
| Basics | `name`, `tool` (an id from the tool table below), `character`, `foreground`, `background`, `foregroundNull`, `backgroundNull` |
| Brush | `brushSize` (`{"type": "cells", "n": 2}`, `{"type": "half"}` or `{"type": "quarter"}`), `brushRounded`, `symmetryMode`, `paintSimulatedPressureControlsSize`, `simulatedPressureControlsSize`, `velocitySensitivity` |
| Color cycle and random | `foregroundCycle`, `fgCycleColors`, `fgCycleStep`, `foregroundRandom`, `fgRandomColors`, and the same five for background (`bg…`), plus `colorCycleResetPerStroke` |
| Character cycle and random | `characterCycle`, `charCycleSequence`, `charCycleStep`, `characterRandom`, `charRandomGroups`, `charCycleChars`, `charRandomChars`, `charCycleForward`, `charCycleWrap`, `charCycleResetPerStroke` |
| Pen and Line | `penLineStyle`, `lineDrawStyle`, `lineStitchMode`, `lineThickness`, `lineThicknessCells` |
| Box | `boxFilled`, `boxOutline`, `boxPenLine`, `boxThickness`, `boxStyle` (`single`, `double`, `mixedH`, `mixedV`) |
| Oval | `ovalFilled`, `ovalOutline`, `ovalDrawMode`, `ovalThickness`, `ovalLinesPen`, `ovalLinesTangent` |
| Shape fill | `shapeFillTUI` (the terminal editions' own fill style; the app reads and writes it too) |
| Fill tool | `gradientFill`, `gradientDirection`, `fillMatchChar`, `fillMatchFg`, `fillMatchBg` |
| Spray | `sprayRadius`, `sprayDensity`, `sprayTargetBG`, `sprayVariance`, `sprayVariancePct`, `simulatedPressureControlsSpray` |
| Dodge and Burn | `dodgeBurnStrength` (0–1), `dodgeBurnTargetFG`, `dodgeBurnTargetBG`, `dodgeBurnGreys`, `simulatedPressureControlsDodgeBurn` |
| Prism | `prismAngle` (degrees), `prismTargetFG`, `prismTargetBG`, `prismGreys`, `simulatedPressureControlsPrism` |
| Scatter | `scatterVariance` (0–1), `scatterTargetFG`, `scatterTargetBG`, `scatterGreys`, `simulatedPressureControlsScatter` |
| Smudge | `smudgeTargetFG`, `smudgeTargetBG`, `smudgeTargetChar`, `smudgeResample` |
| Text | `fontScale`, `textTypeDirection` |

**Tool ids.** The `tool` value is the app's icon name for the tool, so both sides agree on one spelling. An unknown id loads as Paint.

| Tool | `tool` id |
| --- | --- |
| Paint | `paintbrush.fill` |
| Draw | `pencil.tip` |
| Pen | `scribble` |
| Eraser | `eraser.fill` |
| Fill | `drop.fill` |
| Spray | `aqi.medium` |
| Line | `line.diagonal` |
| Box | `rectangle` |
| Oval | `oval` |
| Smudge | `hand.point.up.left.fill` |
| Dodge | `sun.max` |
| Burn | `moon.fill` |
| Prism | `camera.filters` |
| Scatter | `sparkles` |
| Text | `keyboard` |
| Select | `rectangle.dashed` |
| Panspect | `eye` (or `magnifyingglass`) |

A few pairs exist for history: `lineThickness` and `sprayVariance` are the app's forms, while `lineThicknessCells` and `sprayVariancePct` are older terminal forms that are still read. `velocitySensitivity` is the speed-to-pressure curve, and must not be confused with the app's Apple Pencil `pressureSensitivity`.

**One bad slot doesn't sink the bank.** Each slot is decoded on its own. A slot that can't be read becomes empty and the others load. If a saved bank can't be read at all, the terminal editions keep a one-time `.corrupt` copy before writing a new one.

## .ans and SAUCE

An `.ans` file is the classic format: the flattened picture as characters and escape codes, then an end-of-file byte, then a 128-byte SAUCE record. ANSIpants writes it the way classic viewers expect, so old and new tools show it the same. The code is `Core/ANSFile.swift` (16M) and `DrawingViewModel+FileIO.swift` (app).

### The body

- **Always CP437, one byte per cell.** Characters outside CP437 (rounded corners, quadrant blocks, emoji) are swapped for the nearest CP437 glyph, and the save reports how many cells it changed. A UTF-8 body is never written, because classic viewers decode `.ans` as CP437 and a multi-byte glyph would shift every row after it. The `.utf8.ans` companion is for UTF-8.
- **No line breaks.** Rows are written back to back. The SAUCE width tells a viewer where each row ends.
- **Escape codes only when something changes.** Each row starts with a reset, and a color code is written only when the colors differ from the cell before.
- **16 colors use the classic encoding.** A bright foreground is "bold" plus the base color. A bright background is the blink code plus the base background, unless the document is in blinking-text mode, where backgrounds stay in the dark eight and the blink code means blink.
- **256 and truecolor use modern codes** (`38;5;n`, `38;2;r;g;b` and their background forms), which newer terminals understand.
- **Mono uses no color at all,** only normal, bold and reverse, so the reader's screen supplies the tint.

### The SAUCE record

After the body comes the byte `0x1A` (the old DOS end-of-file mark, so a plain viewer stops there), an optional comment block, and the 128-byte record:

| Field | What ANSIpants writes |
| --- | --- |
| Title, Author, Group | From the save options. Kept from the original when you open art someone else signed |
| Date | The save date, `YYYYMMDD` |
| Data type, File type | Character, ANSi |
| Width, Height | The canvas size in cells |
| Flags | Whether bright backgrounds are used (set only when one actually appears), letter spacing and aspect ratio hints |
| Font name | Usually `IBM VGA` |
| Comments | Up to 255 lines of 64 characters, in a `COMNT` block before the record |

When ANSIpants opens an `.ans`, it reads the record back. The width splits the rows, the flags decide how the blink bit is read, and the credit is kept in the document (`sourceSauce` in `.ansipants`), so saving again doesn't replace the original artist's name with yours.

## Compatibility rules

Three separate programs read and write these files, so the formats follow five rules. The code comments record the bugs that taught each one.

1. **Add, never change.** A new setting is a new optional key. An existing key never changes meaning or type. (`letterSpacing` is a number because the app writes a number; a string there would be silently dropped.)
2. **Absent means the old behavior.** A reader that finds no `charset` assumes Unicode; no `backgroundMode` means bright backgrounds; no `colorDepth` means 16 colors.
3. **Write defaults as nothing.** A value at its default is left out (`blink: false`, `charset: "unicode"`), so a file saved today is byte-identical to one saved before the key existed.
4. **Never throw on a strange value.** An unknown enum value is decoded by hand to a default. Swift's automatic decoding would throw, and since loads use `try?`, the drawing would open blank with no error.
5. **Salvage, don't wipe.** Grids are padded or cropped to the stated size, preset slots decode one by one, and out-of-range numbers are clamped.

### What doesn't survive a round trip today

Two app-only fields are read by 16M but not written back when it saves an `.ansipants`: the reference image (with its position and scale) and `compoMode`. Dropping the reference image is accepted, since hybrid text-on-image art only happens in the app. Dropping `compoMode` is still an open question: an app document in Classic mode comes back from 16M with Classic off. The app's live terminal layers are also dropped when another edition opens the file.
