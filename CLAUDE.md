# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GWASTutorial is an educational resource providing hands-on training in Genome-Wide Association Studies (GWAS) and complex trait genomics. The project is a documentation site built with MkDocs Material, containing tutorials, exercises, and Jupyter notebooks covering the full GWAS analysis pipeline from data QC through post-GWAS analysis.

## Repository Structure

The repository follows a numbered module system:

- **Numbered directories** (e.g., `01_Dataset`, `04_Data_QC`, `06_Association_tests`) contain individual tutorial modules
- Each module typically contains:
  - `README.md` with tutorial content and module metadata (YAML frontmatter)
  - Shell scripts (`.sh`) demonstrating command-line workflows
  - Jupyter notebooks (`.ipynb`) for interactive analysis and visualization
  - Python scripts for data processing or visualization
- **`docs/`** contains the built documentation site (generated from module READMEs)
- **`.development/`** contains maintainer tooling (not part of published tutorial):
  - `scripts/` - Lint, crosslink checking, knowledge graph extraction
  - `dictionary/` - GWAS Dictionary integration for term expansion
  - `kg/` - Knowledge graph schema and pipeline metadata
- **Root-level scripts**:
  - `deploy.sh` - Builds docs and serves site locally
  - `convert_notebook_to_md.py` - Converts Jupyter notebooks to markdown
  - `generate_wordcloud.py` - Generates word cloud visualization from tutorial content

## Development Commands

### Documentation Site

The site is built using MkDocs Material with the Zensical theme.

**Primary workflow (recommended):**
```bash
bash deploy.sh
```
This script orchestrates the full build pipeline:
1. Copies module READMEs from numbered directories to `docs/`
2. Copies selected Jupyter notebooks to `docs/`
3. Runs validation: `extract_kg.py`, `lint_module.py`, `check_crosslinks.py`
4. Serves the site at `http://127.0.0.1:9999`
5. **Stops on validation errors** (exit code 1) - fix issues before the site serves

**Manual build (for faster iteration without validation):**
```bash
# Install dependencies
pip install -r requirements-docs.txt

# Serve site without running validators
zensical serve -a 127.0.0.1:9999

# Deploy to GitHub Pages
mkdocs gh-deploy
```

**Python environment:**
```bash
# Full development environment (includes mkdocs, validators, notebook tools)
pip install -e .
pip install -r requirements-docs.txt

# For validation scripts only
pip install pyyaml  # optional, for term_aliases.yaml support
```

### Validation and Linting

**Run all validation checks:**
```bash
python3 .development/scripts/lint_module.py      # Validates frontmatter completeness
python3 .development/scripts/extract_kg.py       # Extracts knowledge graph, verifies structure
python3 .development/scripts/check_crosslinks.py # Validates prerequisite links
```

**What each validator checks:**

- **`lint_module.py`** - Enforces module frontmatter schema:
  - Required fields: `module_id`, `type`, `title`, `prerequisites`, `produces`, `primary_script`, `tools`, `concepts`
  - Valid types: `hands_on`, `concept`, `resource`, `reference`
  - `module_id` matches directory name
  - Lists are properly formatted (not empty strings)

- **`extract_kg.py`** - Builds knowledge graph from metadata:
  - Parses YAML frontmatter from all module READMEs
  - Extracts tool usage, concepts, data flow (prerequisites → produces)
  - Writes JSON to `.development/kg/modules/*.json` and `pipeline_core.json`
  - Validates against `.development/kg/schema.json`

- **`check_crosslinks.py`** - Validates module dependencies:
  - Ensures all `prerequisites` reference existing `module_id`s
  - Checks for circular dependencies
  - Verifies files listed in `produces` exist or are documented

Run individually during development; `deploy.sh` runs all three before serving.

### Convert Notebooks to Markdown

```bash
python3 convert_notebook_to_md.py path/to/notebook.ipynb
```

This extracts outputs, saves images, and converts HTML tables to markdown.

### Generate Word Cloud

```bash
python3 generate_wordcloud.py
```

Creates a word cloud visualization from GWAS-related terms in all tutorial markdown files. Output: `images/wordcloud.png` (used in README).

### GWAS Dictionary Integration

Link tutorial terms to the [GWAS Dictionary](https://cloufield.github.io/GWASDictionary/):

```bash
# Expand "## Key terms" sections in docs/*.md with dictionary definitions
python3 .development/dictionary/expand_key_terms.py

# Target specific file
python3 .development/dictionary/expand_key_terms.py docs/05_PCA.md

# Offline mode (use cached dictionary only, no network)
python3 .development/dictionary/expand_key_terms.py --offline

# Preview changes without writing
python3 .development/dictionary/expand_key_terms.py --dry-run
```

**How it works:**
1. Add a `## Key terms` section to a module README with comma-separated terms
2. Run `expand_key_terms.py` - it fetches definitions from GWAS Dictionary and rewrites the section as bulleted links
3. If a term isn't found, add an alias in `.development/dictionary/term_aliases.yaml`

**Currently disabled in `deploy.sh`** (line commented out) - uncomment to auto-expand terms on every build.

## Module Metadata

Each module's `README.md` begins with YAML frontmatter specifying:

```yaml
---
module_id: 06_Association_tests
type: hands_on
title: Association test
prerequisites: [04_Data_QC, 05_PCA, 01_Dataset]
produces: [1kgeas.B1.glm.firth]
primary_script: run_association_test.sh
tools: [plink2]
concepts:
  - genome-wide association study
  - logistic regression
---
```

**Required fields (enforced by `lint_module.py`):**
- `module_id` - Unique identifier, **must match directory name exactly**
- `type` - Module classification: `hands_on`, `concept`, `resource`, or `reference`
- `title` - Human-readable module title (displayed in navigation)
- `prerequisites` - List of prerequisite `module_id`s (use `[]` for none)
- `produces` - Key output files/data generated by this module (can be empty list)
- `primary_script` - Main executable script (e.g., `run_pca.sh`; use empty string `""` if none)
- `tools` - Bioinformatics software used (e.g., `[plink2, gcta]`)
- `concepts` - Key concepts covered (used for knowledge graph and search)

**Module types:**
- `hands_on` - Executable tutorial with scripts and exercises (e.g., 04_Data_QC, 06_Association_tests)
- `concept` - Theoretical content explaining principles (e.g., 13_heritability, 19_ld)
- `resource` - Reference materials or datasets (e.g., 01_Dataset, 40_1000_genome_project)
- `reference` - Documentation or guides (e.g., 90_Recommended_Reading)

**Dependency rules:**
- `prerequisites` must reference existing `module_id`s (validated by `check_crosslinks.py`)
- Avoid circular dependencies (module A → B → A)
- Order matters for the tutorial flow - earlier modules should have lower numbers

## Architecture

**Content Pipeline:**
1. **Source of truth**: Module directories (`NN_module_name/README.md`, scripts, notebooks)
2. **Build step**: `deploy.sh` copies to `docs/` and validates
3. **Static site generation**: Zensical/MkDocs builds from `docs/` + `mkdocs.yml`
4. **Deployment**: GitHub Actions builds and deploys to GitHub Pages

**Key Design Patterns:**

- **Separation of source and build artifacts**
  - **Never edit `docs/` directly** - all changes go in numbered module directories
  - `docs/` is git-tracked but fully generated (allows GitHub Pages to build from it)
  - Module `README.md` → `docs/NN_module_name.md` (filename transformation)
  
- **Metadata-driven validation**
  - YAML frontmatter enables automated prerequisite checking, dependency graphs, and tool tracking
  - Knowledge graph extracted from frontmatter powers the pipeline visualization
  - Validation runs **before** site build to catch errors early
  
- **Jupyter notebook integration**
  - Notebooks are first-class content via `mkdocs-jupyter` plugin
  - Selected notebooks copied to `docs/` in `deploy.sh` (not all notebooks are published)
  - Notebooks can contain executable code, outputs, and visualizations
  
- **Progressive disclosure**
  - Tutorial complexity increases: introductory (02_Linux_basics) → core GWAS (04-06) → advanced (21_twas, 22_bias)
  - Numbered prefixes (`01_`, `02_`, ...) indicate recommended sequence
  - Prerequisites enforce learning path

**Why this architecture?**
- **Maintainability**: Source files live with their scripts/data; site generation is reproducible
- **Validation**: Catch broken links and missing prerequisites before deployment
- **Flexibility**: Modules can be developed independently; frontmatter wires them together
- **Searchability**: Knowledge graph enables finding modules by tool, concept, or data flow

## Configuration Files

- **`mkdocs.yml`** - Site configuration, navigation structure, theme settings, plugins
  - `nav:` section defines site navigation (must be updated when adding modules)
  - `plugins:` includes `mkdocs-jupyter` for notebook rendering
  - Theme: Material with custom overrides in `overrides/`
  
- **`pyproject.toml`** - Python package metadata and dependencies
  - Installs as `gwastutorial` package via `pip install -e .`
  - Core dependencies: mkdocs, mkdocs-material, mkdocs-jupyter, nbformat, nbconvert
  
- **`requirements-docs.txt`** - Pinned Zensical version for reproducible builds
  - **Critical**: Zensical version must match between local dev and GitHub Actions
  - Currently pinned to `zensical==0.0.31`
  
- **`.devcontainer/`** - VS Code development container configuration
  - Sets up containerized development environment with all tools pre-installed

- **`.development/`** - Maintainer tooling (not part of published tutorial)
  - `scripts/` - Validation and knowledge graph extraction
  - `dictionary/` - GWAS Dictionary integration
  - `kg/` - Knowledge graph schema and generated JSON

## Common Patterns

**Adding a new module:**
1. Create directory with format `NN_module_name/` (choose number to indicate sequence)
2. Add `README.md` with complete YAML frontmatter (all required fields)
3. Add module to `mkdocs.yml` navigation section (maintain logical grouping)
4. Add copy command to `deploy.sh` if needed: `cp NN_module_name/README.md docs/NN_module_name.md`
5. If the module includes notebooks, add: `cp NN_module_name/notebook.ipynb docs/notebook.ipynb`
6. Run `bash deploy.sh` - validation will catch missing prerequisites or invalid metadata
7. Check the built site at `http://127.0.0.1:9999` to verify rendering

**Working with notebooks:**
- Place notebooks in their module directory (e.g., `06_Association_tests/Visualization.ipynb`)
- Only copy to `docs/` if they should appear in the published site (see `deploy.sh` for examples)
- Use `convert_notebook_to_md.py` for standalone markdown versions (extracts outputs, converts tables)
- Notebooks are rendered via `mkdocs-jupyter` plugin (preserves interactive outputs)

**Module dependencies:**
- Declare all prerequisite modules in `prerequisites: [...]` frontmatter
- `check_crosslinks.py` validates that all referenced module_ids exist
- If a module produces data that another consumes, list it in `produces` and reference in `prerequisites`
- Visual dependency graph generated by `extract_kg.py` → `.development/kg/pipeline_core.json`

**Modifying existing content:**
- Edit the source `NN_module_name/README.md`, **never** `docs/NN_module_name.md`
- Run `bash deploy.sh` to regenerate `docs/` and validate
- For quick iteration, use `zensical serve -a 127.0.0.1:9999` (skips validation)

**Tool and concept tracking:**
- Add new tools to `tools: []` frontmatter when introducing them
- Tools are aggregated across modules for discovery (see `.development/kg/tools.yaml`)
- Concepts should use consistent naming (check existing modules for patterns)
- Concepts power search and enable "find all modules teaching X"

## Working with GWAS Tools

This tutorial uses many bioinformatics tools. When documenting tool usage:

**Script organization:**
- Primary scripts (e.g., `run_association_test.sh`) should be executable and self-contained
- Add `# Step: Description` comments for knowledge graph extraction
- Include `!!! note` sections in README listing required data and software

**Tool naming conventions:**
- Use canonical names in frontmatter: `plink2` (not `plink`), `gcta` (not `GCTA64`)
- Track tool versions in module README when version-specific behavior matters
- Reference `.development/kg/tools.yaml` for standard tool names

**Common tools in this tutorial:**
- **PLINK/PLINK2** - Genotype QC, association testing, LD pruning
- **GCTA** - Heritability estimation (GREML), PCA, GRM construction
- **REGENIE** - Whole genome regression (fast LMM for biobank-scale data)
- **SAIGE** - Saddlepoint approximation for unbalanced case/control
- **LDSC** - LD score regression (heritability, genetic correlation)
- **ANNOVAR/VEP** - Variant annotation
- **SuSIE** - Fine-mapping
- **Python packages** - gwaslab (visualization), pandas, matplotlib

**When adding a new tool to the tutorial:**
1. Add installation instructions to the module README
2. List in `tools:` frontmatter
3. Provide example command with explanation
4. Link to official tool documentation
5. Add to `.development/kg/tools.yaml` if introducing tool for first time

## Testing and Validation

The repository uses validation scripts rather than traditional unit tests:

**Validation scripts (run automatically by `deploy.sh`):**
- `lint_module.py` - Checks frontmatter completeness and correctness
- `check_crosslinks.py` - Ensures module references and prerequisites are valid
- `extract_kg.py` - Verifies module relationships and generates knowledge graph

**Pre-commit checklist:**
1. Run `bash deploy.sh` to validate all changes
2. Check that the site builds without errors at `http://127.0.0.1:9999`
3. Verify new modules appear in navigation
4. Test any new scripts/notebooks execute correctly
5. Review git diff to ensure no unintended changes to `docs/`

**Common validation errors:**

- **"module_id mismatch"** - Directory name doesn't match `module_id` in frontmatter
  - Fix: Rename directory or update `module_id` to match

- **"Unknown prerequisite: XX_module"** - Referenced module doesn't exist
  - Fix: Check spelling, ensure prerequisite module has valid frontmatter

- **"Missing required field"** - Frontmatter incomplete
  - Fix: Add all required fields (see Module Metadata section)

- **"Circular dependency detected"** - Module A requires B, B requires A
  - Fix: Restructure dependencies to be unidirectional

**Debugging build issues:**
```bash
# Run validators individually to isolate issues
python3 .development/scripts/lint_module.py
python3 .development/scripts/extract_kg.py
python3 .development/scripts/check_crosslinks.py

# Check generated knowledge graph
cat .development/kg/modules/06_Association_tests.json

# Validate mkdocs configuration
mkdocs build --strict  # Fails on any warnings
```
