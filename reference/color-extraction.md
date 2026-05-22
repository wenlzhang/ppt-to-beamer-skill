# Colour extraction — PowerPoint theme → LaTeX / Beamer

Maps PowerPoint theme colours to LaTeX `\definecolor` declarations and Beamer `\setbeamercolor` bindings.

## Step 1 — Read the palette from `theme1.xml`

```bash
grep -oE 'srgbClr val="[A-F0-9]{6}"' ppt/theme/theme1.xml | sort -u
```

You'll get a list like:

```
srgbClr val="000000"
srgbClr val="00B356"
srgbClr val="222222"
srgbClr val="4A55A2"
srgbClr val="ED7122"
srgbClr val="F0EDE6"
srgbClr val="FFFFFF"
...
```

## Step 2 — Classify each HEX value

Look at *where* each colour appears in `theme1.xml`. The XML wrap tells you the role:

```xml
<a:clrScheme name="Default">
  <a:dk1>...000000...</a:dk1>      ← dk1 = body text (dark on light themes)
  <a:lt1>...FFFFFF...</a:lt1>      ← lt1 = background (light)
  <a:dk2>...222222...</a:dk2>      ← dk2 = secondary text colour
  <a:lt2>...F0EDE6...</a:lt2>      ← lt2 = secondary background (block bodies)
  <a:accent1>...4A55A2...</a:accent1>  ← PRIMARY brand colour
  <a:accent2>...00B356...</a:accent2>  ← secondary accent (often green)
  <a:accent3>...ED7122...</a:accent3>  ← (often orange)
  <a:accent4>...36B7F6...</a:accent4>  ← (often cyan)
  <a:accent5>...D52515...</a:accent5>  ← (often red)
  <a:accent6>...F6D62A...</a:accent6>  ← (often yellow)
  <a:hlink>...6746EB...</a:hlink>      ← link colour
  <a:folHlink>...9E92E8...</a:folHlink> ← visited-link colour
</a:clrScheme>
```

## Step 3 — Translate to LaTeX `\definecolor`

Pick a naming prefix `<Inst>` (e.g. `Acme`, `MIT`, `Stanford`). Avoid generic names like `Purple`, `Blue` — they could clash with other packages.

```latex
% Primary
\definecolor{<Inst>Primary}{HTML}{4A55A2}       % accent1

% Secondary primary variants (if the brand defines them)
\definecolor{<Inst>PrimaryDark}{HTML}{332085}   % pressed / hover (if separate)
\definecolor{<Inst>PrimaryLight}{HTML}{6746EB}  % link colour (or use directly)
\definecolor{<Inst>PrimarySoft}{HTML}{9E92E8}   % followed-link (or background)

% Neutrals
\definecolor{<Inst>Cream}{HTML}{F0EDE6}         % lt2
\definecolor{<Inst>Black}{HTML}{222222}         % dk2 (body text)
\definecolor{<Inst>Grey}{HTML}{7F7F7F}          % derived; not always in theme

% Accents
\definecolor{<Inst>Green}{HTML}{00B356}         % accent2
\definecolor{<Inst>Orange}{HTML}{ED7122}        % accent3
\definecolor{<Inst>Cyan}{HTML}{36B7F6}          % accent4
\definecolor{<Inst>Red}{HTML}{D52515}           % accent5
\definecolor{<Inst>Yellow}{HTML}{F6D62A}        % accent6
```

If the theme provides fewer accents (e.g. only 3 instead of 6), document the gaps in the README rather than inventing colours.

## Step 4 — Bind to Beamer roles

Once the palette is defined, wire each colour into Beamer's role-based system:

```latex
% --- Structure colour drives the whole cascade ---
\setbeamercolor{structure}{fg=<Inst>Primary}

% --- Text colours ---
\setbeamercolor{normal text}{fg=<Inst>Black,bg=white}
\setbeamercolor{alerted text}{fg=<Inst>Red}
\setbeamercolor{example text}{fg=<Inst>Green}

% --- Frame title ---
\setbeamercolor{frametitle}{fg=<Inst>Primary,bg=white}
\setbeamercolor{framesubtitle}{fg=<Inst>Grey}

% --- Cover slide (white text on coloured bg) ---
\setbeamercolor{title}{fg=white}
\setbeamercolor{subtitle}{fg=white}
\setbeamercolor{author}{fg=white}
\setbeamercolor{institute}{fg=white}
\setbeamercolor{date}{fg=white}

% --- Itemize / enumerate bullets ---
\setbeamercolor{itemize item}{fg=<Inst>Primary}
\setbeamercolor{itemize subitem}{fg=<Inst>PrimaryLight}
\setbeamercolor{itemize subsubitem}{fg=<Inst>PrimarySoft}
\setbeamercolor{enumerate item}{fg=<Inst>Primary}

% --- TOC entries ---
\setbeamercolor{section in toc}{fg=<Inst>Black}
\setbeamercolor{subsection in toc}{fg=<Inst>Grey}

% --- Blocks ---
\setbeamercolor{block title}{fg=white,bg=<Inst>Primary}
\setbeamercolor{block body}{fg=<Inst>Black,bg=<Inst>Cream}
\setbeamercolor{block title alerted}{fg=white,bg=<Inst>Red}
\setbeamercolor{block body alerted}{fg=<Inst>Black,bg=<Inst>Red!8}
\setbeamercolor{block title example}{fg=white,bg=<Inst>Green}
\setbeamercolor{block body example}{fg=<Inst>Black,bg=<Inst>Green!8}

% --- Footer ---
\setbeamercolor{footline}{fg=<Inst>Primary,bg=white}
\setbeamercolor{page number in head/foot}{fg=<Inst>Primary}
```

## Step 5 — Handle the link colour

Hyperref's `\hypersetup{colorlinks=true,allcolors=<Inst>Primary}` works for most institutional themes (link colour matches the brand). If the theme defines a *separate* hlink colour (different from accent1), expose both:

```latex
\definecolor{<Inst>Link}{HTML}{6746EB}     % from <a:hlink>
\hypersetup{colorlinks=true,
            urlcolor=<Inst>Link,
            linkcolor=<Inst>Primary,
            citecolor=<Inst>Primary}
```

## Step 6 — Validate

Compile a test deck containing each element and visually check that:

- Bullets, frame title, and block-title backgrounds all show the primary brand colour.
- Cover slide background is the primary colour (full-bleed); cover text is white.
- Block bodies show the cream (lt2) colour, not pure white — otherwise the warm tone is lost.
- Alerted blocks show red; example blocks show green.
- Section-divider list highlights the current section in white (on coloured bg) or primary (on white bg).
- Frame-number and short-title in the footer are visible (primary colour on white background).
- Links render in the brand colour and are underlined/coloured on hover (in PDF viewers that support it).

## Common pitfalls

- **Generic colour names clash with `xcolor`'s svgnames / dvipsnames.** Always prefix with the institution name. Don't use `Purple`, `Red`, etc. on their own.
- **The same HEX appearing twice in the palette** usually means it serves two roles (e.g. primary as both accent1 and hlink). Document this in the README rather than inventing different names.
- **Pure black (`#000000`) is rarely the right body-text colour.** Most institutional themes use `#222222` or similar to soften the contrast. Check `<a:dk2>` for the actual body-text colour.
- **Backgrounds aren't always pure white.** Some themes use `#F8F7F4` or similar warm off-white. Surface this in `\setbeamercolor{normal text}{bg=...}` rather than defaulting to `white`.
