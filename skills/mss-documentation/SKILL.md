---
name: mss-documentation
description: Create company-branded MSS (MS Solutions Group) documentation as a Word .docx using the official MSS Word template (logo cover page, info table, auto table of contents, numbered MSS heading styles, branded tables, header/footer). Use this skill whenever the user asks for a document, guide, procedure, specification, report, policy, framework, process description, handover, user manual or any "documentation" for MSS or "our company" - even if they only say "write the doc", "document this", "make a Word document", or "use our template" and do not mention the template. Always prefer this skill over generic docx creation when the deliverable is an MSS/internal/company document. Do NOT use it for slides, spreadsheets, PDFs, or for editing an existing .docx the user uploaded.
---

# MSS documentation

Produces a **.docx built on the official MSS template** so every document has the same cover,
header/footer, colours, fonts and numbered headings. The template is never re-created in code:
`scripts/build_doc.py` opens `assets/Template_WordNew_Mss.docx` and only replaces the content.

```
mss-documentation/
├── SKILL.md
├── assets/Template_WordNew_Mss.docx   # official template (logos, styles, theme) - do not edit
├── scripts/build_doc.py               # JSON spec -> branded .docx
├── scripts/render_preview.py          # .docx -> PNG pages for visual QA
└── references/
    ├── spec_format.md                 # every block type + examples (read before writing a spec)
    ├── brand.md                       # colours, fonts, rules from the brand centre
    ├── structures.md                  # recommended outlines per document type
    └── example_spec.json              # complete working example
```

## Workflow

1. **Collect the inputs.** From the request and any attached material work out:
   `title`, `tagline` (0-2 short keywords shown on the cover), `owner` (team + contact email),
   `audience`, `purpose` (1-2 sentences), `version`. Ask the user only for what you cannot infer
   and that matters (usually owner/contact and audience). Use sensible defaults for the rest,
   and state the assumptions in your final message. The date defaults to today.
2. **Choose the structure.** Read `references/structures.md` and pick the outline that matches the
   document type, then adapt it to the content. Write the real content - no lorem ipsum.
3. **Write the spec** as JSON (format in `references/spec_format.md`) to `/home/claude/spec.json`.
4. **Build:**
   ```bash
   python /mnt/skills/user/mss-documentation/scripts/build_doc.py /home/claude/spec.json \
          -o /mnt/user-data/outputs/<File_Name>.docx
   ```
   (Use the skill's actual install path.) The script runs a second pass that renders the document
   to fill the table-of-contents page numbers.
5. **Check visually** - always, before delivering:
   ```bash
   python scripts/render_preview.py /mnt/user-data/outputs/<File_Name>.docx /home/claude/preview
   ```
   Look at the PNGs: cover title fits, info table is correct, TOC lists every heading with page
   numbers, no orphan headings at the bottom of a page, tables not cut awkwardly, images readable.
   Fix the spec and rebuild if needed.
6. **Deliver** with `present_files`, plus a 2-3 line summary: what the document contains, the
   assumptions made (owner, audience, version) and anything the user should confirm.

## Rules for a well-labelled document

- **File name:** `MSS_<Topic>_<DocType>_v<version>.docx`, e.g. `MSS_Payment_Gateway_Integration_Guide_v1.0.docx`
  (no spaces, no accents). Adjust if the user gives a naming convention.
- **Title** is short and descriptive; the cover shows it large, so keep it under ~60 characters.
  The same title is used in the footer and the document properties automatically.
- **Headings:** `h1` major sections, `h2` subsections, `h3` details, `h4` rare (not in the TOC).
  Never skip a level, never use bold text in place of a heading, no trailing colons or numbering
  in the text (numbering I. / 1. / a. / i. is automatic from the template styles).
  Every `h1`/`h2`/`h3` appears in the TOC, so write them as meaningful labels.
- **Tables:** give every table a `title`; header row text is short; use `widths` ratios so wide
  columns get more room. Figures get a `caption` ("Figure 1 - ...").
- **Bullets** for unordered facts, `numbered` for sequential steps, `note` for warnings/tips,
  `code` for commands and snippets, `kpis` (max 4) only for headline figures.
- **Language:** write the content in the language the user used. Structural labels in the
  template (info-table labels, "Table of content", footer) stay as designed.
- **Brand:** do not introduce other colours or fonts, do not restyle the template (see
  `references/brand.md`). All styling comes from the template and the builder.
- Do not leave empty sections; if information is missing, say so in a `note` ("To be confirmed: ...")
  and list it in your final message.

## Boundaries

- Editing an existing user-uploaded .docx, or producing slides/spreadsheets/PDF: use the matching
  other skill instead.
- If the user wants a different layout than the template offers (landscape, two columns),
  tell them it is outside the brand template and ask before deviating.
- If `build_doc.py` reports "Template layout changed", the template was modified: restore the
  official file in `assets/` or update the script's anchors (cover = body[1], info table = body[2],
  TOC = body[6], last element = section properties).

## Known behaviours

- TOC page numbers are filled via a LibreOffice render. If LibreOffice is unavailable the TOC shows
  placeholders; in Word, right-click the TOC -> Update field.
- The template fonts (Funnel Display / Funnel Sans) must be installed on the viewer's machine; otherwise
  Word falls back to Segoe UI per the brand guideline. Previews rendered on the server use a
  substitute font, so line breaks may differ slightly from Word.
- The footer prints the file name followed by the title (template design, `FILENAME` field).