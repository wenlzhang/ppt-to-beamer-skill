# Worked example — Acme University template

A sanitised end-to-end conversion using a fictional "Acme University" PowerPoint template. The same flow applies to any real institutional template; substitute the names and HEX values as appropriate.

## Inputs

The user provides:

- `Acme-University.potx` — the PowerPoint template (16:9, blue primary brand).
- `acme-logos/` — folder containing:
  - `Acme-logo-blue.png` (combined logo + wordmark, blue on transparent)
  - `Acme-logo-white.png` (same, white)
  - `Acme-emblem-blue.png` (round emblem only)
  - `Acme-emblem-white.png`
  - `Acme-wordmark-blue.png`
  - `Acme-wordmark-white.png`
- `Acme-brand-guidelines.pdf` — institutional brand book.

## Conversation

> **User:** "I've got my university's PowerPoint template at `~/Downloads/Acme-University.potx` plus the official logos in `~/Downloads/acme-logos/`. Please convert this into a Beamer theme."

> **Claude (this skill):** OK, on it. I'll:
>
> 1. Extract the `.potx` and read the theme XML.
> 2. Pull the colour palette, fonts, and slide-layout geometry.
> 3. Copy your logos into a `logographics/` folder.
> 4. Generate `beamerthemeAcme.sty`, an `acme.tex` tutorial deck, plus `LICENSE`, `NOTICE`, `README.md`, and `.gitignore`.
> 5. Compile twice and show you the result.
>
> Where should I drop the output folder?

## Stage 1 — Extract and inspect

```bash
mkdir -p /tmp/acme-potx && cd /tmp/acme-potx && unzip -o ~/Downloads/Acme-University.potx
grep -oE 'srgbClr val="[A-F0-9]{6}"' ppt/theme/theme1.xml | sort -u
```

Output (fictional):

```
srgbClr val="000000"
srgbClr val="002F6C"          ← primary brand blue
srgbClr val="0066CC"
srgbClr val="00A651"
srgbClr val="222222"
srgbClr val="E6B800"
srgbClr val="F5F2EC"
srgbClr val="FFFFFF"
```

```bash
grep -oE 'typeface="[^"]*"' ppt/theme/theme1.xml | sort -u
```

Output:

```
typeface=""
typeface="Calibri"
```

```bash
grep "sldSz" ppt/presentation.xml
```

Output:

```xml
<p:sldSz cx="12192000" cy="6858000"/>     <!-- 16:9 -->
```

## Stage 2 — Palette classification

| HEX | Theme role | LaTeX name |
|------|------------|------------|
| `002F6C` | `accent1` — primary brand | `AcmeBlue` |
| `0066CC` | hlink — bright blue | `AcmeBlueLight` |
| `00A651` | `accent2` — green | `AcmeGreen` |
| `E6B800` | `accent3` — gold | `AcmeGold` |
| `F5F2EC` | `lt2` — warm off-white | `AcmeCream` |
| `222222` | `dk2` — body text | `AcmeBlack` |

Font substitution: Calibri → `\usepackage{carlito}` (open-source Calibri clone).

## Stage 3 — Layout geometry

`slideMaster1.xml` reveals:

- Title placeholder: `<a:off x="457200" y="457200"/> <a:ext cx="11277600" cy="1143000"/>` → roughly 12.5 mm from top-left, 308 mm × 31 mm.
- Body placeholder: starts at y=1714500 EMU → ~47 mm from top.
- Date / footer / slide-num placeholders at y=6477000 EMU → ~177 mm from top (close to bottom of the 190 mm slide).

Title default font size: 32 pt (`sz="3200"`). Body: 18 pt (`sz="1800"`). Footer items: 8 pt (`sz="800"`).

## Stage 4 — Logo placement

User's `acme-logos/` already provides three colour variants for emblem / wordmark / combined. Copy verbatim into `logographics/` with the convention names:

```
logographics/
├── Acme-logo-blue.png         → from Acme-logo-blue.png
├── Acme-logo-white.png        → from Acme-logo-white.png
├── Acme-emblem-blue.png       → from Acme-emblem-blue.png
├── Acme-emblem-white.png      → from Acme-emblem-white.png
├── Acme-wordmark-blue.png     → from Acme-wordmark-blue.png
└── Acme-wordmark-white.png    → from Acme-wordmark-white.png
```

Black variants are not supplied; mention this in NOTICE.

## Stage 5 — Generate `beamerthemeAcme.sty`

(Excerpt — the full file follows the section structure outlined in `SKILL.md` Stage 5.)

```latex
\ProvidesPackage{beamerthemeAcme}[2026/05/22 v1.0 Acme University Beamer theme]
\mode<presentation>

\RequirePackage{tikz}
\RequirePackage{xcolor}
\RequirePackage{etoolbox}

% --- Palette ---
\definecolor{AcmeBlue}{HTML}{002F6C}
\definecolor{AcmeBlueLight}{HTML}{0066CC}
\definecolor{AcmeGreen}{HTML}{00A651}
\definecolor{AcmeGold}{HTML}{E6B800}
\definecolor{AcmeCream}{HTML}{F5F2EC}
\definecolor{AcmeBlack}{HTML}{222222}

% --- Typography (Calibri-clone) ---
\RequirePackage{carlito}
\renewcommand{\familydefault}{\sfdefault}

% --- Beamer colour bindings ---
\setbeamercolor{structure}{fg=AcmeBlue}
\setbeamercolor{normal text}{fg=AcmeBlack,bg=white}
\setbeamercolor{frametitle}{fg=AcmeBlue,bg=white}
\setbeamercolor{title}{fg=white}
\setbeamercolor{block title}{fg=white,bg=AcmeBlue}
\setbeamercolor{block body}{fg=AcmeBlack,bg=AcmeCream}
\setbeamercolor{footline}{fg=AcmeBlue,bg=white}

% --- Logos ---
\pgfdeclareimage[height=11mm]{AcmeLogo}{logographics/Acme-logo-blue}
\pgfdeclareimage[height=11mm]{AcmeLogoWhite}{logographics/Acme-logo-white}
\pgfdeclareimage[height=12mm]{AcmeEmblem}{logographics/Acme-emblem-blue}
\pgfdeclareimage[height=12mm]{AcmeEmblemWhite}{logographics/Acme-emblem-white}
\pgfdeclareimage[height=8mm]{AcmeWordmark}{logographics/Acme-wordmark-blue}
\pgfdeclareimage[height=8mm]{AcmeWordmarkWhite}{logographics/Acme-wordmark-white}

% --- Dispatcher for the headline-style switch ---
\newcommand{\Acmelogostyle}{logo}
\newcommand{\acmeheadline}[1]{\renewcommand{\Acmelogostyle}{#1}}

\newcommand{\Acmeinsertlogoblue}{%
    \ifstrequal{\Acmelogostyle}{word}{\pgfuseimage{AcmeWordmark}}{%
    \ifstrequal{\Acmelogostyle}{logo+word}{\pgfuseimage{AcmeLogo}}{%
        \pgfuseimage{AcmeEmblem}}}%
}
% (mirror with white variants)

% --- Headline (logo top-right) ---
\setbeamertemplate{headline}{%
    \vspace*{2mm}%
    \hbox to \paperwidth{\hfill\Acmeinsertlogoblue\hspace*{6mm}}\par%
}

% --- Frame title ---
\setbeamertemplate{frametitle}{%
    \vspace{2mm}%
    \begin{beamercolorbox}[wd=\paperwidth,leftskip=8mm,rightskip=8mm]{frametitle}%
        \usebeamerfont{frametitle}\insertframetitle%
    \end{beamercolorbox}%
    \vspace{2mm}%
}

% --- Footline (rule + 3-slot footer) ---
\setbeamertemplate{footline}{%
    \hbox to \paperwidth{\hspace*{8mm}%
        \textcolor{AcmeBlue}{\rule{\dimexpr\paperwidth-16mm\relax}{0.3pt}}%
        \hspace*{8mm}}%
    \vspace{1mm}%
    \usebeamercolor[fg]{footline}\usebeamerfont{footline}%
    \hbox to \paperwidth{\hspace*{8mm}%
        \insertdate\hfill\insertshorttitle\hfill\insertframenumber%
        \hspace*{8mm}}%
    \vspace{1mm}%
}

% --- Title-page environment ---
\newenvironment{acmecover}{%
    \begingroup
    \setbeamercolor{background canvas}{bg=AcmeBlue}%
    \setbeamertemplate{headline}{%
        \vspace*{2mm}%
        \hbox to \paperwidth{\hfill\Acmeinsertlogowhite\hspace*{6mm}}\par%
    }%
    \setbeamertemplate{footline}{}%
    \setbeamertemplate{frametitle}{}%
    \begin{frame}%
}{\end{frame}\endgroup}

% --- (Section divider, customisation commands, etc. follow the same patterns) ---
\mode<all>
```

## Stage 6 — Tutorial deck

Generate `acme.tex` following the three-section structure in `SKILL.md` Stage 6. Open with an unofficial-template alert block, walk through Quick start / Visual identity / Customisation, close with a Thank-you `acmecover` slide.

## Stage 7 — Companion files

`LICENSE`, `NOTICE`, `README.md`, `.gitignore` per the templates in `SKILL.md` Stage 7. Use the institution's name and primary brand colour throughout.

## Stage 8 — Compile and verify

```bash
cd ~/Documents/acme-beamer
pdflatex acme.tex && pdflatex acme.tex
```

Both passes should output cleanly. Render previews to inspect visually:

```bash
pdftoppm -r 150 -png acme.pdf /tmp/acme-preview
```

Open `/tmp/acme-preview-01.png` etc. and confirm:

- Cover: full-bleed AcmeBlue, white logo top-right, large white title.
- Outline: TOC entries in AcmeBlack, logo top-right in AcmeBlue.
- Section divider: list of all sections, current in white, others dimmed.
- Normal frame: white bg, AcmeBlue title, footer with rule.
- Thank-you: full-bleed AcmeBlue, centred white message.

## Stage 9 — Hand-off

> **Claude:** Done — `~/Documents/acme-beamer/` is ready. Compiles in two passes to a 22-page PDF. Want me to create a GitHub repo for it?

User decides whether to publish. Done.

---

## Lessons from this walkthrough

- The `.potx` extraction takes < 30 s; the bulk of the time is the `.sty` synthesis and tutorial-deck generation.
- Font substitution honesty matters: documenting that Calibri → Carlito in the NOTICE file makes the result trustworthy.
- Asking up front for logo variants saves a round trip; if the user only has one colour variant, generate the others *only with explicit permission* (anti-aliasing degrades quickly).
- The dispatch macro (`\acmeheadline{logo|word|logo+word}`) is worth the up-front investment — it lets users switch styles deck-wide with one line.
