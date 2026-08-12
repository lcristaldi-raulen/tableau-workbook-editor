[README.md](https://github.com/user-attachments/files/30989697/README.md)
# Tableau Workbook Editor

A [Claude Skill](https://code.claude.com/docs/en/skills) for editing Tableau packaged workbook
(`.twbx`) files programmatically — no Tableau Desktop required.

A `.twbx` is a ZIP archive containing a `.twb` XML file, data extracts, and images. Everything
about a workbook — fonts, colors, layout zones, calculated fields, tooltips, navigation actions,
embedded shapes — lives in that XML. This skill teaches Claude how to extract, modify, and safely
repackage it, including the parts Tableau's internal DTD will reject.

## What it does

- **Font replacement** — swap fonts across an entire workbook (e.g. Helvetica → Tableau Light for Tableau Cloud compatibility)
- **Color modification** — find and replace hex values across style rules, formats, and formatted text
- **Navigation analysis** — map every navigation button to its destination sheet via zone GUIDs
- **Custom shape export** — extract base64-encoded shape images to files, preserving palette folders
- **Image replacement** — swap dashboard images such as logos, nav buttons, and backgrounds
- **Calculated field inspection** — list every calculated field and its decoded formula
- **Tooltip editing** — read and rewrite worksheet tooltip content
- **Field descriptions** — add or read field comments (Default Properties → Comment)
- **Accessibility work** — inspect and reorder dashboard zone tab/focus order for WCAG 2.4.3

## Install

The repository *is* the skill directory, so cloning it into your skills folder is the whole install.

**For yourself, in every project:**

```bash
git clone https://github.com/OWNER/tableau-workbook-editor \
  ~/.claude/skills/tableau-workbook-editor
```

**For one project, shared with your team via version control:**

```bash
git clone https://github.com/OWNER/tableau-workbook-editor \
  .claude/skills/tableau-workbook-editor
```

Restart Claude Code (or run `/reload-plugins`) and the skill loads automatically — no install
command or marketplace needed. To update later, `git pull` in that directory.

Prefer not to use git? Download `SKILL.md` and place it at
`~/.claude/skills/tableau-workbook-editor/SKILL.md`. That single file is the entire skill.

## Usage

Attach or point Claude at a `.twbx` file and ask in plain language. The skill triggers on its own:

- "Change all Helvetica fonts to Tableau Light in this workbook"
- "Extract all custom shape images from this Tableau file"
- "Show me the navigation structure of this dashboard"
- "What calculated fields are in this workbook?"
- "Replace #d9261c with #ff5733 across the workbook"
- "Fix the tab order in this dashboard for ADA compliance"

## Requirements

None beyond a Python 3 interpreter. The skill uses only the standard library — `zipfile`, `re`,
`base64`, `os` — so there is nothing to install and no external service to configure.

## Known limitations

Tableau validates the `.twb` against a strict internal DTD. You can change the *values* of existing
attributes, but adding elements or attributes that aren't in Tableau's schema makes the workbook
fail to open. Specifically:

- **Navigation button tooltips** can only be set through the Tableau Desktop UI (Edit Button →
  Tooltip). The `<button>` element accepts no `tooltip` attribute.
- **New XML elements or attributes** of any kind are rejected.

The skill documents these boundaries and falls back to step-by-step Tableau Desktop instructions
when an edit isn't possible in XML.

## Verifying edits

The skill always repackages with Python's `zipfile` (never shell `zip`, which can silently produce
corrupt archives) and checks integrity with `testzip()`. Even so, open the result in Tableau Desktop
before shipping it — Tableau's DTD validation can't be reproduced outside the product.

---

Built by [XeoMatrix](https://www.xeomatrix.com).
