# SPINE Showcase Backlog

> Planned features and demos for the SPINE showcase

---

## Demos

### Done

| Demo | Description | Status |
|------|-------------|--------|
| [Tier Classifier](demos/tier-classifier/) | Interactive 4-question task classifier | Done |

### Planned

| Demo | Description | Priority | Source |
|------|-------------|----------|--------|
| Provider Picker | Select use case, get LLM recommendation | High | `providers.py` |
| Cost Calculator | Estimate API costs by model and tokens | High | `providers.py::PRICING` |
| Fan-Out Simulator | Visualize parallel task execution | Medium | `fan_out.py` |
| Pipeline Builder | Build and visualize sequential workflows | Medium | `pipeline.py` |
| Context Stack Builder | Compose hierarchical context stacks | Low | Context stack docs |

---

## Demo Details

### Provider Picker

**Concept:** Help users choose the right LLM provider for their task.

**Interactions:**
- Select task type (planning, execution, multimodal, discovery)
- See recommended provider and model
- View pricing comparison
- Show capability matrix

**Data to extract:**
- ProviderType enum
- DEFAULT_MODELS mapping
- PRICING dictionary
- Use-case recommendations from comments

---

### Cost Calculator

**Concept:** Real-time cost estimation for LLM API usage.

**Interactions:**
- Select model from dropdown
- Adjust input/output token sliders
- See cost update in real-time
- Compare multiple models

**Data to extract:**
- PRICING dictionary (per-model rates)
- estimate_cost() logic

---

### Fan-Out Simulator

**Concept:** Visualize how parallel execution works.

**Interactions:**
- Set number of tasks
- Set max_workers (parallel limit)
- Click "Run" to simulate
- Watch animated timeline
- See success/failure aggregation

**Visual elements:**
- Task boxes with progress
- Timeline with concurrent execution
- Summary stats (duration, success rate)

---

### Pipeline Builder

**Concept:** Build and visualize sequential processing chains.

**Interactions:**
- Add/remove pipeline steps
- Define step names
- See visual flow diagram
- Simulate data passing through
- Toggle error handling modes

**Visual elements:**
- Step boxes connected by arrows
- Data flow animation
- Error state visualization

---

## Documentation

| Doc | Status |
|-----|--------|
| Architecture Overview | Done |
| Tiered Enforcement | Done |
| Pattern Guide | Done |
| API Reference | Not planned (showcase only) |

---

## Improvements

| Item | Priority | Notes |
|------|----------|-------|
| Mobile responsiveness | Medium | Current demos work but could be better |
| Dark mode toggle | Low | Currently follows system preference |
| Demo gallery thumbnails | Low | Add preview images |

---

## Source Reference

Content generated from: `D:\temp\myprojects\spine`
Feature spec: `D:\temp\myprojects\showcasing-projects\docs\feature-auto-demo-extraction.md`

---

*Last updated: 2025-12-13*
