# Core Scorer Migration Plan: Embedded → Library Import

**Date**: 2026-03-27
**Author**: Hulk (GEO #72)
**Status**: Planning Complete — Ready for Execution (Starts 2026-03-31)

---

## Executive Summary

This document outlines the migration plan for integrating `narrative-scorer` into the `core` repository as a proper Python library dependency, replacing the current embedded copy pattern.

**Current State**: Embedded copy (Pattern 3 — Deprecated)
**Target State**: Library import (Pattern 1 — Recommended)
**Timeline**: 4 weeks (2026-03-31 to 2026-04-28)
**Risk Level**: Low (backward-compatible wrapper layer)

---

## Motivation

### Current Problems (Embedded Copy)

1. **Code Duplication**: Same scorer logic exists in both `core/` and `narrative-scorer/`
2. **Sync Drift**: Bug fixes in `narrative-scorer` don't automatically propagate to `core`
3. **Maintenance Burden**: Two codebases to maintain, test, and document
4. **Version Confusion**: Unclear which version is authoritative
5. **Testing Overhead**: Must test both repos independently and together

### Benefits of Library Import

1. **Single Source of Truth**: `narrative-scorer` is the only authoritative implementation
2. **Automatic Updates**: `pip install --upgrade narrative-scorer` gets latest fixes
3. **Clear Versioning**: `core` specifies `narrative-scorer>=0.7.0,<0.8.0`
4. **Reduced Maintenance**: One codebase to maintain
5. **Better Testing**: Test integration, not duplication

---

## Migration Phases

### Phase 1: Preparation (Week 1: 2026-03-31 to 2026-04-07)

**Goal**: Set up infrastructure for library-based integration.

#### Tasks

- [ ] **1.1**: Update `core/requirements.txt`
  ```txt
  # narrative-scorer library (v0.7.0+)
  narrative-scorer>=0.7.0,<0.8.0
  ```

- [ ] **1.2**: Create `core/services/narrative_scorer_wrapper.py`
  ```python
  """Compatibility wrapper for narrative-scorer library.
  
  This wrapper provides a stable API for core services,
  absorbing any breaking changes in narrative-scorer.
  """
  from narrative_scorer import score_narrative as _score_narrative
  from narrative_scorer import LLMFeatureExtractor
  
  def score_narrative(text: str, use_llm: bool = True) -> dict:
      """Score a single narrative (wrapper for library function)."""
      return _score_narrative(text, use_llm=use_llm)
  
  class NarrativeScorerService:
      """Service layer for narrative scoring (used by core APIs)."""
      
      def __init__(self, use_llm: bool = True, api_key: str | None = None):
          self.use_llm = use_llm
          self.extractor = LLMFeatureExtractor(api_key=api_key) if use_llm else None
      
      def score(self, text: str) -> dict:
          """Score a narrative."""
          return score_narrative(text, use_llm=self.use_llm)
  ```

- [ ] **1.3**: Write integration tests for wrapper
  - Test `score_narrative()` function
  - Test `NarrativeScorerService` class
  - Mock LLM API calls (no API key needed for tests)

- [ ] **1.4**: Document breaking changes (if any)
  - Compare embedded API vs library API
  - List any incompatible changes
  - Plan mitigation (wrapper layer should absorb most)

#### Deliverables

- [ ] `core/requirements.txt` updated
- [ ] `core/services/narrative_scorer_wrapper.py` created
- [ ] `core/tests/test_narrative_scorer_wrapper.py` created
- [ ] `core/docs/scorer-migration-notes.md` created

---

### Phase 2: Dual-Run (Week 2: 2026-04-08 to 2026-04-14)

**Goal**: Run both embedded and library-based scorers in parallel to validate correctness.

#### Tasks

- [ ] **2.1**: Deploy wrapper alongside embedded code
  - Do NOT remove embedded code yet
  - Add feature flag: `NARRATIVE_SCORER_USE_LIBRARY` (default: `false`)

- [ ] **2.2**: Route 10% traffic to library-based scorer
  ```python
  # core/services/narrative_scoring_service.py
  import os
  import random
  
  def should_use_library() -> bool:
      """Determine if library-based scorer should be used."""
      if os.getenv("NARRATIVE_SCORER_USE_LIBRARY") == "true":
          return random.random() < 0.1  # 10% traffic
      return False
  
  def score_narrative(text: str) -> dict:
      if should_use_library():
          from .narrative_scorer_wrapper import score_narrative as library_score
          return library_score(text)
      else:
          from .narrative_scorer_embedded import score_narrative as embedded_score
          return embedded_score(text)
  ```

- [ ] **2.3**: Monitor metrics
  - Latency: library vs embedded (should be similar)
  - Accuracy: scores should match (within floating-point tolerance)
  - Error rates: library should not introduce new errors

- [ ] **2.4**: Compare outputs
  - Log both scores for same input (for 10% traffic)
  - Verify: `abs(library_score - embedded_score) < 0.01`
  - Alert if divergence detected

#### Deliverables

- [ ] Feature flag implemented
- [ ] 10% traffic routing working
- [ ] Monitoring dashboard updated
- [ ] Output comparison logs collected

---

### Phase 3: Cutover (Week 3: 2026-04-15 to 2026-04-21)

**Goal**: Switch 100% traffic to library-based scorer and remove embedded code.

#### Tasks

- [ ] **3.1**: Route 100% traffic to library-based scorer
  ```python
  # Update feature flag default
  NARRATIVE_SCORER_USE_LIBRARY=true  # In production config
  ```

- [ ] **3.2**: Remove embedded `core/services/narrative_scorer/` directory
  ```bash
  git rm -r core/services/narrative_scorer_embedded/
  ```

- [ ] **3.3**: Update imports in all core modules
  ```python
  # Before
  from .narrative_scorer_embedded import score_narrative
  
  # After
  from .narrative_scorer_wrapper import score_narrative
  ```

- [ ] **3.4**: Run full test suite
  - Unit tests
  - Integration tests
  - End-to-end tests
  - Performance tests

#### Deliverables

- [ ] 100% traffic on library-based scorer
- [ ] Embedded code removed
- [ ] All imports updated
- [ ] Full test suite passing

---

### Phase 4: Cleanup (Week 4: 2026-04-22 to 2026-04-28)

**Goal**: Finalize migration and document completion.

#### Tasks

- [ ] **4.1**: Remove compatibility layer (if no longer needed)
  - If wrapper is just a thin pass-through, consider removing
  - If wrapper provides value (stability, logging), keep it

- [ ] **4.2**: Update documentation
  - `core/README.md`: Update scorer integration docs
  - `core/docs/architecture.md`: Update data flow diagrams
  - `core/docs/api.md`: Update API references

- [ ] **4.3**: Tag release: `core v2.1.0`
  ```bash
  git tag -a v2.1.0 -m "Migrate to narrative-scorer library (embedded → library import)"
  git push origin v2.1.0
  ```

- [ ] **4.4**: Write migration retrospective
  - What went well?
  - What challenges arose?
  - Lessons learned for future migrations

#### Deliverables

- [ ] Documentation updated
- [ ] Release v2.1.0 tagged and pushed
- [ ] Migration retrospective written

---

## Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Version incompatibility | Low | High | Pin `narrative-scorer>=0.7.0,<0.8.0` |
| Breaking API changes | Low | High | Wrapper layer absorbs changes |
| Performance regression | Medium | Medium | Dual-run comparison before cutover |
| Import errors | Low | High | Comprehensive integration tests |
| Data corruption | Low | Critical | Output validation, rollback plan |

### Rollback Plan

If issues detected during Phase 2 or 3:

1. **Immediate**: Set `NARRATIVE_SCORER_USE_LIBRARY=false` (reverts to embedded)
2. **Investigate**: Analyze logs, identify root cause
3. **Fix**: Address issue in wrapper or library
4. **Retry**: Resume migration after fix validated

---

## Timeline Summary

| Phase | Start Date | End Date | Duration |
|-------|------------|----------|----------|
| Phase 1: Preparation | 2026-03-31 | 2026-04-07 | 1 week |
| Phase 2: Dual-Run | 2026-04-08 | 2026-04-14 | 1 week |
| Phase 3: Cutover | 2026-04-15 | 2026-04-21 | 1 week |
| Phase 4: Cleanup | 2026-04-22 | 2026-04-28 | 1 week |

**Total Duration**: 4 weeks
**Target Completion**: 2026-04-28

---

## Success Criteria

Migration is considered successful when:

1. ✅ 100% of narrative scoring uses library-based implementation
2. ✅ Embedded code fully removed from `core`
3. ✅ All tests passing (unit, integration, e2e)
4. ✅ No performance regression (latency within 10% of baseline)
5. ✅ No accuracy regression (scores match within tolerance)
6. ✅ Documentation updated
7. ✅ Release v2.1.0 tagged and deployed

---

## Dependencies

- **narrative-scorer v0.7.0**: Must be released before Phase 1 starts (target: 2026-03-31)
- **DASHSCOPE_API_KEY**: Required for v0.7.0 LLM validation (currently blocked >288h)
- **Core team availability**: For code review and deployment approval

---

## Post-Migration Maintenance

After migration complete:

1. **Monitor**: Watch for `narrative-scorer` updates (pip notifications)
2. **Upgrade**: Test and upgrade `narrative-scorer` quarterly (or as needed)
3. **Deprecate**: Plan for `narrative-scorer v0.8.0` breaking changes (if any)

---

## References

- **Repo Differentiation Doc**: `core/docs/repo-differentiation.md` (GEO #71)
- **Integration Patterns**: See Pattern 1 (Library Import) in differentiation doc
- **narrative-scorer Repo**: https://github.com/cittaverse/narrative-scorer
- **core Repo**: https://github.com/cittaverse/core

---

*Migration plan created for GEO #72. Execution starts 2026-03-31 (after v0.7.0 release).*
