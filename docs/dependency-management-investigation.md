# Core Dependency Management — Investigation Notes

**Date**: 2026-03-28 (GEO #75)
**Author**: Hulk
**Status**: Investigation Complete

---

## Current State

As of 2026-03-28, the `core` repository **does not have** a `requirements.txt` file or any other Python dependency management file.

### Investigation Results

```bash
# Checked for common dependency files
ls -la /home/node/.openclaw/workspace-hulk/github-repos/core/

# Result: No requirements.txt, pyproject.toml, setup.py, or Pipfile found
```

### Repository Structure

```
core/
├── .git/
├── .github/
├── CHANGELOG.md
├── README.md
├── docs/
│   ├── scorer-migration-phase1.md
│   ├── scorer-migration-plan.md
│   └── [other docs...]
└── media/
```

**Note**: This is a **documentation-only repository** for the CittaVerse core product strategy. It does not contain executable Python code.

---

## Implications for Scorer Migration

The `scorer-migration-phase1.md` plan assumed `core` was a Python package with `requirements.txt`. However, since `core` is a **documentation repository**, the migration approach needs adjustment.

### Revised Understanding

| Original Assumption | Actual State | Implication |
|---------------------|--------------|-------------|
| `core` is a Python package | `core` is docs-only | No `requirements.txt` to update |
| `core` imports `narrative-scorer` | `core` documents architecture | Migration is conceptual, not code-level |
| Phase 1: Add dependency | Phase 1: Document integration pattern | Wrapper layer goes in `pipeline` or separate service repo |

### Correct Integration Target

The actual Python code that uses `narrative-scorer` is likely in:

1. **`pipeline` repository**: Contains the scoring engine and API layer
2. **Future service repository**: Backend service that wraps `narrative-scorer` for production use

---

## Recommended Actions

### 1. Update scorer-migration-phase1.md

Clarify that `core` is documentation-only and the actual integration happens in `pipeline` or a dedicated service repository.

### 2. Investigate pipeline Repository

Check if `pipeline` has `requirements.txt` and is the correct target for `narrative-scorer` dependency.

```bash
ls -la /home/node/.openclaw/workspace-hulk/github-repos/pipeline/
```

### 3. Create Dependency Management Proposal

Document the intended architecture:

```
┌─────────────────────────────────────────────┐
│  narrative-scorer (PyPI library, v0.7.0+)   │
└──────────────────┬──────────────────────────┘
                   │ pip install
                   ▼
┌─────────────────────────────────────────────┐
│  pipeline / scorer-service (Python backend) │
│  - requirements.txt: narrative-scorer>=0.7.0 │
│  - wrapper layer: narrative_scorer_wrapper.py│
│  - API: FastAPI endpoints for scoring        │
└──────────────────┬──────────────────────────┘
                   │ REST API
                   ▼
┌─────────────────────────────────────────────┐
│  core (Documentation)                        │
│  - Documents architecture                    │
│  - References pipeline service API           │
│  - No direct Python dependencies             │
└─────────────────────────────────────────────┘
```

---

## Next Steps

1. ~~**Investigate `pipeline` repository** for existing dependency management~~ ✅ Done (has requirements.txt + pyproject.toml)
2. **Update `scorer-migration-phase1.md`** to reflect correct integration target
3. ~~**Create `pipeline/requirements.txt`** (if not exists) with `narrative-scorer>=0.7.0,<0.8.0`~~ ✅ Done (added 2026-03-28)
4. **Implement wrapper layer** in `pipeline/services/narrative_scorer_wrapper.py`

---

## Investigation Log

| Time (UTC) | Action | Result |
|------------|--------|--------|
| 2026-03-28 06:15 | Checked `core/` for dependency files | None found |
| 2026-03-28 06:15 | Reviewed `scorer-migration-phase1.md` | Assumed Python package |
| 2026-03-28 06:16 | Analyzed repository purpose | Docs-only, not executable code |
| 2026-03-28 06:17 | Documented findings | This file |

---

*GEO #75 — Core Dependency Management Investigation Complete*
