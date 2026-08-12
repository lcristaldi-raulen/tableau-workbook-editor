---
name: tableau-workbook-editor
description: >
  Edit Tableau packaged workbook (.twbx) files programmatically — replace fonts, extract
  custom shapes and images, analyze navigation structure, modify colors, swap images, and
  inspect calculated fields. Use this skill whenever the user mentions .twbx, Tableau
  workbook, Tableau packaged workbook, or wants to batch-edit properties across a Tableau
  file without opening Tableau Desktop. Also trigger when the user asks to extract assets
  from a Tableau file, audit fonts or colors in dashboards, or understand the internal
  structure of a workbook. Even if the user just says "change fonts in my dashboard" or
  "pull out the images from my Tableau file" — use this skill.
---

# Tableau Workbook Editor

A skill for programmatically reading and modifying Tableau packaged workbook (.twbx) files.
Tableau workbooks are ZIP archives containing XML, data extracts, and images. This skill
teaches you how to safely manipulate them.

## How .twbx files work

A `.twbx` file is a ZIP archive with this structure:

```
workbook.twbx
├── workbook.twb          # XML file — all workbook configuration lives here
├── Data/Extracts/        # .hyper files (Tableau data extracts)
└── Image/                # Dashboard images (SVG, PNG) used by navigation buttons, backgrounds, etc.
```

The `.twb` file is XML and contains everything: data source definitions, worksheet
configurations, dashboard layouts, calculated fields, formatting, navigation actions,
custom shape images (base64-encoded), and style rules. All editing happens in this file.

**Critical rule**: Tableau validates the .twb against a strict internal DTD. You cannot
add new XML elements or attributes that aren't part of Tableau's schema. If you do,
the workbook will fail to open with validation errors. Only modify *values* of existing
attributes, or use patterns documented in this skill.

## Setup: Extract and repackage

Always follow this three-step pattern:

### Step 1: Extract

```python
import zipfile
with zipfile.ZipFile("workbook.twbx", "r") as zf:
    zf.extractall("work_dir")
```

### Step 2: Modify the .twb

Read the XML, make changes, write it back. Details in sections below.

### Step 3: Repackage

Always use Python's `zipfile` module — never shell `zip`, which can hit permission
errors and silently produce corrupt files.

```python
import zipfile, os

with zipfile.ZipFile("output.twbx", "w", zipfile.ZIP_DEFLATED) as zf:
    # Add the .twb
    zf.write("work_dir/workbook.twb", "workbook.twb")
    # Add Data/ and Image/ recursively
    for folder in ["Data", "Image"]:
        folder_path = os.path.join("work_dir", folder)
        if os.path.exists(folder_path):
            for root, dirs, files in os.walk(folder_path):
                for f in files:
                    fp = os.path.join(root, f)
                    arcname = os.path.relpath(fp, "work_dir")
                    zf.write(fp, arcname)

# Always verify
with zipfile.ZipFile("output.twbx", "r") as zf:
    assert zf.testzip() is None, "Zip integrity check failed"
```

After creating the file, verify it with `zf.testzip()`. Report the file count and size
to the user so they know it packaged correctly.

## Operations

### Font replacement

Fonts appear as `fontname='FontName'` attribute values throughout the .twb. A simple
string replacement works because font names don't appear in other contexts.

```python
content = content.replace("Helvetica", "Tableau Light")
```

Before replacing, audit what's there:

```python
import re
fonts = re.findall(r"fontname='([^']*)'", content)
from collections import Counter
for font, count in Counter(fonts).most_common():
    print(f"  {font}: {count}")
```

Also check for fonts referenced in `<run>` elements (formatted text):

```python
runs = re.findall(r"font='([^']*)'", content)
```

**Tableau Cloud font compatibility**: If the user is publishing to Tableau Cloud, only
certain fonts are available server-side. Common safe replacements:
- Helvetica/Helvetica Neue/Helvetica Light → Tableau Light, Tableau Book, or Arial
- The Tableau font family (Tableau Book, Tableau Light, Tableau Regular, Tableau
  Semibold, Tableau Bold) is always available on Tableau Cloud.

### Color modification

Colors appear as hex values in multiple contexts:

```python
# Style rules
re.findall(r"color='(#[0-9a-fA-F]{6})'", content)

# Format elements
re.findall(r"value='(#[0-9a-fA-F]{6})'", content)

# Font colors in formatted text
re.findall(r"fontcolor='(#[0-9a-fA-F]{6})'", content)
```

To replace a specific color across the workbook:

```python
content = content.replace("#d9261c", "#ff5733")
```

Be careful with broad color replacements — audit occurrences first and confirm with the
user, since a single hex code might be used in unexpected places.

### Analyzing navigation structure

Tableau dashboards use two navigation mechanisms:

**1. Button objects** (`<button action='tabdoc:goto-sheet'>`):
These are image-based navigation buttons in dashboard zones. Structure:

```xml
<zone type-v2='dashboard-object'>
  <button action='tabdoc:goto-sheet window-id=&quot;{GUID}&quot;'>
    <button-visual-state>
      <image-path>Image/Index Nav Dark.svg</image-path>
    </button-visual-state>
  </button>
</zone>
```

**2. Nav-actions** (`<nav-action>`):
These are navigation actions triggered by clicking worksheets:

```xml
<nav-action caption='Return Home' name='[Action7_...]'>
  <source dashboard='Index: View 1' type='sheet' worksheet='Brand Nav' />
</nav-action>
```

**Mapping button destinations**: Button GUIDs map to sheets via `<simple-id>` inside
`<window>` elements:

```python
# Build UUID → sheet name map
uuid_map = {}
for m in re.finditer(r"<window\b([^>]*)>(.*?)</window>", content, re.DOTALL):
    name = re.search(r"name='([^']*)'", m.group(1))
    uuid = re.search(r"<simple-id uuid='\{([^}]+)\}'", m.group(2))
    if name and uuid:
        uuid_map[uuid.group(1)] = name.group(1)
```

### Extracting custom shapes

Custom shapes are base64-encoded images embedded in `<shapes>` blocks:

```python
import base64, os

shapes_block = re.compile(r"<shapes[^>]*>(.*?)</shapes>", re.DOTALL)
shape_pattern = re.compile(r"<shape name='([^']*)'>\s*(.*?)\s*</shape>", re.DOTALL)

for block in shapes_block.finditer(content):
    for m in shape_pattern.finditer(block.group(1)):
        name = m.group(1)
        b64 = m.group(2).replace('\n', '').replace(' ', '')
        os.makedirs(os.path.dirname(name), exist_ok=True)
        with open(name, 'wb') as f:
            f.write(base64.b64decode(b64))
```

Shape names include their palette directory (e.g., `BP Brands/starbucks.png`,
`Custom/square.png`). Preserve this structure when exporting.

Dashboard images (SVGs, PNGs used by navigation buttons and backgrounds) are already
in the `Image/` folder inside the .twbx — just copy them out.

### Replacing dashboard images

To swap an image (e.g., update a logo), replace the file in the `Image/` folder before
repackaging. The filename must match exactly what the .twb references in `<image-path>`.

### Inspecting calculated fields

Calculated fields are in `<column>` elements with `<calculation>` children:

```python
calc_pattern = re.compile(
    r"<column[^>]*caption='([^']*)'[^>]*>\s*<calculation[^>]*formula='([^']*)'",
    re.DOTALL
)
for m in calc_pattern.finditer(content):
    print(f"{m.group(1)}: {m.group(2)}")
```

Note: formulas use `&quot;` for quotes, `&amp;` for ampersands, `&lt;`/`&gt;` for
angle brackets. Decode these for readability when presenting to the user.

### Inspecting and modifying tooltips

**Worksheet tooltips** are in `<customized-tooltip>` blocks within each `<worksheet>`.
These contain `<formatted-text>` with `<run>` elements:

```python
ws_pattern = re.compile(r"<worksheet name='([^']*)'>(.*?)</worksheet>", re.DOTALL)
tt_pattern = re.compile(r"<customized-tooltip[^>]*>(.*?)</customized-tooltip>", re.DOTALL)

for ws in ws_pattern.finditer(content):
    for tt in tt_pattern.finditer(ws.group(2)):
        runs = re.findall(r"<run[^>]*>(.*?)</run>", tt.group(1))
        text = " ".join(runs)
        if text.strip():
            print(f"{ws.group(1)}: {text}")
```

You can modify the text inside `<run>` elements to change worksheet tooltip content.

### Adding field descriptions (Default Properties > Comment)

Field descriptions are stored as `<desc>` child elements inside `<column>` elements.
These correspond to what users see in Tableau Desktop under right-click → Default
Properties → Comment.

**Reading existing descriptions:**

```python
desc_pattern = re.compile(
    r"<column\b[^>]*caption='([^']*)'[^>]*>\s*<desc>\s*<formatted-text>\s*<run[^>]*>(.*?)</run>",
    re.DOTALL
)
for m in desc_pattern.finditer(content):
    print(f"{m.group(1)}: {m.group(2)}")
```

**Adding a description to a field:**

Many `<column>` elements are self-closing (`/>`). To add a `<desc>`, convert them to
open/close tags and insert the description block:

```python
# Before: <column caption='Field Name' ... />
# After:  <column caption='Field Name' ...>
#           <desc><formatted-text><run>Description here</run></formatted-text></desc>
#         </column>

old = "<column caption='Brand Name' datatype='string' name='[BRAND_NAME]' role='dimension' type='nominal' />"
new = """<column caption='Brand Name' datatype='string' name='[BRAND_NAME]' role='dimension' type='nominal'>
        <desc>
          <formatted-text>
            <run>The display name of the brand in the index</run>
          </formatted-text>
        </desc>
      </column>"""
content = content.replace(old, new)
```

**Important**: A field name like `[BRAND_NAME]` may appear in multiple datasource
blocks within the same .twb. Decide whether to add descriptions to all instances or
just the primary datasource. Use `content.replace(old, new, 1)` to limit to the first
occurrence if needed.

**Status**: The `<desc>` element with `<formatted-text><run>` children follows the same
pattern as `<customized-tooltip>` and `<customized-label>`, which are validated by
Tableau's DTD. Test any new description additions by opening the resulting workbook in
Tableau before delivering in bulk.

### Known limitations — what you CANNOT change via XML

Tableau's internal DTD strictly validates the .twb file. These operations will cause the
workbook to fail to open:

- **Navigation button tooltips**: The `<button>` element only allows `action` as an
  attribute and `<button-visual-state>` as children. Adding `tooltip` as an attribute
  or child element causes DTD validation failure. The auto-generated "Goes to [sheet]"
  tooltip can only be customized through Tableau Desktop's UI (Edit Button → Tooltip).

- **Adding new XML elements**: Any element not in Tableau's schema will be rejected.

- **Adding undeclared attributes**: Any attribute not declared for its element will be
  rejected.

When you encounter something that can't be done via XML, tell the user clearly and
provide step-by-step instructions for doing it in Tableau Desktop instead.

## Quality checklist

Before delivering the modified workbook:

1. Verify the zip is valid (`zf.testzip() is None`)
2. Confirm the file count matches the original
3. For font changes: grep the .twb to confirm zero instances of the old font remain
4. For color changes: audit that only intended colors were changed
5. Report the file size — it should be roughly similar to the original
