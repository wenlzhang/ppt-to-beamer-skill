# PowerPoint `.potx` / `.pptx` OpenXML structure

A PowerPoint template is a ZIP archive containing XML files. This is the decoder ring you need to extract everything the conversion skill cares about.

## Top-level layout

```
template.potx                          (or .pptx)
├── [Content_Types].xml                MIME-type registry; ignore
├── _rels/.rels                        Top-level relationships; ignore
├── docProps/                          Document properties; ignore
└── ppt/
    ├── presentation.xml               Slide size, default settings
    ├── _rels/presentation.xml.rels    Links presentation to masters / themes
    ├── slideMasters/
    │   ├── slideMaster1.xml           THE master -- placeholders, default text
    │   └── _rels/slideMaster1.xml.rels
    ├── slideLayouts/
    │   ├── slideLayout1.xml           Layout 1 (typically the cover)
    │   ├── slideLayout2.xml           Layout 2 (typically title + content)
    │   ├── ...                        Up to ~30 layouts in modern templates
    │   └── _rels/...
    ├── theme/
    │   ├── theme1.xml                 Colour palette + fonts (CRUCIAL)
    │   └── ...
    └── media/                         Embedded logos, watermarks (PNG/SVG)
        ├── image1.svg
        ├── image2.png
        └── ...
```

## Extraction

```bash
mkdir -p /tmp/potx-extract && cd /tmp/potx-extract && unzip -o "<path>.potx"
```

`.potx` and `.pptx` use identical internal structure; the only difference is the MIME type. Extract either with `unzip`.

## Colour palette — `ppt/theme/theme1.xml`

Colours appear as `<a:srgbClr val="HEXHEX"/>` tokens inside the theme's `<a:clrScheme>`:

```xml
<a:clrScheme name="...">
  <a:dk1><a:sysClr val="windowText" lastClr="000000"/></a:dk1>
  <a:lt1><a:sysClr val="window"     lastClr="FFFFFF"/></a:lt1>
  <a:dk2><a:srgbClr val="222222"/></a:dk2>
  <a:lt2><a:srgbClr val="F0EDE6"/></a:lt2>
  <a:accent1><a:srgbClr val="4A55A2"/></a:accent1>   ← primary brand colour
  <a:accent2><a:srgbClr val="00B356"/></a:accent2>
  <a:accent3><a:srgbClr val="ED7122"/></a:accent3>
  <a:accent4><a:srgbClr val="36B7F6"/></a:accent4>
  <a:accent5><a:srgbClr val="D52515"/></a:accent5>
  <a:accent6><a:srgbClr val="F6D62A"/></a:accent6>
  <a:hlink><a:srgbClr val="6746EB"/></a:hlink>
  <a:folHlink><a:srgbClr val="9E92E8"/></a:folHlink>
</a:clrScheme>
```

Conventional role mapping:

| OpenXML name | Conventional role | LaTeX naming pattern |
|--------------|-------------------|----------------------|
| `dk1`        | Dark text (body) | `<Inst>Black` |
| `lt1`        | Background light | `<Inst>White` (often `FFFFFF`; rarely needed as a colour) |
| `dk2`        | Secondary dark | `<Inst>Grey` or `<Inst>BlackText` |
| `lt2`        | Secondary light (block bodies, callouts) | `<Inst>Cream` |
| `accent1`    | **Primary brand** | `<Inst>Primary` (e.g. `<Inst>Purple`, `<Inst>Blue`) |
| `accent2..6` | Data-viz accents | `<Inst>Green`, `<Inst>Orange`, etc. |
| `hlink` / `folHlink` | Link / visited link | Often a colour variant; can be exposed or skipped |

To find all unique HEX colours in one shot:

```bash
grep -oE 'srgbClr val="[A-F0-9]{6}"' ppt/theme/theme1.xml | sort -u
```

## Fonts — `ppt/theme/theme1.xml`

```xml
<a:fontScheme name="...">
  <a:majorFont>
    <a:latin typeface="Arial"/>
    <a:ea typeface=""/>
    <a:cs typeface=""/>
  </a:majorFont>
  <a:minorFont>
    <a:latin typeface="Arial"/>
    ...
  </a:minorFont>
</a:fontScheme>
```

`<a:majorFont>` is used for titles, `<a:minorFont>` for body. Most institutional templates use Arial / Calibri / Helvetica Neue for both. Use the substitution table in `SKILL.md` Stage 2.

## Slide dimensions — `ppt/presentation.xml`

```xml
<p:sldSz cx="12192000" cy="6858000"/>
```

Values are EMU (English Metric Unit). `cx` = width, `cy` = height. Conversion:

| Ratio | cx × cy (EMU) | Dimensions | Beamer option |
|-------|----------------|------------|---------------|
| 16:9  | 12192000 × 6858000 | 13.33″ × 7.5″ | `aspectratio=169` |
| 16:10 | 12192000 × 7620000 | 13.33″ × 8.33″ | `aspectratio=1610` |
| 4:3   | 9144000 × 6858000  | 10″ × 7.5″ | `aspectratio=43` (or default) |
| 3:2   | 9144000 × 6096000  | 10″ × 6.67″ | `aspectratio=32` |

EMU → inches: divide by 914400.

## Slide master — `ppt/slideMasters/slideMaster1.xml`

The master defines default placeholder positions and text styles for every layout that inherits from it. Look for `<p:sp>` (shape) elements with `<p:ph>` (placeholder) children:

```xml
<p:sp>
  <p:nvSpPr>
    <p:cNvPr id="7" name="Title Placeholder 6"/>
    <p:nvPr>
      <p:ph type="title"/>
    </p:nvPr>
  </p:nvSpPr>
  <p:spPr>
    <a:xfrm>
      <a:off x="377509" y="339725"/>       <!-- top-left, EMU -->
      <a:ext cx="10152854" cy="1159676"/>  <!-- size, EMU -->
    </a:xfrm>
  </p:spPr>
  <p:txBody>
    <a:bodyPr.../>
    <a:lstStyle>
      <a:lvl1pPr>
        <a:defRPr sz="3200" b="1">          <!-- 32 pt, bold -->
          ...
        </a:defRPr>
      </a:lvl1pPr>
    </a:lstStyle>
  </p:txBody>
</p:sp>
```

Placeholder `type` values to look for:

| `type` | Meaning | Maps to (in Beamer) |
|--------|---------|---------------------|
| `title`     | Title placeholder | `frametitle` template |
| `body`      | Body content    | The frame body (no template; honour text-margins) |
| `ctrTitle`  | Centred title (cover) | `\titleframe` user command |
| `subTitle`  | Subtitle (cover) | The subtitle inside `\titleframe` |
| `dt`        | Date placeholder | `\insertdate` in footline |
| `ftr`       | Footer text | `\insertshorttitle` in footline |
| `sldNum`    | Slide number | `\insertframenumber` in footline |
| `pic`       | Picture placeholder | Background image / decorative logo |

Default text sizes are in `<a:defRPr sz="...">` where the value is half-points × 100. So `sz="3200"` = 32 pt, `sz="1800"` = 18 pt, `sz="800"` = 8 pt.

## Slide layouts — `ppt/slideLayouts/slideLayoutN.xml`

Each layout inherits from the master and overrides specific placeholders. Layout 1 is conventionally the **cover** (`<p:cSld name="Title picture">` or similar).

For the conversion, look at layout 1 to extract:

- The cover background (often a full-bleed coloured rectangle via `<p:sp>` with `<a:solidFill>`).
- The title position on the cover (usually larger and lower than the master title).
- Any cover-only graphics (decorative bars, watermarks).

Other layouts often share enough structure with the master that you only need to read layout 1 + master for a faithful conversion.

## Media — `ppt/media/`

Logos, emblems, and watermarks live here. File extensions vary:

- `.svg` — vector (best; can be rasterised at any resolution)
- `.png` — raster, transparent background (good)
- `.emf` / `.wmf` — Windows vector formats; harder to handle

Identify which media file is which logo by:

1. **Aspect ratio** — wordmark-only is wide (4:1 or wider); emblem-only is square (1:1); combined is in between (2:1 to 3:1).
2. **Filename** — often descriptive in non-Microsoft templates; cryptic (`image1.png`, `image2.svg`) in Microsoft-generated ones.
3. **Cross-reference with layouts** — `ppt/slideLayouts/_rels/slideLayout1.xml.rels` lists the images used by layout 1 (usually the cover, which uses the prominent logo).

When the user supplies logos separately (typical for institutional brand portals), prefer those over `ppt/media/` — they are usually higher resolution and brand-portal-canonical.

## Verifying your extraction

After extracting, sanity-check:

```bash
# Colour palette
grep -oE 'srgbClr val="[A-F0-9]{6}"' ppt/theme/theme1.xml | sort -u

# Font names
grep -oE 'typeface="[^"]*"' ppt/theme/theme1.xml | sort -u

# Slide dimensions
grep "sldSz" ppt/presentation.xml

# Placeholder types in the master
grep -oE 'ph type="[a-zA-Z]+"' ppt/slideMasters/slideMaster1.xml | sort -u

# Media inventory
ls ppt/media/
file ppt/media/*
```

If any of these come up empty, the template may use a non-standard structure (custom slide masters, content placeholders without explicit types). In that case, open the `.potx` in PowerPoint / LibreOffice Impress and inspect visually.
