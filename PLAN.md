# PLAN.md — vizli Development Plan

## Overview

This document is the implementation plan for vizli, a Python CLI that
renders visualization templates into output files (SVG, PNG, HTML, PDF).

vizli is defined by three specs:
- **TEMPLATE_SPEC_FINAL.md** — what templates look like and how authors write them
- **OUTPUT_SPEC_FINAL.md** — how the CLI renders templates into output files
- **SIDECAR_SPEC.md** — how templates are documented for agent discovery

Plus one operating manual:
- **VIZLI_README.md** — how an AI agent uses vizli at runtime

The plan follows 6 phases. Each phase produces a working, testable
subset of vizli. No phase depends on a later phase. Each phase's
acceptance criteria must pass before the next phase begins.

---

## Phase 0 — Project Bootstrap

**Goal**: Working Python project with CLI skeleton, tooling, and CI.
No vizli logic — just plumbing.

### 0.1 Project structure

Create the package layout from OUTPUT_SPEC_FINAL §4.2:

```
vizli/
  pyproject.toml
  README.md
  src/vizli/
    __init__.py               # version string
    __main__.py               # python -m vizli entrypoint
    cli.py                    # Typer app with command stubs
    models.py                 # empty, Pydantic models come in Phase 1
    errors.py                 # VizliError base class + hierarchy stubs
    logging.py                # Rich-based logging setup

    parsing/
      __init__.py
    payload/
      __init__.py
    validation/
      __init__.py
    renderers/
      __init__.py
    themes/
      __init__.py
    html/
      __init__.py
    output/
      __init__.py
    testing/
      __init__.py
    convert/
      __init__.py

  tests/
    __init__.py
    conftest.py               # shared fixtures
```

### 0.2 pyproject.toml

Configure with all dependency groups from OUTPUT_SPEC_FINAL §16:

- **Core**: typer, rich, pydantic, jsonschema, pyyaml, jinja2, orjson,
  vl-convert-python, defusedxml
- **Optional groups**: `[browser]`, `[altair]`, `[images]`, `[pdf]`,
  `[parquet]`, `[convert]`, `[watch]`, `[dev]`
- Build system: hatchling or setuptools
- Project metadata: name, version, description, python_requires ≥ 3.10

### 0.3 CLI skeleton

Typer app with all commands stubbed (print "not implemented yet"):

- `vizli render <template>`
- `vizli validate <template>`
- `vizli inspect <template>`
- `vizli golden <template>`
- `vizli doctor`
- `vizli convert <input_file>`
- `vizli serve <template>`
- `vizli render-all <dir>`

### 0.4 `vizli doctor` (real implementation)

The first real command. Checks:
- Python version ≥ 3.10
- Core dependencies importable (typer, pydantic, jinja2, orjson, vl-convert-python, defusedxml)
- Optional dependencies (playwright, altair, cairosvg, weasyprint) — report installed/missing
- Playwright chromium available (if playwright installed)
- Temp directory writable

Output uses Rich formatting: green checkmarks for pass, red X for fail.

### 0.5 Tooling

- `ruff` config in pyproject.toml (linting + formatting)
- `mypy` config (strict mode)
- `pytest` config with `tests/` discovery

### 0.6 CI

GitHub Actions workflow:
- Trigger on push
- Python 3.12
- Install `pip install -e ".[dev]"`
- Run `ruff check`, `mypy src/`, `pytest`

### 0.7 Git

Initialize repo, create `.gitignore` (Python, IDE, OS files), initial commit.

### Acceptance criteria

- [ ] `pip install -e ".[dev]"` succeeds
- [ ] `vizli --help` prints all command names
- [ ] `vizli doctor` reports dependency status correctly
- [ ] `vizli render foo` prints "not implemented yet" (no crash)
- [ ] `ruff check src/` passes
- [ ] `mypy src/` passes
- [ ] `pytest` runs (0 tests, 0 failures)
- [ ] CI passes

---

## Phase 1 — Foundation

**Goal**: Parse single-file templates, validate payloads, render
Vega-Lite and SVG templates to SVG and PNG. The `render`, `validate`,
and `inspect` commands work for these two engines.

### 1.1 Pydantic models (`models.py`)

Define the core data structures from OUTPUT_SPEC_FINAL §5:

- `TemplateManifest` — parsed from YAML frontmatter. Fields: spec_version,
  template_id, type, title, description, tags, outputs, canvas (width,
  height, background, scale), schema (params, datasets), sample, runtime,
  template_version, interactive, author, category
- `CanonicalPayload` — params dict, datasets dict, meta dict
- `RenderRequest` — template_root, manifest, payload, format, output_path,
  theme, trace
- `RenderResult` — format, output_path, width, height, engine, duration_ms,
  warnings, diagnostics
- `ParamSchema` — type, required, default, description, enum, min, max,
  examples
- `DatasetSchema` — required, description, columns

### 1.2 Error hierarchy (`errors.py`)

Full error class tree from OUTPUT_SPEC_FINAL §5.5:

```python
class VizliError(Exception): ...
class ManifestError(VizliError): ...
class SchemaError(VizliError): ...
class PayloadError(VizliError): ...
class TemplateCompileError(VizliError): ...
class RenderError(VizliError): ...
class ExtractionError(VizliError): ...
class TimeoutError(VizliError): ...
class PathSafetyError(VizliError): ...
class OutputWriteError(VizliError): ...
class DependencyError(VizliError): ...
```

Each error class carries structured context (template path, failing
field, reason) for Rich-formatted output.

### 1.3 Single-file parser (`parsing/single_file.py`)

Parse `.vizli.*` files:
1. Split on `---` fences
2. Parse YAML frontmatter into `TemplateManifest`
3. Return (manifest, body_string)

Handle edge cases: missing fences, malformed YAML, encoding issues.

### 1.4 Payload resolution (`payload/`)

- `loader.py` — Load data files: CSV → list of dicts, JSON → parsed,
  TSV → list of dicts. Use orjson for JSON.
- `resolver.py` — Merge payload sources in priority order
  (OUTPUT_SPEC_FINAL §3.1): sample → payload file → dataset flags →
  param flags → params-file → canvas overrides
- `normalizer.py` — Type coercion (string "42" → int 42 for int params),
  NaN → null conversion

### 1.5 Validation (`validation/`)

- `manifest.py` — Pydantic model validation of the parsed frontmatter.
  Unknown keys produce warnings. Missing required fields produce errors.
- `payload_schema.py` — Validate resolved payload against the template's
  schema block. Check required params present, types match, enum values
  valid, ranges respected, required datasets present.
- `template_compile.py` — Parse-without-rendering check: Jinja2 syntax
  valid, JSON parseable (for vega-lite), XML parseable via defusedxml
  (for svg).
- `path_safety.py` — Reject paths with `..`, absolute paths, symlink
  escapes. Raise `PathSafetyError`.

### 1.6 Theme system (`themes/`)

- `base.py` — Theme dataclass with all fields from OUTPUT_SPEC_FINAL §9.1.
  Methods: `to_vega_config()`, `to_css_vars()`, `to_svg_data_theme()`.
- `light.py` — Light theme instance
- `dark.py` — Dark theme instance
- `loader.py` — Load custom themes from `~/.config/vizli/themes/*.yaml`

### 1.7 Renderer registry (`renderers/`)

- `base.py` — `RendererAdapter` protocol from OUTPUT_SPEC_FINAL §6
- `registry.py` — Map engine type string → adapter class. Raise
  `ManifestError` for unknown types.

### 1.8 Vega-Lite renderer (`renderers/vega_lite.py`)

Implementation from OUTPUT_SPEC_FINAL §7.1:
1. Render Jinja2 template with payload as context (strict undefined)
2. Parse JSON output, validate it's an object
3. Inject width, height from canvas
4. Apply theme via `to_vega_config()`
5. SVG: `vlc.vegalite_to_svg()`
6. PNG: `vlc.vegalite_to_png(scale=scale)`

### 1.9 SVG renderer (`renderers/svg_engine.py`)

Implementation from OUTPUT_SPEC_FINAL §7.3:
1. Render Jinja2 template with payload as context
2. Parse with defusedxml, validate root `<svg>` with viewBox
3. Inject `data-theme` attribute
4. Set width/height attributes
5. SVG: return normalized string
6. PNG: `cairosvg.svg2png()` with scale and background

### 1.10 Output writer (`output/`)

- `writer.py` — File naming logic (OUTPUT_SPEC_FINAL §10.1): directory
  → `<stem>.<fmt>`, file path → use as-is, `--all` → all formats to
  `--out-dir`. Create directories as needed.
- `optimizer.py` — Lightweight SVG normalization: remove comments,
  collapse whitespace in `d` attributes, round numerics to 2 decimals,
  remove unused IDs.
- `metadata.py` — Embed provenance: SVG comment, PNG EXIF, HTML meta tag.

### 1.11 Wire up CLI commands

Implement for real:
- `vizli render` — full pipeline: parse → validate → resolve → render → write
- `vizli validate` — parse + validate, no render
- `vizli inspect` — parse + print normalized metadata (template ID,
  engine, outputs, canvas, schema summary)

CLI output formatting per OUTPUT_SPEC_FINAL §15:
- Default: `✓ revenue.vizli.json → revenue.png (800×500, 1.2s)`
- Verbose: full diagnostics block
- Errors: structured context with hints

### 1.12 Tests

- `test_models.py` — Pydantic model validation (valid/invalid manifests)
- `test_parser.py` — Single-file parsing (good files, malformed YAML,
  missing fences)
- `test_resolver.py` — Payload merge order, type coercion
- `test_validation.py` — Schema validation (missing params, wrong types,
  enum violations)
- `test_path_safety.py` — Path traversal rejection
- `test_themes.py` — Theme loading, Vega config conversion
- `test_vegalite.py` — Render fixture templates, verify SVG output is
  valid XML, PNG is non-empty with correct dimensions
- `test_svg.py` — Render fixture templates, verify output
- `test_cli.py` — Integration tests: `vizli render`, `vizli validate`,
  `vizli inspect` with fixture templates

Fixture templates:
```
tests/fixtures/
  vegalite/
    bar_simple.vizli.json
    line_timeseries.vizli.json
  svg/
    diagram_simple.vizli.svg
```

### Acceptance criteria

- [ ] `vizli render bar_simple.vizli.json --format svg` produces valid SVG
- [ ] `vizli render bar_simple.vizli.json --format png` produces PNG with correct dimensions
- [ ] `vizli render bar_simple.vizli.json --theme dark` applies dark theme
- [ ] `vizli render diagram.vizli.svg --format png` produces PNG via cairosvg
- [ ] `vizli validate` catches missing required params
- [ ] `vizli validate` catches malformed YAML
- [ ] `vizli inspect` prints template metadata
- [ ] `--dry-run` validates without rendering
- [ ] `--verbose` prints full diagnostics
- [ ] Error messages include template path, failing field, and hint
- [ ] Payload resolution order is correct (CLI overrides file overrides sample)
- [ ] All unit and integration tests pass
- [ ] CI passes

---

## Phase 2 — Browser Engines

**Goal**: Render HTML and JS templates via Playwright. SVG extraction
from rendered pages. PNG screenshotting. Directory package support.

### 2.1 Directory package parser (`parsing/directory.py`)

Parse template directories:
1. Detect `vizli.template.json` presence
2. Parse manifest JSON
3. Resolve `schema.json`, `sample-payload.json`, entrypoint paths
4. Validate path safety for all referenced files
5. Return (manifest, template_root)

Also: `parsing/detect.py` — format detection logic (directory vs
single-file vs error).

### 2.2 Playwright orchestration (`html/browser.py`)

Shared browser management:
- Launch Chromium (headless)
- Create isolated, ephemeral context per render
- Disable network via route interception (`route.abort()`)
- Set viewport, scale, color_scheme from request
- Capture console errors into warnings list
- Enforce timeout
- One browser instance per command invocation, reused across renders
- Cleanup on exit

### 2.3 Readiness resolution (`html/readiness.py`)

Three-tier readiness from TEMPLATE_SPEC_FINAL §8.1:
1. Custom event (`runtime.ready_event`) → listen on document
2. CSS selector (`runtime.ready_selector`) → poll DOM
3. Window flag → poll `window.__VIZLI_READY__ === true`
4. Timeout → raise `TimeoutError`

### 2.4 SVG extraction (`html/extraction.py`)

Three-tier extraction from TEMPLATE_SPEC_FINAL §8.2:
1. Selector (`runtime.svg_selector`) → query + outerHTML
2. Getter function (`window.__VIZLI_GET_SVG__()`) → call + return
3. Auto-detect → `document.querySelector('svg')` → outerHTML
4. Fail → raise `ExtractionError`

### 2.5 HTML renderer (`renderers/html_engine.py`)

Implementation from OUTPUT_SPEC_FINAL §7.4:
1. Render Jinja2 template with payload
2. Inject dataset `<script>` block with `__VIZLI_PARAMS__`,
   `__VIZLI_DATA__`, `__VIZLI_DATASETS__`, `__VIZLI_META__`
3. Write to temp file
4. Load in Playwright (`file://` protocol)
5. Wait for readiness
6. Capture console errors
7. Outputs: HTML (write file), PNG (screenshot), SVG (extraction),
   PDF (print-to-PDF)

### 2.6 JS engine harness (`html/harness.py`)

Generate the HTML wrapper from OUTPUT_SPEC_FINAL §7.5:
- Container div sized to canvas dimensions
- D3 v7 loaded from local bundle (vizli ships a copy)
- Payload injected as script variables
- User's JS module inlined
- `render()` call

### 2.7 JS renderer (`renderers/js_engine.py`)

1. Generate harness HTML via `harness.py`
2. Load in Playwright (same browser rules as HTML engine)
3. Wait for readiness
4. Extract SVG from DOM
5. Screenshot for PNG

### 2.8 Bundle D3 v7

Include `d3.v7.min.js` as a package data file in `src/vizli/assets/`.
The JS harness references it via `file://` path. No CDN.

### 2.9 `vizli inspect` for directory packages

Extend inspect to show directory package metadata: entrypoint path,
schema file, sample payload file, asset directory, golden paths.

### 2.10 Tests

- `test_html_engine.py` — Render HTML fixture, verify PNG dimensions,
  no console errors, readiness fires
- `test_js_engine.py` — Render JS fixture, verify SVG extraction
  produces valid SVG, PNG has correct dimensions
- `test_readiness.py` — Unit tests for each readiness tier
- `test_extraction.py` — Unit tests for SVG extraction tiers
- `test_directory_parser.py` — Parse valid/invalid directory packages
- `test_browser.py` — Network isolation (attempted requests logged),
  timeout enforcement

Fixture templates:
```
tests/fixtures/
  html/
    calculator_simple.vizli.html
    expected/
  js/
    bar_d3.vizli.js
    expected/
  directory_pkg/
    vizli.template.json
    schema.json
    sample-payload.json
    entry/template.html.j2
```

### Acceptance criteria

- [ ] `vizli render calculator.vizli.html --format html` produces working HTML file
- [ ] `vizli render calculator.vizli.html --format png` produces screenshot PNG
- [ ] `vizli render bar_d3.vizli.js --format svg` extracts valid SVG from DOM
- [ ] `vizli render bar_d3.vizli.js --format png` produces screenshot PNG
- [ ] Network requests are blocked and logged as warnings
- [ ] Console errors are captured and surfaced
- [ ] Readiness timeout produces clear `TimeoutError`
- [ ] Directory packages parse and render correctly
- [ ] D3 v7 loads from local bundle (no network)
- [ ] All tests pass, CI passes

---

## Phase 3 — Testing Infrastructure and Altair

**Goal**: Golden testing framework, the `golden` command, Altair engine
with constrained execution, and the parametrized fixture suite.

### 3.1 Golden testing framework (`testing/`)

- `diff.py`:
  - SVG normalization: strip comments, normalize whitespace, sort
    attributes, remove generated IDs and timestamps
  - SVG structural comparison: compare normalized strings
  - PNG pixel-level comparison: load both images, compute per-pixel
    difference, compare against tolerance threshold
    (`visual_diff_tolerance` from manifest, default 0.0)
  - HTML DOM comparison on key selectors (not raw bytes)

- `golden.py`:
  - `golden_check(template_path)` — render sample payload, compare
    against `golden/` directory, return pass/fail with diff details
  - `golden_update(template_path)` — render sample payload, overwrite
    `golden/` directory

### 3.2 `vizli golden` command

```bash
vizli golden <template> --check       # compare
vizli golden <template> --update      # overwrite
vizli golden <dir> --check            # batch check
```

Output:
- Pass: `✓ revenue.vizli.json golden check passed`
- Fail: `✗ revenue.vizli.json golden check FAILED: SVG diff in 3 elements`
  with details of what changed

### 3.3 Altair renderer (`renderers/altair_engine.py`)

Implementation from OUTPUT_SPEC_FINAL §7.2:

1. Load module via `importlib.util.spec_from_file_location`
2. Install import guard (ALLOWED_IMPORTS / BLOCKED_IMPORTS lists)
3. Call `module.build_chart(params, datasets, meta)`
4. Validate return is `alt.TopLevelMixin`
5. Apply width, height via `.properties()`
6. Convert to Vega-Lite spec via `.to_dict()`
7. Delegate to Vega-Lite renderer for SVG/PNG/HTML output

Security: no `exec()`, constrained imports, logged warnings for
suspicious activity.

### 3.4 Engine contract validation (`validation/engine_contract.py`)

Engine-specific validation rules from OUTPUT_SPEC_FINAL §8.4:
- `html` with `svg` output → must declare extraction behavior
- `svg` → must have root `<svg>` with viewBox
- `altair` → module must export `build_chart`
- `vega-lite` → body must parse to JSON object
- `html`/`js` → readiness config must be present
- Declared outputs must be subset of engine's `supported_outputs`

### 3.5 Parametrized fixture suite

Build a library of fixture templates covering all 5 engine types:

```
tests/fixtures/
  vegalite/
    bar_simple.vizli.json
    line_timeseries.vizli.json
    scatter_colored.vizli.json
  altair/
    bar_altair.vizli.py
    layered_chart.vizli.py
  svg/
    diagram_simple.vizli.svg
    themed_boxes.vizli.svg
  html/
    calculator_simple.vizli.html
    stepper_basic.vizli.html
  js/
    bar_d3.vizli.js
    scatter_d3.vizli.js
```

Each fixture has a sample payload and expected outputs in `expected/`.
Parametrized pytest runs every fixture through the render contract.

### 3.6 Tests

- `test_altair_engine.py` — Render Altair fixtures, verify output,
  test import guards (blocked imports raise errors)
- `test_golden.py` — Golden check/update workflow, SVG normalization,
  PNG diff with tolerance
- `test_engine_contract.py` — Validation rules for each engine type
- `test_cross_engine.py` — Parametrized tests across all engines:
  render → verify output exists, valid structure, correct dimensions

### Acceptance criteria

- [ ] `vizli render chart.vizli.py --format svg` renders Altair template
- [ ] Altair import guard blocks `os`, `subprocess`, `socket` etc.
- [ ] `vizli golden chart.vizli.json --check` passes for unmodified template
- [ ] `vizli golden chart.vizli.json --check` fails after modifying template
- [ ] `vizli golden chart.vizli.json --update` regenerates golden files
- [ ] SVG normalization ignores whitespace and attribute ordering
- [ ] PNG comparison respects `visual_diff_tolerance`
- [ ] Engine contract validation catches misconfigurations
- [ ] Parametrized fixture suite covers all 5 engine types
- [ ] All tests pass, CI passes

---

## Phase 4 — Workflow Tools

**Goal**: Convert command (LLM integration), serve with live-reload,
render-all batch, PDF output, custom themes, watch mode, SVG
optimization, offline HTML bundling.

### 4.1 PDF output (`output/pdf.py`)

Two paths:
- SVG → PDF via weasyprint
- HTML → PDF via Playwright print-to-PDF

Wire into each renderer's `supported_outputs`. The template doesn't
know about PDF — it's a secondary conversion from SVG or HTML.

### 4.2 `vizli convert` (`convert/llm_converter.py`)

Implementation from OUTPUT_SPEC_FINAL §11:
- Anthropic SDK integration (`anthropic` package)
- API key from `ANTHROPIC_API_KEY` env or `~/.config/vizli/config.yaml`
- Sends raw file + TEMPLATE_SPEC_FINAL + SIDECAR_SPEC as context
- Parses response into template file + sidecar file
- Writes both to output directory
- Validates the produced template with `vizli validate`

### 4.3 `vizli serve` (`serve.py`)

Local development server for HTML templates:
- HTTP server on configurable port (default 8400)
- Serves the rendered HTML template
- WebSocket for live-reload notifications
- Watches template + payload + data files for changes
- On change: re-render → push reload via WebSocket

### 4.4 `vizli render-all`

Batch render all templates in a directory:
- Discover templates (single-file and directory packages)
- For each: find matching payload (`<stem>.payload.json`) or use sample
- Render requested format(s) to output directory
- Report results summary (passed/failed/skipped)

### 4.5 Watch mode (`watch.py`)

`--watch` flag for `vizli render`:
- Use `watchdog` to monitor template file, data files, payload files
- On change: re-render, write output
- Combined with `--preview`: open output in default viewer

### 4.6 Custom theme loading (`themes/loader.py`)

Load YAML theme files from `~/.config/vizli/themes/`:
- Parse YAML into Theme dataclass
- Register by name
- Available via `--theme <name>`

### 4.7 Offline HTML bundling (`output/bundler.py`)

For HTML output: inline all CSS and JS assets into a single file.
For Vega-Lite/Altair HTML output: bundle the Vega runtime locally.
No CDN references in any output file.

Modes:
- `single-file` — inline everything (default)
- `bundle-dir` — HTML + sibling asset files
- `auto` — choose by size

### 4.8 SVG optimization (`output/optimizer.py`)

Extend the lightweight optimizer with optional `--optimize` flag that
shells out to `svgo` if installed. Also `--png-optimize` for `oxipng`.

### 4.9 Tests

- `test_pdf.py` — SVG → PDF, HTML → PDF
- `test_convert.py` — Mock LLM API, verify template + sidecar produced,
  validate output
- `test_serve.py` — Start server, verify HTTP response, test WebSocket
  reload
- `test_render_all.py` — Batch render fixture directory
- `test_watch.py` — File change triggers re-render
- `test_bundler.py` — HTML bundling inlines assets, no external refs

### Acceptance criteria

- [ ] `vizli render chart.vizli.json --format pdf` produces valid PDF
- [ ] `vizli convert raw_chart.json` produces valid template + sidecar
- [ ] `vizli serve explainer.vizli.html` starts server, live-reload works
- [ ] `vizli render-all templates/ --format png` batch renders all templates
- [ ] `--watch` re-renders on file change
- [ ] `--theme brand` loads custom theme from `~/.config/vizli/themes/`
- [ ] HTML output is fully offline (no external URLs in source)
- [ ] `--optimize` produces smaller SVG
- [ ] All tests pass, CI passes

---

## Phase 5 — Template Library and Skill Integration

**Goal**: Build an initial library of production templates with
sidecars, integrate vizli as a Claude Code skill, and validate the
full agent workflow end-to-end.

### 5.1 Template library

Create 10-15 templates covering the most common visualization needs.
Each template includes a sidecar file per SIDECAR_SPEC.md.

**Charts** (vega-lite):
- `bar_simple` — single-category bar chart
- `bar_grouped` — grouped bars with color encoding
- `bar_stacked` — stacked bars (part-of-whole)
- `line_timeseries` — single time series
- `line_multiseries` — multiple series with legend
- `scatter_simple` — basic scatter plot
- `scatter_colored` — scatter with categorical color
- `heatmap` — matrix heatmap with color scale

**Explainers** (html):
- `ab_test_power` — A/B test power calculator with sliders
- `cuped_variance` — CUPED variance reduction explainer

**Diagrams** (svg):
- `pipeline_flow` — data pipeline architecture
- `system_architecture` — service architecture diagram

**D3** (js):
- `force_graph` — force-directed network graph
- `treemap` — hierarchical treemap

Each template must:
- Pass `vizli validate`
- Pass `vizli golden --check`
- Have a complete sidecar per SIDECAR_SPEC
- Render in both light and dark themes
- Meet the acceptance checklist in TEMPLATE_SPEC_FINAL §16

### 5.2 Vizli rendering skill

Create a Claude Code skill that wraps the agent workflow from
VIZLI_README.md:

1. Receive user request for a visualization
2. Search sidecars to find matching template
3. Inspect user data (column names, types, shape)
4. Resolve params from user request + data inspection
5. Call `vizli render` with correct arguments
6. Verify output, deliver to user

The skill file follows the structure of the existing `explainer` skill
but delegates rendering to vizli instead of generating HTML directly.

### 5.3 Explainer skill migration

Evaluate migrating the existing `explainer` skill to produce vizli
templates instead of raw HTML. The explainer skill currently generates
standalone HTML files — these could become `html`-type vizli templates
with sidecars, gaining golden testing and theme consistency.

This is an evaluation step, not a commitment. If the migration adds
complexity without clear value, keep the explainer skill independent.

### 5.4 Template registry

Create `vizli-templates.yaml` at the library root for discovery and
batch operations (TEMPLATE_SPEC_FINAL §14.3).

### 5.5 End-to-end validation

Test the full agent workflow:
1. Agent receives "chart my revenue data"
2. Agent searches sidecars, finds `bar_grouped`
3. Agent inspects CSV, maps columns to params
4. Agent calls `vizli render` with correct flags
5. Output file is valid, dimensions correct, theme applied

This is a manual validation step, not an automated test. Run it
against multiple request types (chart, explainer, diagram) to verify
the skill and templates work together.

### 5.6 Documentation

- Update `README.md` with installation, quickstart, and examples
- Ensure all specs are consistent with the built implementation
- Remove v0 spec files (`TEMPLATE_SPEC.md`, `VIZLI_OUTPUT_SPEC.md`,
  `v1_TEMPLATE_SPEC.md`, `v1_VIZLI_OUTPUT_SPEC.md`) once the FINAL
  versions are confirmed as authoritative

### Acceptance criteria

- [ ] 10+ templates pass `vizli validate` and `vizli golden --check`
- [ ] Every template has a complete sidecar
- [ ] Templates render correctly in both light and dark themes
- [ ] Template registry is valid and lists all templates
- [ ] Vizli rendering skill invokes `vizli render` successfully
- [ ] End-to-end agent workflow produces correct output for chart, explainer, and diagram requests
- [ ] Documentation is complete and accurate

---

## Cross-cutting concerns

### Exit codes

All phases use the exit codes from OUTPUT_SPEC_FINAL §18:

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Template, validation, or render error |
| 2 | CLI usage error |
| 3 | Missing dependency |
| 4 | Timeout |

### Logging

All phases use Rich-formatted output per OUTPUT_SPEC_FINAL §15:
- Default: one-line success/failure
- `--verbose`: full diagnostics
- `--quiet`: errors only
- `--trace`: debug artifacts (browser traces, intermediate files)

### Security

All phases enforce the security rules from OUTPUT_SPEC_FINAL §12:
- Network isolation for browser rendering
- Path safety validation for all file access
- XML safety via defusedxml for SVG
- Constrained imports for Altair
- No `exec()` anywhere

### Performance

Per OUTPUT_SPEC_FINAL §17: correctness first, predictability second,
speed third. Acceptable caching: manifest cache within process, Jinja2
compiled templates, browser reuse within command. No daemon mode, no
persistent browser pools, no cross-command caching.

---

## Dependency graph

```
Phase 0 ──→ Phase 1 ──→ Phase 2 ──→ Phase 3 ──→ Phase 4
                                        │
                                        └──→ Phase 5
```

Phase 5 depends on Phase 3 (needs golden testing and all engines).
Phase 4 can run in parallel with Phase 5 once Phase 3 is complete —
workflow tools are independent of the template library.

---

## Summary

| Phase | Delivers | Key risk |
|---|---|---|
| 0 | Working project, CI, `doctor` | None — pure setup |
| 1 | Vega-Lite + SVG rendering, validation, inspect | vl-convert API surface |
| 2 | HTML + JS rendering via Playwright | Browser flakiness in CI |
| 3 | Altair engine, golden testing | Import guard completeness |
| 4 | Convert, serve, render-all, PDF, watch | LLM API integration |
| 5 | Template library, skill, end-to-end | Template quality at scale |

The first useful tool emerges at the end of Phase 1. By Phase 3, vizli
is a complete rendering pipeline for all 5 engine types with regression
testing. Phases 4 and 5 add workflow convenience and the content library
that makes vizli valuable to agents.
