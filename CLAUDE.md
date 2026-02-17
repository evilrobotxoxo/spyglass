# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Spyglass (`spyglass-neuro` on PyPI) is a neuroscience data analysis framework built on **DataJoint** (reproducible pipelines) and **NWB** (Neurodata Without Borders data format). It provides standardized storage, reproducible analysis, and sharing of neuroscience data. Python 3.10-3.12.

## Build & Install

```bash
# Install with test dependencies
pip install -e .[test]

# Full install via conda (recommended for users)
conda env create -f environments/environment.yml

# Start local MySQL database
docker compose up -d
```

## Linting & Formatting

Pre-commit hooks handle formatting. Key tools: **Black** (80 char lines), **Ruff**, **autoflake**, **codespell**.

```bash
pre-commit run --all-files    # Run all hooks
black src/                    # Format only
ruff check src/               # Lint only
```

Ruff ignores: F401 (unused imports needed for DataJoint foreign keys), E402 (lazy loading), E501 (long table definitions).

## Testing

Tests require a MySQL database (Docker or remote). Test data is downloaded from UCSF Box.

```bash
# Run all tests (needs MySQL + test data)
pytest --no-docker --no-dlc tests/

# Fast tests only (skip slow/very_slow)
pytest --no-docker --no-dlc -m 'not slow and not very_slow' tests/

# Run a single test file
pytest --no-docker --no-dlc tests/path/to/test_file.py

# Run a single test
pytest --no-docker --no-dlc tests/path/to/test_file.py::test_name

# Useful pytest options already in pyproject.toml:
#   -s (no capture), --doctest-modules, --cov=spyglass
```

Custom pytest flags: `--no-teardown` (keep DB after tests), `--quiet-spy` (suppress spyglass logging), `--no-dlc` (skip DLC tests), `--no-docker` (use existing MySQL).

Test markers: `fast` (<1s), `medium` (1-10s), `slow` (10-60s), `very_slow` (>60s), `unit`, `integration`, `serial`.

## Architecture

### Package Structure (`src/spyglass/`)

Each subdirectory is an analysis pipeline. Pipelines may have versioned subdirs (`v0/`, `v1/`):

- **common/** — Shared tables across all pipelines (Session, Subject, Lab, IntervalList, Nwbfile, etc.)
- **spikesorting/** — Spike sorting (v0, v1)
- **lfp/** — Local field potential analysis (v1)
- **position/** — Position tracking (DeepLabCut integration)
- **decoding/** — Neural decoding (v0, v1)
- **linearization/** — Track linearization (v0, v1)
- **ripple/** — Ripple detection
- **mua/** — Multi-unit activity
- **behavior/** — Behavioral analysis (MoSeq)
- **sharing/** — Kachery data sharing integration
- **utils/** — Mixins, merge tables, helpers

### DataJoint Table Hierarchy

Tables follow a standard pipeline pattern: **Parameters** (manual) → **Selection** (manual) → **Computed Data** (dj.Computed with `make()` method).

Key table base classes in `utils/dj_mixin.py`:
- **SpyglassMixin** — All tables inherit this. Composes: CautiousDeleteMixin, ExportMixin, FetchMixin, HelperMixin, PopulateMixin, RestrictByMixin
- **Merge** (`utils/dj_merge_tables.py`) — Special table type that unifies output from different pipeline versions (v0, v1) for downstream analysis. Uses `merge_id` (UUID) as primary key

### Configuration

**SpyglassConfig** (`settings.py`) loads from: CLI args → DataJoint config → env vars → defaults. Directory layout defined in `directory_schema.json` (single source of truth). Key env var: `SPYGLASS_BASE_DIR`.

### Schema Naming

Each module registers a DataJoint schema (= MySQL database): `common_lab`, `common_session`, `common_ephys`, `spikesorting_v1_recording`, etc. Schema is declared once per module: `schema = dj.schema("schema_name")`.

## PR Conventions

- Reference issues with `Fixes #NNN`
- Include `alter()` snippets for any table schema changes
- Run position tests if touching position code
- Update `CHANGELOG.md` with PR number
- Update `CITATION.cff` if release-worthy

## Release Process

1. Update version in `CITATION.cff`
2. Merge PR, then tag: `git tag {version} && git push origin {version}`
3. Tag push triggers docs rebuild and PyPI publish
