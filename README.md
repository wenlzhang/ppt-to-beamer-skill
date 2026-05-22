# PPT to Beamer Skill

A Claude Skill that converts a PowerPoint institutional template (`.potx` / `.pptx`) into a faithful LaTeX Beamer theme.

Drop your institution's PowerPoint template, brand-guidelines PDF, and logo PNGs in front of Claude Code with this skill installed, and you get back a ready-to-publish folder with `beamertheme<Name>.sty`, a self-documenting tutorial deck, all the brand assets correctly placed, plus `LICENSE`, `NOTICE`, `README.md`, and `.gitignore`.

The skill encodes the *methodology* of the conversion — colour palette extraction from theme XML, slide-layout reading from the OpenXML master, the Beamer-internals quirks to work around, the dual-licence wording. It does not assume any specific institution; the same skill produces a working Beamer theme for any university, lab, or company brand whose PowerPoint master and logos you can supply.

## What this skill is

A Claude Skill is a Markdown file (`SKILL.md`) with YAML frontmatter that Claude Code matches against user prompts. When the user provides a PowerPoint template and asks for a Beamer equivalent, Claude reads `SKILL.md`, follows the encoded workflow, and consults the `reference/` files as needed.

The skill is **portable**: clone the repo into `~/.claude/skills/ppt-to-beamer/` (or anywhere on the skills path) and Claude Code picks it up automatically.

## Installation

```bash
git clone https://github.com/<your-handle>/ppt-to-beamer-skill ~/.claude/skills/ppt-to-beamer
```

Or, if you maintain a Claude Code plugin manifest, add this repo as a plugin dependency.

## Usage

Once installed, simply talk to Claude Code:

> *"I have my university's PowerPoint template at `~/Downloads/uni-template.potx` and the official logos in `~/Downloads/uni-logos/`. Please convert this into a Beamer theme that mimics the original look."*

Claude will:

1. Extract and inspect the `.potx` XML structure.
2. Identify the colour palette, fonts, slide-layout geometry, and embedded media.
3. Generate a complete Beamer package — `.sty` + tutorial `.tex` + `logographics/` + `LICENSE` + `NOTICE` + `README.md`.
4. Compile the tutorial deck twice with `pdflatex` to verify it renders cleanly.
5. Optionally create a GitHub repository and push the result (with your approval).

You provide:

- The `.potx` / `.pptx` template.
- Optionally: official logo PNGs in colour / white / black variants (otherwise Claude uses whatever is embedded in the `.potx`).
- Optionally: a brand-guidelines PDF / web page to inform the unofficial / official phrasing of the NOTICE file.
- The target folder for the output (or let Claude pick a sensible default).

## What you get back

A folder structured like:

```
<repo-name>/
├── .gitignore
├── LICENSE                       MIT covering the LaTeX code only
├── NOTICE                        Brand-asset statement (logos remain institution IP)
├── README.md                     Single-file command reference + customisation guide
├── beamertheme<Name>.sty         The theme (colours, fonts, layouts, dispatchers)
├── <name>.tex                    Self-documenting tutorial deck (~25 slides)
└── logographics/                 Brand PNGs (logo, wordmark, emblem variants)
```

The output is designed to be *publishable*: drop it into a public GitHub repo with confidence that no personal information, no internal-only assets, and no official-endorsement claims have leaked into the published files.

## Limitations and honest caveats

- **Font substitution is approximate.** Most institutional templates use Arial / Calibri / Helvetica Neue — none of which ship freely with TeX Live. The skill uses Helvetica (`helvet`) or Calibri-clone (`carlito`) as the closest substitute and documents the trade-off in the NOTICE file.
- **Logo placement is heuristic.** The skill reads the PowerPoint master's placeholder geometry and approximates it in Beamer, but PowerPoint's pixel-level positioning system doesn't map perfectly onto LaTeX's text-flow model. Expect millimetre-level drift from the original.
- **Trademark assets stay trademark assets.** This skill does *not* relicense any institutional logos. The `NOTICE` file makes this explicit; you are responsible for honouring your institution's brand-use guidelines when sharing the resulting template.
- **Brand-portal-only assets are not auto-fetched.** If your institution's logos are behind a single-sign-on portal, the skill cannot retrieve them; you must download them and supply them as input.

## Contributing

This skill grew out of one author's conversion of a specific institutional template. The encoded workflow is therefore opinionated — it reflects the gotchas they encountered, not a comprehensive survey of every possible PowerPoint master layout.

Pull requests welcome, especially:

- New `reference/<institution-pattern>.md` notes documenting layout quirks of specific institutional templates.
- Improvements to `reference/beamer-internals.md` covering Beamer compatibility issues with newer / older TeX Live releases.
- Worked example folders under `examples/` showing the skill applied to publicly-licensed PowerPoint templates (e.g. an open-source brand kit, a creative-commons template).

## Licence

This skill repository ships under the **MIT licence** — see `LICENSE`. You may use, modify, redistribute, and sublicense it with attribution.

Outputs the skill produces are subject to the licences of the assets they contain:

- The LaTeX `.sty` / `.tex` / README that the skill *generates* are MIT-licensable by you (the user) — the skill imposes no restrictions on derivative work.
- The institutional logos that you supply as input remain the trademark of the originating institution. The skill copies them into the output unchanged; it does *not* relicense them.

See `NOTICE` for the brand-asset statement template the skill embeds in generated repositories.
