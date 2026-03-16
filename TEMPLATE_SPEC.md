# TEMPLATE_SPEC.md — vizli Template Specification v0.1

## Overview

A **vizli template** is a self-contained file that describes a visualization
with parameterized data slots, rendering hints, and output expectations. The
template is the unit of work in the vizli pipeline: an LLM agent produces a
template from a raw sample, and the vizli CLI renders it into output files.

This document defines the template format, the five supported template types,
the data binding conventions, and the instructions an LLM agent follows when
converting raw visualization code into a conformant template.

---

## 1. Template anatomy

Every template is a single file with two sections:

1. **YAML frontmatter** — metadata, parameters, rendering hints
2. **Body** — the visualization definition (Vega-Lite JSON, Altair Python,
   SVG markup, HTML+JS, or raw JS)

The frontmatter is delimited by `---` fences. The body follows immediately.

```
---
# YAML frontmatter
template_version: "0.1"
type: vega-lite
meta:
  title: "Monthly revenue by region"
  description: "Grouped bar chart with region color encoding"
  tags: [bar-chart, revenue, regional]
output:
  formats: [svg, png]
  width: 800
  height: 500
  scale: 2
params:
  - name: data_url
    type: url
    required: true
    description: "CSV or JSON endpoint for revenue data"
  - name: color_scheme
    type: string
    default: "tableau10"
    description: "Vega color scheme name"
data:
  inline: null
  sample: |
    region,month,revenue
    East,Jan,12000
    East,Feb,13500
    West,Jan,9800
    West,Feb,11200
---
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {"url": "{{data_url}}"},
  "mark": "bar",
  "encoding": {
    "x": {"field": "month", "type": "nominal"},
    "y": {"field": "revenue", "type": "quantitative"},
    "color": {"field": "region", "scale": {"scheme": "{{color_scheme}}"}}
  }
}
```

---

## 2. Frontmatter schema

### Required fields

| Field | Type | Description |
|---|---|---|
| `template_version` | string | Always `"0.1"` for this spec version |
| `type` | enum | One of: `vega-lite`, `altair`, `svg`, `html`, `js` |
| `meta.title` | string | Human-readable title for the visualization |
| `output.formats` | list | Target formats: `svg`, `png`, `html`, `pdf` |

### Optional fields

| Field | Type | Default | Description |
|---|---|---|---|
| `meta.description` | string | `""` | What this visualization shows |
| `meta.tags` | list[string] | `[]` | Searchable tags for template discovery |
| `meta.author` | string | `""` | Who created or converted this template |
| `output.width` | int | `800` | Output width in pixels |
| `output.height` | int | `500` | Output height in pixels |
| `output.scale` | float | `2` | Pixel density multiplier for PNG (1 = 72dpi, 2 = 144dpi) |
| `output.background` | string | `"white"` | Background color; `"transparent"` for no bg |
| `output.theme` | enum | `"light"` | `light`, `dark`, or `auto` |
| `params` | list[Param] | `[]` | Parameterized slots (see §3) |
| `data.inline` | object/null | `null` | Inline data object (embedded in template) |
| `data.sample` | string/null | `null` | Sample data for preview rendering |
| `data.schema` | object/null | `null` | JSON Schema describing expected data shape |
| `rendering.engine` | enum | auto | Override rendering engine (see VIZLI_OUTPUT_SPEC) |
| `rendering.timeout` | int | `30` | Max seconds for rendering |
| `rendering.interactive` | bool | auto | Whether output supports interaction |

### Param schema

Each entry in the `params` list:

```yaml
params:
  - name: identifier        # snake_case, unique within template
    type: string|int|float|bool|url|color|enum|json
    required: true|false     # if false, must have `default`
    default: value           # used when param not supplied at CLI
    description: "..."       # human-readable, shown in --help
    enum_values: [a, b, c]   # only for type: enum
    min: 0                   # only for int/float
    max: 100                 # only for int/float
```

---

## 3. Template types

### 3.1 `vega-lite`

**Body format**: JSON conforming to the Vega-Lite v5 schema.

**Parameter interpolation**: Use `{{param_name}}` Mustache-style placeholders
in the JSON body. vizli performs string interpolation before passing to the
Vega-Lite compiler. For complex data substitutions, use the `data.inline`
frontmatter field rather than interpolating into the JSON body.

**Data binding**:
- `"data": {"url": "{{data_url}}"}` — external data source
- `"data": {"values": "__INLINE_DATA__"}` — vizli replaces the sentinel
  `"__INLINE_DATA__"` with `data.inline` from frontmatter or `--data` CLI arg
- `"data": {"values": "__SAMPLE_DATA__"}` — uses `data.sample` for previews

**What the LLM agent should produce**: A valid Vega-Lite JSON spec with all
hardcoded data extracted into params or the `data` frontmatter section. The
spec should use `"autosize": {"type": "fit", "contains": "padding"}` to
respect the output dimensions from frontmatter.

**Example body**:
```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "autosize": {"type": "fit", "contains": "padding"},
  "data": {"values": "__INLINE_DATA__"},
  "mark": {"type": "line", "point": true},
  "encoding": {
    "x": {"field": "date", "type": "temporal", "title": "{{x_label}}"},
    "y": {"field": "value", "type": "quantitative", "title": "{{y_label}}"},
    "color": {"field": "series", "type": "nominal"}
  }
}
```

---

### 3.2 `altair`

**Body format**: A Python script that produces an `altair.Chart` object. The
script must assign the final chart to a variable named `chart`.

**Parameter interpolation**: Parameters are injected as Python variables
before the script executes. The LLM agent should reference them directly
as bare identifiers (not string interpolation).

**Data binding**: Data is available as a pandas DataFrame named `data` if
provided via `--data` or `data.inline`. Sample data is loaded into `data`
for preview renders.

**What the LLM agent should produce**: A Python script using Altair's API.
No `import` statements needed — `altair` is available as `alt` and `pandas`
as `pd`. The script must end by assigning to `chart`.

**Example body**:
```python
chart = alt.Chart(data).mark_bar().encode(
    x=alt.X('month:N', title=x_label),
    y=alt.Y('revenue:Q', title=y_label),
    color=alt.Color('region:N', scale=alt.Scale(scheme=color_scheme))
).properties(
    title=title
)
```

---

### 3.3 `svg`

**Body format**: Raw SVG markup. Must be a complete `<svg>` element with
`xmlns` attribute and explicit `viewBox`.

**Parameter interpolation**: `{{param_name}}` placeholders in attribute
values and text content. vizli replaces before writing output.

**Data binding**: For data-driven SVGs, the LLM agent should include a
`<script>` block at the end of the SVG that reads from a global
`__VIZLI_DATA__` variable (injected by vizli as JSON) and manipulates
the DOM. For static SVGs, no data binding is needed.

**Sizing**: The SVG must use `viewBox="0 0 W H"` matching the frontmatter
`output.width` and `output.height`. Set `width="100%"` for responsive
scaling. vizli sets explicit `width` and `height` attributes at render
time for PNG conversion.

**What the LLM agent should produce**: A clean SVG with parameterized slots
for any values that should be configurable. All text must use system fonts
(no custom font dependencies). Colors should use CSS custom properties or
be parameterized for theme support.

**Dark mode**: If `output.theme` is `auto` or `dark`, the SVG should
include a `<style>` block with `@media (prefers-color-scheme: dark)` rules.
For PNG output, vizli forces one mode — it sets a `data-theme` attribute on
the root `<svg>` element and the template's CSS should respect it:

```svg
<style>
  svg[data-theme="dark"] .bg { fill: #1a1a18; }
  svg[data-theme="dark"] .text-primary { fill: #e8e6dc; }
</style>
```

---

### 3.4 `html`

**Body format**: A complete HTML document (with `<!DOCTYPE html>`) containing
embedded SVG, CSS, and JavaScript. This is the type for interactive
visualizations — calculators, steppers, illustrative explainers.

**Parameter interpolation**: `{{param_name}}` in both HTML content and
`<script>` blocks. vizli performs interpolation before writing. For complex
JS objects, use `type: json` params and reference as `{{param_name}}` inside
a script — vizli serializes to JSON.

**Data binding**: Data is injected as a `<script>` block before the template's
own scripts:
```html
<script>const __VIZLI_DATA__ = {{__DATA_JSON__}};</script>
```
The template's JS reads from `__VIZLI_DATA__`.

**Output modes**:
- `formats: [html]` — vizli writes the interpolated HTML file directly.
  Interactive controls, animations, and JS all work.
- `formats: [png]` or `formats: [svg, png]` — vizli opens the HTML in a
  headless browser, waits for rendering, and screenshots. The template
  should fire a `window.__VIZLI_READY__` event or set
  `window.__VIZLI_READY__ = true` when the visual is fully rendered.
  vizli polls for this flag (timeout from `rendering.timeout`).

**What the LLM agent should produce**: A self-contained HTML file following
the structure from the explainer skill. Zero external dependencies except
Chart.js from cdnjs if needed. Light/dark mode via CSS custom properties.
The `<title>` should reflect `meta.title`.

**Readiness signal** (required for PNG/SVG capture from HTML):
```javascript
// At the end of initialization:
window.__VIZLI_READY__ = true;
```

---

### 3.5 `js`

**Body format**: A JavaScript module that exports a `render(container, params, data)`
function. This is for D3.js visualizations and other programmatic renderers.

**Execution environment**: vizli loads the script in a headless browser page.
It creates a container `<div>` and calls `render()`. D3 v7 is available
globally (loaded by vizli's HTML harness).

**What the LLM agent should produce**: A JS file with a default export or
a named `render` function:

```javascript
function render(container, params, data) {
  const svg = d3.select(container)
    .append('svg')
    .attr('viewBox', `0 0 ${params.width} ${params.height}`);

  // ... D3 rendering logic ...

  window.__VIZLI_READY__ = true;
}
```

**Available globals**: `d3` (v7), `__VIZLI_DATA__` (the data object),
`__VIZLI_PARAMS__` (the resolved params object).

---

## 4. Data binding conventions

### Data flow

```
CLI --data flag  ──→  frontmatter data.inline  ──→  data.sample
    (highest priority)     (medium)                  (lowest / preview)
```

vizli resolves data in priority order:
1. `--data path/to/file.csv` or `--data path/to/file.json` from CLI
2. `data.inline` in frontmatter (embedded directly)
3. `data.sample` in frontmatter (for preview/development)

### Supported data formats

| Format | Detection | Loaded as |
|---|---|---|
| CSV | `.csv` extension or first-line comma detection | Array of objects |
| TSV | `.tsv` extension | Array of objects |
| JSON | `.json` extension or starts with `[` or `{` | Parsed JSON |
| NDJSON | `.ndjson` extension | Array of objects |
| Parquet | `.parquet` extension | Array of objects (via pyarrow) |

### Data transformation params

Templates can specify lightweight transforms in frontmatter:

```yaml
data:
  transforms:
    - filter: "revenue > 0"
    - derive:
        profit_margin: "(revenue - cost) / revenue"
    - aggregate:
        group_by: [region, quarter]
        ops: {revenue: sum, profit_margin: mean}
```

vizli applies these using pandas before injecting data into the template.
This keeps templates declarative — the rendering logic stays clean.

---

## 5. LLM agent conversion instructions

This section is the prompt context for an LLM agent (such as Claude Opus 4.6)
that converts raw visualization samples into vizli templates.

### The conversion task

You receive a **raw sample** — a Vega-Lite spec, an Altair script, an SVG
file, an HTML page with embedded charts, or a D3 script — and you produce a
**vizli template** conforming to this spec.

### Step-by-step conversion process

#### Step 1: Classify the sample

Determine the template type from the input:

| Input looks like | Template type |
|---|---|
| JSON with `$schema` containing `vega-lite` | `vega-lite` |
| Python code importing `altair` or using `alt.Chart` | `altair` |
| Raw `<svg>` element (no HTML wrapper) | `svg` |
| Full HTML document with `<script>` and interactivity | `html` |
| JavaScript with D3 calls or a `render()` function | `js` |

If the sample mixes types (e.g., HTML with an embedded Vega-Lite spec),
prefer the outer container type (`html`).

#### Step 2: Extract parameters

Scan the sample for hardcoded values that a user would want to change:

- **Data sources**: URLs, file paths, inline data arrays → extract to
  `data.inline` or a `data_url` param
- **Labels and titles**: chart titles, axis labels → string params
- **Visual properties**: color schemes, font sizes, opacity → typed params
- **Thresholds and constants**: statistical parameters, breakpoints → numeric params
- **Feature flags**: whether to show a legend, grid lines, annotations → bool params

Naming conventions for params:
- Use `snake_case`
- Prefix data-related params with `data_` (e.g., `data_url`, `data_path`)
- Prefix color params with `color_` (e.g., `color_scheme`, `color_primary`)
- Use descriptive names: `min_detectable_effect` not `mde`

#### Step 3: Extract sample data

If the sample contains inline data:

1. Move it to `data.sample` in the frontmatter (as CSV or compact JSON)
2. Replace the inline data in the body with the appropriate sentinel
   (`__INLINE_DATA__`, `__SAMPLE_DATA__`, or `__VIZLI_DATA__`)
3. Optionally write a `data.schema` describing the expected columns/fields

Keep sample data to 10-20 rows — enough to verify rendering, small enough
to not bloat the template file.

#### Step 4: Add sizing and output metadata

Determine appropriate dimensions from the sample:
- If the sample has explicit width/height → use those
- If it's a standard chart → default 800×500
- If it's a dashboard or wide viz → 1200×700
- If it's a mobile/narrow viz → 400×600

Set `output.formats` based on the template type:
- `vega-lite` and `altair` → `[svg, png]`
- `svg` → `[svg, png]`
- `html` → `[html, png]` (HTML for interactive, PNG for static capture)
- `js` → `[svg, png]` or `[html, png]` depending on interactivity

#### Step 5: Templatize the body

Replace extracted values with interpolation placeholders:

**For vega-lite and svg types**: Use `{{param_name}}` Mustache syntax.

**For altair type**: Reference params as bare Python variables.

**For html type**: Use `{{param_name}}` in both HTML and `<script>` blocks.
For data, reference `__VIZLI_DATA__` in JavaScript.

**For js type**: Reference `params.param_name` in the render function args.

#### Step 6: Add the readiness signal

For `html` and `js` types, ensure the template signals when rendering is
complete:

```javascript
// After all DOM updates and animations settle:
window.__VIZLI_READY__ = true;
```

For animated visualizations, fire the signal after the first frame renders
(not after all animations complete).

#### Step 7: Validate dark mode

For `svg` and `html` types:
- Ensure all colors use CSS custom properties or parameterized values
- Include dark mode overrides via `@media (prefers-color-scheme: dark)` or
  `data-theme` attribute support
- Test mentally: if the background were near-black, is every text element
  still readable?

For `vega-lite` and `altair` types: dark mode is handled by vizli's
rendering engine (it applies a dark theme config at render time). No
action needed in the template.

#### Step 8: Write the template file

Assemble the frontmatter and body into a single file. Use the `.vizli`
extension (vizli also accepts `.vizli.json`, `.vizli.py`, `.vizli.svg`,
`.vizli.html`, `.vizli.js` for editor syntax highlighting).

### Conversion quality checklist

Before finalizing the template, verify:

- [ ] `template_version: "0.1"` is present
- [ ] `type` correctly matches the body format
- [ ] All hardcoded data is extracted to `data.sample` or params
- [ ] All user-configurable values are parameterized
- [ ] Required params have `required: true`, optional params have `default`
- [ ] `output.formats` matches the visualization's nature (interactive → html)
- [ ] Sample data is ≤20 rows and representative
- [ ] Body is valid for its type (parseable JSON, valid SVG, syntactically
      correct Python/JS)
- [ ] Dark mode support is present for svg/html types
- [ ] Readiness signal present for html/js types
- [ ] No external dependencies except Chart.js and D3 (loaded by vizli)

---

## 6. File naming and organization

### Single templates

```
revenue-by-region.vizli           # generic extension
revenue-by-region.vizli.json      # vega-lite (editor highlights as JSON)
revenue-by-region.vizli.py        # altair (editor highlights as Python)
revenue-by-region.vizli.svg       # svg type
revenue-by-region.vizli.html      # html type
revenue-by-region.vizli.js        # js type
```

### Template collections

```
templates/
├── charts/
│   ├── bar-grouped.vizli.json
│   ├── line-timeseries.vizli.json
│   └── scatter-regression.vizli.json
├── dashboards/
│   ├── executive-summary.vizli.html
│   └── funnel-analysis.vizli.html
├── explainers/
│   ├── ab-test-power.vizli.html
│   ├── cuped-variance.vizli.html
│   └── gradient-descent.vizli.html
└── diagrams/
    ├── ci-cd-pipeline.vizli.svg
    └── data-architecture.vizli.svg
```

### Template registry (optional)

A `vizli-templates.yaml` file at the collection root enables discovery:

```yaml
version: "0.1"
templates:
  - path: charts/bar-grouped.vizli.json
    title: Grouped bar chart
    tags: [bar, comparison, categorical]
  - path: explainers/ab-test-power.vizli.html
    title: A/B test power calculator
    tags: [statistics, ab-test, interactive]
```

---

## 7. Template inheritance (future)

Reserved for v0.2. Templates will be able to extend a base template:

```yaml
extends: charts/base-timeseries.vizli.json
overrides:
  params:
    color_scheme:
      default: "dark2"
  meta:
    title: "Custom timeseries variant"
```

This is not yet implemented. Do not use `extends` in v0.1 templates.
