---
name: ppt-to-beamer
description: Convert a PowerPoint institutional template (.potx / .pptx) into a LaTeX Beamer theme that mimics the original colour palette, fonts, slide layouts, and logos. Produces a complete package -- `beamertheme<Name>.sty`, a self-documenting `.tex` tutorial deck, `logographics/` with the brand PNGs, `LICENSE`, `NOTICE`, `README.md`, `.gitignore`. Use when the user provides a PowerPoint template (with optional brand-guidelines PDF and logo files) and asks for a Beamer equivalent.
---

# PPT to Beamer Skill

This skill walks Claude through converting an institutional PowerPoint template (typically a `.potx` master) into a faithful LaTeX Beamer theme. The output is a self-contained folder ready to push to a GitHub repository.

## When to use this skill

Trigger this skill when the user:

- Provides a `.potx` or `.pptx` file and asks for a Beamer template that "looks the same" / "matches the brand" / "mimics the PowerPoint".
- Provides institutional brand guidelines (PDF, web page, or design manual) and asks for a Beamer theme that complies with them.
- Has logo / wordmark PNGs and wants them placed consistently on Beamer cover, section, and normal slides.
- Says "convert this PowerPoint template to LaTeX Beamer" or any variation thereof.

Do **not** trigger this skill when the user only asks for a generic Beamer theme without an institutional reference, or when they ask for non-template work (writing content, editing slides, etc.).

## What you need from the user

Ask up front for any of these that are missing:

1. **The PowerPoint template** — `.potx` (template) preferred over `.pptx` (presentation); both work.
2. **Logo / brand assets** — PNG or SVG, ideally in three colour variants (primary, white, black) and multiple shape variants (wordmark only, emblem only, combined).
3. **Brand guidelines** (optional but helpful) — typography rules, colour usage policy, "official vs unofficial" requirements.
4. **Target audience** — if academic / scientific, the deck should support `siunitx`, `amsmath`, `booktabs`, etc.; if business, drop those.
5. **Repo intent** — public open-source, private internal, or just local? Affects licence and NOTICE wording.

## Workflow

### Stage 1 — Inspect the PowerPoint master

A `.potx` is a ZIP archive. Extract it into a temp directory and read the OpenXML structure.

```bash
mkdir -p /tmp/potx-extract && cd /tmp/potx-extract && unzip -o "<path-to>.potx"
```

Files of interest:

| Path | Purpose |
|------|---------|
| `ppt/theme/theme1.xml`            | Colour palette (`<a:srgbClr val="HEXHEX">`) and font names (`<a:majorFont>`, `<a:minorFont>`) |
| `ppt/slideMasters/slideMaster1.xml` | Master placeholder positions (title, body, footer, slide number, date), default font sizes |
| `ppt/slideLayouts/slideLayout1.xml` ... | Per-layout positions; layout 1 is typically the cover |
| `ppt/media/`                      | Logos, emblems, watermarks (PNG/SVG) embedded in the template |
| `ppt/presentation.xml`            | Slide dimensions (`<p:sldSz cx="..." cy="..."/>`) — derives the aspect ratio |

See `reference/potx-structure.md` for the full XML decoder ring.

### Stage 2 — Extract the palette and fonts

Grep the HEX colours and font typefaces from `theme1.xml`:

```bash
grep -oE 'srgbClr val="[A-F0-9]{6}"' /tmp/potx-extract/ppt/theme/theme1.xml | sort -u
grep -oE 'typeface="[^"]*"' /tmp/potx-extract/ppt/theme/theme1.xml | sort -u
```

Classify the colours by role:

- **Primary brand colour** — usually the boldest, most saturated; appears in the theme's accent1 mapping.
- **Neutrals** — near-white / cream backgrounds, near-black text colour, mid-grey muted text.
- **Accent palette** — green, red, blue, etc. used for data viz / blocks.

For each colour, decide the LaTeX name. Conventionally: `<InstitutionName><Role>` (e.g. `AcmePurple`, `AcmeCream`, `AcmeGreen`). Avoid embedding the user's personal name.

For fonts: PowerPoint typically uses Arial / Calibri / Helvetica Neue. None of these ship with TeX Live for free. The closest free LaTeX equivalents are:

- **Arial / Helvetica Neue** → `\usepackage{helvet}` + `\renewcommand{\familydefault}{\sfdefault}`
- **Calibri** → `\usepackage{carlito}` (open-source Calibri clone)
- **Times New Roman** → `\usepackage{mathptmx}` (default in `book` class)

Document the substitution in the README's "Typography" section.

### Stage 3 — Extract the layout geometry

From `slideMaster1.xml`, read the placeholder `<p:sp>` positions and sizes. Each placeholder has:

```xml
<p:sp>
  <p:nvSpPr>...<p:ph type="title"/>...</p:nvSpPr>
  <p:spPr>
    <a:xfrm>
      <a:off x="377509" y="339725"/>   <!-- EMU coordinates -->
      <a:ext cx="10152854" cy="1159676"/>
    </a:xfrm>
  </p:spPr>
</p:sp>
```

EMU = English Metric Unit. 914400 EMU = 1 inch. Convert to mm with `EMU * 25.4 / 914400`.

Map the placeholders to Beamer templates:

| PPT placeholder | Beamer template |
|-----------------|-----------------|
| `type="title"` (master) | `\setbeamertemplate{frametitle}` |
| `type="body"`           | the regular frame body (no template needed; honour margins) |
| `type="ctrTitle"` (layout 1 / cover) | `\titleframe` user command |
| `type="dt"` (date)      | `\insertdate` slot in `\setbeamertemplate{footline}` |
| `type="ftr"` (footer text) | `\insertshorttitle` slot in `footline` |
| `type="sldNum"` (slide number) | `\insertframenumber` slot in `footline` |

### Stage 4 — Extract logos

List `ppt/media/`:

```bash
ls /tmp/potx-extract/ppt/media/
```

Group images by use:

- **Wordmark only** — usually has a tall, narrow aspect ratio (e.g. 4:1).
- **Emblem only** — square or near-square aspect.
- **Combined** — wordmark + emblem horizontally / vertically arranged.

If the user has provided separate higher-resolution logos (typical for brand portals), prefer those over the PPT-embedded media. If only PPT media exists, accept that resolution.

Provide three colour variants per logo if available (primary, white, black). If the user only supplies one colour, document the gap in the NOTICE file rather than fabricating others.

Copy the chosen PNGs into `logographics/` of the output folder, with a consistent naming scheme:

```
<Institution>-logo-<colour>.png            # combined, with subtitle
<Institution>-logo-horiz-<colour>.png      # horizontal, no subtitle
<Institution>-logo-vert-<colour>.png       # vertical, no subtitle
<Institution>-wordmark-<colour>.png        # wordmark only
<Institution>-emblem-<colour>.png          # emblem only (may need cropping)
```

### Stage 5 — Generate `beamertheme<Name>.sty`

Build the theme file with these numbered sections (the structure that works well in practice):

```
% 1. Colour palette         -- \definecolor for each HEX
% 2. Typography             -- \usepackage{helvet}/{carlito}/...
% 3. Beamer colour bindings -- \setbeamercolor{structure}{fg=<primary>}
% 4. Font sizes             -- \setbeamerfont per element
% 5. Geometry               -- \setbeamersize, text margins
% 6. Logo declarations      -- \pgfdeclareimage for every variant
% 7. Customisation commands -- \shorttitle, \<name>headline, etc.
% 8. Headline template      -- top-right logo at fixed margins
% 9. Frame title            -- bold primary-coloured, no rule
% 10. Footline              -- thin rule + date | short title | slide number
% 11. Title page (cover)    -- full-bleed primary bg + white logo
% 12. Section dividers      -- presets: progress / solid / none
```

**Critical implementation gotchas** documented in `reference/beamer-internals.md` — pre-empt these in any new conversion:

- `\AtBeginSection` sometimes runs *before* Beamer updates `\insertsection`. Capture the name yourself by wrapping `\section` in your own command.
- `\tableofcontents[sectionstyle=show/shaded]` inside `\AtBeginSection` can silently drop the current section. Either read the `.toc` file directly with a custom `\beamer@sectionintoc` redefinition, or omit `currentsection` from the options.
- The `[plain]` frame option suppresses both headline and footline. If you want the logo on a plain frame, draw it in the frame body via a helper macro.
- `\hfill` inside Beamer's headline template doesn't always push to the right. Use `\hbox to \paperwidth{\hfill ...}` for guaranteed alignment.
- For frames containing `\end{frame}` literally inside verbatim, use `[fragile=singleslide]`.

See `reference/beamer-internals.md` for a fuller catalogue.

### Stage 6 — Generate the tutorial deck

Write `<name>.tex` as a self-documenting deck that doubles as starter code AND an interactive command reference. Suggested structure (3 sections, ~25 slides):

1. **Quick start** — what the template provides, minimal source, file layout, three slide types, frame alignment options.
2. **Visual identity** — logo style switch, colour palette tables (primary + accent), blocks, equations + units, tables, section-divider presets.
3. **Customisation** — overriding colours / fonts / footer, partner logos, summary.

Plus closing thank-you slide. Keep section count low (≤4) so the progress-style section divider isn't overcrowded.

Add prominent **unofficial-template / vibe-coded disclaimers** in three places:

- README top (blockquote)
- `.sty` header comment
- A red `alertblock` on the first tutorial slide

### Stage 7 — Generate companion files

| File | Content |
|------|---------|
| `README.md` | Hook + unofficial / vibe-coded disclaimer + quick start + colour palette + command reference + customisation + dual-licence section. Prose on single long lines (markdown editors handle wrapping). |
| `LICENSE`   | MIT licence covering the LaTeX code. Use the author's real name plus their GitHub profile URL for the copyright line (e.g. `Copyright (c) YEAR Name (https://github.com/handle)`) — never an email address. Append a paragraph noting the licence does NOT cover the institutional logos. |
| `NOTICE`    | Detailed brand statement: which assets are trademarks of the institution, which are licensed under MIT (the code), what approximations were made (e.g. Arial → Helvetica), where to report brand-compliance issues. |
| `.gitignore` | LaTeX aux (`*.aux`, `*.log`, `*.nav`, `*.snm`, `*.toc`, `*.out`, `*.synctex.gz`, `*.fls`, `*.fdb_latexmk`, `*.bbl`, `*.blg`, `*.vrb`), `*.pdf`, editor junk (`.DS_Store`, `*.swp`, `.vscode/`, `.idea/`). |

### Stage 8 — Verify

Before declaring done:

1. Compile the tutorial: `pdflatex <name>.tex` × 2 (TOC needs two passes).
2. Visually inspect each slide type with `pdftoppm` previews — cover, outline, every section-divider preset, normal frame, blocks, equations, tables, thank-you.
3. Audit for sensitive info: `grep -inE "<institution>|<user-name>|<email>"` should return only intentional references.
4. Audit for word-choice typos common to LaTeX writing: `override` not `overwrite`, `licence` (UK) vs `license` (US) — pick one and stick with it; the MIT licence file uses American spelling because that's the canonical text, but README prose can use whichever the user prefers.
5. Sanity-check that the four logo variants all reference real PNG files in `logographics/`.

### Stage 9 — Output the final folder

The final delivered folder should be:

```
<repo-name>/
├── .gitignore
├── LICENSE
├── NOTICE
├── README.md
├── beamertheme<Name>.sty
├── <name>.tex
└── logographics/
    ├── <Institution>-logo-{purple,white,black}.png
    ├── <Institution>-logo-horiz-{purple,white,black}.png
    ├── <Institution>-logo-vert-{purple,white,black}.png
    ├── <Institution>-logo-vert-full-{purple,white,black}.png
    ├── <Institution>-wordmark-{purple,white,black}.png
    └── <Institution>-emblem-{purple,white,black}.png
```

(Drop variants that the user did not provide assets for.)

If the user wants a GitHub repo, offer to create it via `gh repo create`. Always commit the initial state as a single clean commit — no development history exposed.

## Reference

- [reference/potx-structure.md](reference/potx-structure.md) — full decoder ring for the OpenXML PowerPoint structure
- [reference/beamer-internals.md](reference/beamer-internals.md) — Beamer template / `\AtBeginSection` / `\hfill` gotchas and workarounds
- [reference/color-extraction.md](reference/color-extraction.md) — mapping PPT theme colours to LaTeX `\definecolor` + Beamer `\setbeamercolor`
- [examples/walkthrough.md](examples/walkthrough.md) — a sanitised end-to-end example

## Anti-patterns to avoid

- **Don't bake the originating institution into Claude's general knowledge.** Never assume what brand the user is converting. Always ask for / extract from the `.potx`.
- **Don't fabricate logo variants.** If the user provides only a purple logo, do not generate a white version with ImageMagick `-fill white -opaque purple` unless explicitly asked — that often produces washed-out anti-aliased edges that fail brand QA.
- **Don't claim official endorsement.** Every README and `.sty` header must call the result *unofficial* unless the user explicitly states they have institutional approval.
- **Don't hard-wrap markdown prose.** Single long lines per paragraph; the editor handles wrap.
- **Don't use emojis** in `.sty`, `.tex`, README, or commit messages unless the user explicitly asks.
- **Don't push to `main` / `master` directly** if the user's environment has a pre-commit hook blocking it. Commit on a `release` branch and `git push origin release:main`.

## Output style

The tutorial deck and README should adopt a friendly, professional academic tone. Compact prose, concrete examples, no marketing language. Default to UK English unless the user's source materials are American.
