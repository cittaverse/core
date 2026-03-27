# Repository Differentiation: core vs narrative-scorer

**Version**: 1.0  
**Last Updated**: 2026-03-27  
**Owner**: CittaVerse Engineering Team

---

## Overview

This document clarifies the responsibilities and boundaries between the `core` and `narrative-scorer` repositories to support v0.7 LLM integration and future development.

---

## Repository Purposes

### core

**Purpose**: Product logic, API services, user interaction, and business orchestration.

**Nature**: Application layer — the "brain" that coordinates all components.

**Key Responsibilities**:
- User authentication and session management
- WeChat ecosystem integration (Mini Program, Service Account)
- Voice interaction pipeline (ASR → TTS)
- Photo upload and processing
- Multi-agent empathy kernel (clinical analyst + listener personas)
- Safety layer (hard logic state machine, CST protocol, zero medical hallucination)
- Data persistence and user profile management
- Analytics dashboards (vocabulary trends, cognitive decline alerts)
- Cross-generational biography generation
- Subscription and payment integration
- Push notifications and report delivery

**Tech Stack**:
- Backend: FastAPI / Node.js (TBD)
- Database: PostgreSQL + Redis
- Cloud: Alibaba Cloud (China) / AWS (Overseas)
- Integrations: WeChat API, DashScope, Xunfei ASR

**Deployment**: Production environment with high availability requirements.

---

### narrative-scorer

**Purpose**: Narrative quality assessment algorithm, benchmark suite, and standalone scoring library.

**Nature**: Algorithm layer — a specialized, domain-specific computation engine.

**Key Responsibilities**:
- Six-dimension narrative scoring (event richness, temporal/causal coherence, emotional depth, identity integration, information density)
- Rule-based feature extraction (emotion words, temporal markers, causal markers, self-references)
- LLM-based feature extraction (v0.7+: implicit emotion detection, event segmentation, causality detection)
- Benchmark testing and validation
- Letter grade assignment (S/A/B/C/D/F)
- Natural language feedback generation
- Standalone Python library (pip-installable)
- Plugin integration (nlg-metricverse, etc.)

**Tech Stack**:
- Language: Python 3.8+
- Dependencies: dashscope, pyyaml, gradio (optional)
- Testing: pytest (72+ test cases)
- Benchmark: 15 gold-standard samples

**Deployment**: Can run standalone, embedded in core, or as a microservice.

---

## Boundary Matrix

| Function | core | narrative-scorer | Notes |
|----------|------|------------------|-------|
| User login/session | ✅ | ❌ | core owns user state |
| Voice ASR/TTS | ✅ | ❌ | core handles audio pipeline |
| Photo upload | ✅ | ❌ | core manages media storage |
| Safety guardrails | ✅ | ❌ | core enforces CST protocol |
| Narrative scoring | ⚠️ Consumer | ✅ Owner | core calls scorer API |
| Emotion detection | ⚠️ Consumer | ✅ Owner | scorer provides features |
| Event segmentation | ⚠️ Consumer | ✅ Owner | scorer provides features |
| Causality detection | ⚠️ Consumer | ✅ Owner | scorer provides features |
| Benchmark tests | ❌ | ✅ | scorer owns test suite |
| Letter grades | ❌ | ✅ | scorer defines grading logic |
| Feedback generation | ⚠️ Consumer | ✅ Owner | scorer provides base feedback |
| Dashboard analytics | ✅ | ❌ | core aggregates scores over time |
| Report generation | ✅ | ❌ | core formats user-facing reports |
| Subscription billing | ✅ | ❌ | core handles payments |

---

## Integration Patterns

### Pattern 1: Library Import (Recommended for v0.7)

```python
# In core/src/narrative_service.py
from narrative_scorer.scorer import score_narrative
from narrative_scorer.llm_feature_extractor import LLMFeatureExtractor

def process_user_narrative(text: str, user_id: str):
    # Step 1: Extract rule-based features
    rule_features = score_narrative(text, return_features=True)
    
    # Step 2: Extract LLM features (v0.7+)
    llm_extractor = LLMFeatureExtractor()
    llm_features = llm_extractor.extract_with_fallback(text, rule_features)
    
    # Step 3: Store in core's database
    save_to_database(user_id, llm_features.to_dict())
    
    # Step 4: Generate user-facing report
    return generate_report(llm_features)
```

**Pros**: Simple, no network overhead, version pinning via pip.  
**Cons**: Requires core to manage scorer dependency updates.

---

### Pattern 2: Microservice API

```python
# In core/src/narrative_service.py
import httpx

SCORER_API_URL = "http://narrative-scorer-service:8000"

async def process_user_narrative(text: str, user_id: str):
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{SCORER_API_URL}/score",
            json={"text": text, "include_llm": True}
        )
        features = response.json()
    
    save_to_database(user_id, features)
    return generate_report(features)
```

**Pros**: Loose coupling, independent scaling, language-agnostic.  
**Cons**: Network latency, deployment complexity.

---

### Pattern 3: Embedded Module (Current v0.6)

```
core/
├── src/
│   ├── scorer.py  (copied from narrative-scorer)
│   └── ...
```

**Pros**: No dependency management.  
**Cons**: Code duplication, sync drift, maintenance burden.

**Status**: ❌ **Deprecated** — migrate to Pattern 1 by v0.7 GA.

---

## Version Compatibility

| core Version | narrative-scorer Version | Integration Method | Status |
|--------------|--------------------------|--------------------|--------|
| v1.x | v0.5.x | Embedded copy | Deprecated |
| v2.0 | v0.6.x | Embedded copy | Current (to migrate) |
| v2.1 | v0.7.x | Library import (Pattern 1) | Target |
| v2.2+ | v0.7.x+ | Microservice (Pattern 2) | Optional |

---

## Data Flow (v0.7+)

```
User Voice (WeChat)
       │
       ▼
┌─────────────────────┐
│   core (ASR)        │
│   Speech → Text     │
└─────────────────────┘
       │
       │ Text
       ▼
┌─────────────────────┐
│   narrative-scorer  │
│   • Rule features   │
│   • LLM features    │
│   • Composite score │
│   • Letter grade    │
└─────────────────────┘
       │
       │ Features + Score
       ▼
┌─────────────────────┐
│   core (Analytics)  │
│   • Store in DB     │
│   • Update trends   │
│   • Trigger alerts  │
└─────────────────────┘
       │
       │ Report
       ▼
┌─────────────────────┐
│   core (Delivery)   │
│   • WeChat push     │
│   • Dashboard       │
│   • Biography gen   │
└─────────────────────┘
```

---

## Decision Log

### 2026-03-27: Formalize Separation

**Decision**: Create explicit boundary documentation to support v0.7 LLM integration.

**Rationale**:
- Avoid code duplication (scorer.py currently embedded in core)
- Enable independent versioning and testing
- Clarify ownership for LLM feature extractor
- Prepare for microservice architecture (optional future)

**Action Items**:
- [x] Create `core/docs/repo-differentiation.md` (this document)
- [ ] Migrate core from embedded scorer to library import (v2.1)
- [ ] Add narrative-scorer to core's requirements.txt
- [ ] Update core's narrative_service.py to use Pattern 1

---

## FAQ

### Q: Can core modify scorer logic directly?

**A**: No. All scoring algorithm changes should be made in `narrative-scorer` repo, then consumed by core via version upgrade. This ensures:
- Single source of truth
- Benchmark validation before deployment
- Clear changelog for algorithm updates

### Q: What if core needs custom scoring logic?

**A**: Two options:
1. **Extend narrative-scorer**: Add new feature/dimension to scorer (preferred for reusable features)
2. **Post-processing in core**: Apply core-specific adjustments to scorer output (for product-specific logic only)

### Q: Who owns the LLM prompts?

**A**: `narrative-scorer` owns all prompt templates (`llm_prompts/`), as they are part of the scoring algorithm. Core only configures LLM settings (model, timeout, API key) via environment variables.

### Q: How do we handle breaking changes in scorer API?

**A**: Follow semantic versioning:
- **Minor (0.6 → 0.7)**: Backward compatible (new features, no breaking changes)
- **Major (0.7 → 1.0)**: May break API — requires core migration guide
- **Patch (0.6.3 → 0.6.4)**: Bug fixes only

---

## Related Documents

- [v0.7 LLM Integration Guide](../../pipeline/docs/v0.7-llm-integration-guide.md)
- [Narrative Scorer README](../../narrative-scorer/README.md)
- [LLM-as-Judge Architecture](../../pipeline/docs/llm_as_judge_architecture.md)
- [Core Product Architecture](./index.md)

---

*This document is part of CittaVerse engineering documentation. For questions, open an issue in the core repo or contact engineering@cittaverse.com.*
