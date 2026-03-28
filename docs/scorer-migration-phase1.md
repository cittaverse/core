# Scorer Migration Phase 1: Preparation Checklist

**Date**: 2026-03-28
**Author**: Hulk (GEO #74)
**Status**: Ready for Execution (Starts 2026-03-31)
**Parent Plan**: [scorer-migration-plan.md](scorer-migration-plan.md)

---

## Phase 1 Overview

**Duration**: Week 1 (2026-03-31 to 2026-04-07)
**Goal**: Set up infrastructure for library-based integration
**Risk Level**: Low (no production impact — preparation only)

---

## Pre-requisites

Before starting Phase 1, ensure:

- [ ] **narrative-scorer v0.7.0 released** (blocked by DASHSCOPE_API_KEY validation)
- [ ] **narrative-scorer published to PyPI** (`pip install narrative-scorer` works)
- [ ] **Core team notified** of migration start date (2026-03-31)

### Current Blockers

| Blocker | Owner | Status | Notes |
|---------|-------|--------|-------|
| DASHSCOPE_API_KEY | V | ❌ Blocked >288h | v0.7 LLM validation pending |
| narrative-scorer v0.7.0 release | Hulk | ⏳ Pending | Awaiting API key for V4 testing |

**Mitigation**: Phase 1 can start with v0.6.4 as temporary pin (`narrative-scorer>=0.6.4,<0.7.0`), then upgrade to v0.7.0+ once released.

---

## Task 1.1: Update core/requirements.txt

### Current State

```bash
# Check if requirements.txt exists
ls -la /home/node/.openclaw/workspace-hulk/github-repos/core/requirements.txt
```

**Note**: As of 2026-03-28, `core/requirements.txt` does NOT exist. Need to investigate dependency management approach.

### Investigation Required

```bash
# Check for alternative dependency files
find /home/node/.openclaw/workspace-hulk/github-repos/core -name "*.toml" -o -name "setup.py" -o -name "Pipfile" -o -name "pyproject.toml"
```

### Action Plan

**If requirements.txt exists**:
```diff
+ # narrative-scorer library (v0.7.0+)
+ narrative-scorer>=0.7.0,<0.8.0
```

**If requirements.txt does NOT exist**:
1. Create `core/requirements.txt` with base dependencies
2. Add `narrative-scorer>=0.7.0,<0.8.0` as first external library dependency
3. Document decision in `core/docs/dependency-management.md`

### Deliverable

- [ ] `core/requirements.txt` created/updated with `narrative-scorer>=0.7.0,<0.8.0`

---

## Task 1.2: Create core/services/narrative_scorer_wrapper.py

### Purpose

Provide a stable API layer that absorbs breaking changes in `narrative-scorer` library.

### Implementation

**File**: `core/services/narrative_scorer_wrapper.py`

```python
#!/usr/bin/env python3
"""
Narrative Scorer Wrapper for Core Services

This wrapper provides a stable API for core services,
absorbing any breaking changes in the narrative-scorer library.

Version: 1.0.0 (Phase 1)
Compatibility: narrative-scorer>=0.7.0,<0.8.0
"""

from typing import Optional, Dict, Any
import logging

logger = logging.getLogger(__name__)

# Import from narrative-scorer library
try:
    from narrative_scorer import score_narrative as _score_narrative
    from narrative_scorer import LLMFeatureExtractor, LLMConfig
    LIBRARY_AVAILABLE = True
except ImportError as e:
    logger.warning(f"narrative-scorer library not available: {e}")
    LIBRARY_AVAILABLE = False


def score_narrative(text: str, use_llm: bool = True, api_key: Optional[str] = None) -> Dict[str, Any]:
    """
    Score a single narrative (wrapper for library function).
    
    Args:
        text: The narrative text to score
        use_llm: Whether to use LLM-enhanced scoring (default: True)
        api_key: Optional DashScope API key (falls back to env var if not provided)
    
    Returns:
        Dictionary with scores and metadata:
        {
            'composite_score': float,
            'event_richness': float,
            'temporal_causal_coherence': float,
            'emotional_depth': float,
            'identity_integration': float,
            'information_density': float,
            'narrative_coherence': float,
            'metadata': dict,
        }
    
    Raises:
        ImportError: If narrative-scorer library is not installed
        Exception: If scoring fails (with graceful fallback logging)
    """
    if not LIBRARY_AVAILABLE:
        raise ImportError(
            "narrative-scorer library is required. "
            "Install with: pip install narrative-scorer>=0.7.0,<0.8.0"
        )
    
    try:
        return _score_narrative(text, use_llm=use_llm, api_key=api_key)
    except Exception as e:
        logger.error(f"Narrative scoring failed: {e}")
        raise


class NarrativeScorerService:
    """
    Service layer for narrative scoring (used by core APIs).
    
    This service class provides:
    - Configuration management (LLM on/off, API key handling)
    - Logging and monitoring hooks
    - Batch scoring support
    - Error handling and fallback behavior
    
    Usage:
        scorer = NarrativeScorerService(use_llm=True)
        result = scorer.score("今天天气很好...")
        print(result['composite_score'])
    """
    
    def __init__(
        self,
        use_llm: bool = True,
        api_key: Optional[str] = None,
        fallback_to_rule_only: bool = True,
    ):
        """
        Initialize the narrative scorer service.
        
        Args:
            use_llm: Enable LLM-enhanced scoring (default: True)
            api_key: DashScope API key (optional, falls back to env var)
            fallback_to_rule_only: Fall back to rule-only if LLM fails (default: True)
        """
        self.use_llm = use_llm
        self.api_key = api_key
        self.fallback_to_rule_only = fallback_to_rule_only
        
        if use_llm and LIBRARY_AVAILABLE:
            config = LLMConfig(
                api_key=api_key,
                fallback_to_rule_only=fallback_to_rule_only,
            )
            self.extractor = LLMFeatureExtractor(config)
        else:
            self.extractor = None
        
        logger.info(
            f"NarrativeScorerService initialized: "
            f"use_llm={use_llm}, fallback={fallback_to_rule_only}"
        )
    
    def score(self, text: str) -> Dict[str, Any]:
        """
        Score a narrative.
        
        Args:
            text: The narrative text to score
        
        Returns:
            Scoring result dictionary (see score_narrative())
        """
        return score_narrative(text, use_llm=self.use_llm, api_key=self.api_key)
    
    def score_batch(self, texts: list[str]) -> list[Dict[str, Any]]:
        """
        Score multiple narratives in batch.
        
        Args:
            texts: List of narrative texts
        
        Returns:
            List of scoring result dictionaries
        """
        results = []
        for i, text in enumerate(texts):
            try:
                result = self.score(text)
                results.append(result)
            except Exception as e:
                logger.error(f"Batch scoring failed for text {i}: {e}")
                results.append({'error': str(e)})
        return results


# Convenience function for quick imports
def get_scorer(use_llm: bool = True) -> NarrativeScorerService:
    """Get a configured scorer instance."""
    return NarrativeScorerService(use_llm=use_llm)
```

### Deliverable

- [ ] `core/services/narrative_scorer_wrapper.py` created
- [ ] Code reviewed for style and consistency
- [ ] Unit tests written (Task 1.3)

---

## Task 1.3: Write Integration Tests

### File: `core/tests/test_narrative_scorer_wrapper.py`

```python
#!/usr/bin/env python3
"""
Integration Tests for Narrative Scorer Wrapper

Tests the wrapper layer without requiring live LLM API calls.
Uses mocked responses to validate wrapper behavior.
"""

import unittest
from unittest.mock import patch, MagicMock
import sys
import os

sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..'))

from services.narrative_scorer_wrapper import (
    score_narrative,
    NarrativeScorerService,
    get_scorer,
    LIBRARY_AVAILABLE,
)


class TestWrapperAvailability(unittest.TestCase):
    """Test library availability detection"""

    def test_library_available(self):
        """Verify narrative-scorer library is importable"""
        # This test will fail if requirements.txt not installed
        self.assertTrue(
            LIBRARY_AVAILABLE,
            "narrative-scorer library should be available (check requirements.txt)"
        )


class TestScoreNarrativeFunction(unittest.TestCase):
    """Test score_narrative() wrapper function"""

    @patch('services.narrative_scorer_wrapper._score_narrative')
    def test_score_narrative_calls_library(self, mock_score):
        """Wrapper should delegate to library function"""
        mock_score.return_value = {'composite_score': 75.0}
        
        result = score_narrative("Test narrative")
        
        mock_score.assert_called_once_with("Test narrative", use_llm=True, api_key=None)
        self.assertEqual(result['composite_score'], 75.0)

    @patch('services.narrative_scorer_wrapper._score_narrative')
    def test_score_narrative_with_llm_flag(self, mock_score):
        """Wrapper should pass use_llm flag to library"""
        mock_score.return_value = {'composite_score': 70.0}
        
        score_narrative("Test", use_llm=False)
        
        mock_score.assert_called_once_with("Test", use_llm=False, api_key=None)

    @patch('services.narrative_scorer_wrapper._score_narrative')
    def test_score_narrative_with_api_key(self, mock_score):
        """Wrapper should pass API key to library"""
        mock_score.return_value = {'composite_score': 80.0}
        
        score_narrative("Test", api_key="test-key-123")
        
        mock_score.assert_called_once_with("Test", use_llm=True, api_key="test-key-123")


class TestNarrativeScorerService(unittest.TestCase):
    """Test NarrativeScorerService class"""

    @patch('services.narrative_scorer_wrapper.LLMFeatureExtractor')
    def test_service_initialization_with_llm(self, mock_extractor):
        """Service should initialize LLM extractor when use_llm=True"""
        service = NarrativeScorerService(use_llm=True)
        
        self.assertTrue(service.use_llm)
        self.assertIsNotNone(service.extractor)
        mock_extractor.assert_called_once()

    def test_service_initialization_without_llm(self):
        """Service should skip LLM extractor when use_llm=False"""
        service = NarrativeScorerService(use_llm=False)
        
        self.assertFalse(service.use_llm)
        self.assertIsNone(service.extractor)

    @patch('services.narrative_scorer_wrapper._score_narrative')
    def test_service_score(self, mock_score):
        """Service.score() should delegate to score_narrative()"""
        mock_score.return_value = {'composite_score': 72.5}
        service = NarrativeScorerService(use_llm=True)
        
        result = service.score("Test narrative")
        
        self.assertEqual(result['composite_score'], 72.5)
        mock_score.assert_called_once()

    @patch('services.narrative_scorer_wrapper._score_narrative')
    def test_service_score_batch(self, mock_score):
        """Service should support batch scoring"""
        mock_score.side_effect = [
            {'composite_score': 70.0},
            {'composite_score': 75.0},
            {'composite_score': 80.0},
        ]
        service = NarrativeScorerService(use_llm=True)
        
        results = service.score_batch(["Text 1", "Text 2", "Text 3"])
        
        self.assertEqual(len(results), 3)
        self.assertEqual(results[0]['composite_score'], 70.0)
        self.assertEqual(results[1]['composite_score'], 75.0)
        self.assertEqual(results[2]['composite_score'], 80.0)
        self.assertEqual(mock_score.call_count, 3)

    @patch('services.narrative_scorer_wrapper._score_narrative')
    def test_service_score_batch_with_errors(self, mock_score):
        """Batch scoring should handle individual failures gracefully"""
        mock_score.side_effect = [
            {'composite_score': 70.0},
            Exception("API error"),
            {'composite_score': 80.0},
        ]
        service = NarrativeScorerService(use_llm=True)
        
        results = service.score_batch(["Text 1", "Text 2", "Text 3"])
        
        self.assertEqual(len(results), 3)
        self.assertEqual(results[0]['composite_score'], 70.0)
        self.assertIn('error', results[1])
        self.assertEqual(results[2]['composite_score'], 80.0)


class TestGetScorerHelper(unittest.TestCase):
    """Test get_scorer() convenience function"""

    @patch('services.narrative_scorer_wrapper.NarrativeScorerService')
    def test_get_scorer_returns_service(self, mock_service):
        """get_scorer() should return NarrativeScorerService instance"""
        get_scorer(use_llm=True)
        
        mock_service.assert_called_once_with(use_llm=True)


if __name__ == "__main__":
    unittest.main(verbosity=2)
```

### Deliverable

- [ ] `core/tests/test_narrative_scorer_wrapper.py` created
- [ ] All tests passing (run with `pytest core/tests/test_narrative_scorer_wrapper.py -v`)

---

## Task 1.4: Document Breaking Changes

### Analysis: Embedded API vs Library API

**Embedded API** (current in core):
```python
from services.narrative_scorer_embedded import score_narrative

result = score_narrative(text)
# Returns: dict with composite_score + 6 dimensions
```

**Library API** (narrative-scorer v0.7.0):
```python
from narrative_scorer import score_narrative

result = score_narrative(text, use_llm=True, api_key=None)
# Returns: dict with composite_score + 6 dimensions + metadata
```

### Breaking Changes

| Change | Impact | Mitigation |
|--------|--------|------------|
| New `use_llm` parameter (default: True) | Low (backward compatible — default matches current behavior) | Wrapper sets explicit default |
| New `api_key` parameter | Low (optional, falls back to env var) | Wrapper handles env var |
| New `metadata` field in response | None (additive change) | No action needed |
| LLM scoring requires DASHSCOPE_API_KEY | Medium (new dependency) | Fallback to rule-only if key missing |

### Conclusion

**No breaking changes** — Library API is backward compatible with embedded API.
Wrapper layer provides additional safety and logging.

### Deliverable

- [ ] `core/docs/scorer-migration-notes.md` created with API comparison
- [ ] Breaking changes documented (none identified)

---

## Phase 1 Completion Checklist

| Task | Status | Owner | Due Date |
|------|--------|-------|----------|
| 1.1 Update requirements.txt | ⏳ Pending | Core | 2026-04-02 |
| 1.2 Create wrapper layer | ⏳ Pending | Core | 2026-04-04 |
| 1.3 Write integration tests | ⏳ Pending | Core | 2026-04-05 |
| 1.4 Document breaking changes | ⏳ Pending | Core | 2026-04-07 |

**Phase 1 Gate Review**: 2026-04-07
- All 4 tasks complete → Proceed to Phase 2 (Dual-Run)
- Any task blocked → Extend Phase 1 or escalate

---

## Next Steps (Phase 2 Preview)

After Phase 1 complete:

1. **Phase 2: Dual-Run** (2026-04-08 to 2026-04-14)
   - Deploy wrapper alongside embedded code
   - Route 10% traffic to library-based scorer
   - Compare outputs, monitor metrics

2. **Phase 3: Cutover** (2026-04-15 to 2026-04-21)
   - Route 100% traffic to library
   - Remove embedded code
   - Update all imports

3. **Phase 4: Cleanup** (2026-04-22 to 2026-04-28)
   - Documentation updates
   - Release v2.1.0 tagged
   - Migration retrospective

---

## References

- [Parent Migration Plan](scorer-migration-plan.md)
- [narrative-scorer Repo](https://github.com/cittaverse/narrative-scorer)
- [core Repo](https://github.com/cittaverse/core)
- [Repo Differentiation Doc](repo-differentiation.md)

---

*Phase 1 prep document created for GEO #74. Execution starts 2026-03-31.*
