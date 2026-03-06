# Context Stack Integration (v0.3.20)

Context stacks provide **reproducible, structured prompts** for LLM interactions. In v0.3.20, SPINE executors can load context stacks from YAML scenario files for consistent prompt building.

---

## Overview

<table style="border-collapse: collapse; font-family: 'Inter', system-ui, sans-serif; width: 100%; max-width: 600px;">
  <tr>
    <th colspan="2" style="background: #1e293b; color: #e2e8f0; padding: 12px 16px; text-align: left; font-size: 1.05em; border-radius: 8px 8px 0 0;">Context Stack</th>
  </tr>
  <tr>
    <td style="background: #2563eb; color: #fff; padding: 10px 16px; font-weight: 600; width: 140px; border-bottom: 2px solid #0f172a;">global</td>
    <td style="background: #0f172a; color: #cbd5e1; padding: 10px 16px; border-bottom: 2px solid #1e293b;">operator, brand identity</td>
  </tr>
  <tr>
    <td style="background: #7c3aed; color: #fff; padding: 10px 16px; font-weight: 600; border-bottom: 2px solid #0f172a;">character</td>
    <td style="background: #0f172a; color: #cbd5e1; padding: 10px 16px; border-bottom: 2px solid #1e293b;">speaker persona, target audience</td>
  </tr>
  <tr>
    <td style="background: #ec4899; color: #fff; padding: 10px 16px; font-weight: 600; border-bottom: 2px solid #0f172a;">command</td>
    <td style="background: #0f172a; color: #cbd5e1; padding: 10px 16px; border-bottom: 2px solid #1e293b;">task specification, success criteria</td>
  </tr>
  <tr>
    <td style="background: #f59e0b; color: #000; padding: 10px 16px; font-weight: 600; border-bottom: 2px solid #0f172a;">constraints</td>
    <td style="background: #0f172a; color: #cbd5e1; padding: 10px 16px; border-bottom: 2px solid #1e293b;">tone, format, do/don't rules</td>
  </tr>
  <tr>
    <td style="background: #0d9488; color: #fff; padding: 10px 16px; font-weight: 600; border-bottom: 2px solid #0f172a;">context</td>
    <td style="background: #0f172a; color: #cbd5e1; padding: 10px 16px; border-bottom: 2px solid #1e293b;">background info, references</td>
  </tr>
  <tr>
    <td style="background: #ef4444; color: #fff; padding: 10px 16px; font-weight: 600; border-radius: 0 0 0 8px;">input</td>
    <td style="background: #0f172a; color: #cbd5e1; padding: 10px 16px; border-radius: 0 0 8px 0;">user request</td>
  </tr>
</table>

---

## Scenario File Structure

Scenario files are YAML configurations that define context stacks:

```yaml
# scenarios/research.yaml
name: "Research Task"
description: "Context stack for research-focused tasks"

context_stack:
  global:
    operator: "AdaptiveArts.ai"
    brand: "Technical Research Initiative"

  character:
    speaker: "Senior Research Analyst"
    audience: "Technical stakeholders"

  command:
    task: "Conduct comprehensive research on the given topic"
    success: |
      - All sources verified and cited
      - Key findings synthesized
      - Recommendations provided

  constraints:
    tone: "Professional, analytical"
    format: "Structured report with sections"
    do:
      - "Cite all sources"
      - "Identify contradictions"
      - "Provide actionable insights"
    dont:
      - "Make unsupported claims"
      - "Skip source verification"

  context:
    background: "Research conducted for strategic planning"
    references:
      - "Company knowledge base"
      - "Industry reports"
```

---

## Using Context Stacks with Executors

### CLI Usage

```bash
# Use a specific scenario for all tasks
python -m spine.orchestrator run --project /path \
    --executor subagent \
    --scenario scenarios/research.yaml

# Role-specific scenarios
python -m spine.orchestrator run --project /path \
    --executor claude-code \
    --role-scenario "architect:scenarios/architecture.yaml" \
    --role-scenario "reviewer:scenarios/code-review.yaml" \
    --role-scenario "implementer:scenarios/coding.yaml"

# Disable context stacks (use legacy prompts)
python -m spine.orchestrator run --project /path \
    --executor subagent \
    --no-context-stacks
```

### Programmatic Usage

```python
from spine.orchestrator.context_stack import (
    ContextStack,
    load_context_stack,
    get_default_scenario,
)

# Load from YAML file
stack = load_context_stack("scenarios/research.yaml")

# Add task-specific context
stack = stack.with_task_context(
    task_title="Analyze competitor landscape",
    task_description="Research top 5 competitors in the AI space",
    role="researcher",
    task_id="task-001",
)

# Build prompts
system_prompt = stack.build_system_prompt()
user_prompt = stack.build_user_prompt("Focus on pricing strategies")
```

---

## Built-in Default Scenarios

SPINE provides default scenarios for common roles:

| Role | Default Focus |
|------|---------------|
| `architect` | System design, architectural decisions |
| `implementer` | Code implementation, best practices |
| `reviewer` | Code review, quality assurance |
| `researcher` | Research, analysis, synthesis |

Access default scenarios:

```python
from spine.orchestrator.context_stack import get_default_scenario

# Get default scenario for a role
scenario_path = get_default_scenario("architect")
stack = load_context_stack(scenario_path)
```

---

## Sample Scenario Templates

SPINE includes sample templates in `_templates/scenarios/`:

| Template | Purpose |
|----------|---------|
| `sample-research.yaml` | Research task template |
| `sample-code-review.yaml` | Code review template |
| `sample-content-generation.yaml` | Content creation template |
| `sample-analysis-pipeline.yaml` | Multi-step analysis template |

---

## Context Stack Layers Explained

### 1. Global Layer
System-wide configuration that applies to all interactions:
```yaml
global:
  operator: "Company Name"      # Who operates this system
  brand: "Product/Initiative"   # Brand identity
```

### 2. Character Layer
Defines the AI's persona and audience:
```yaml
character:
  speaker: "Senior Engineer"    # AI's role/persona
  audience: "Development team"  # Who receives the output
```

### 3. Command Layer
The core task specification:
```yaml
command:
  task: "What to do"            # Clear task description
  success: "Success criteria"   # How to measure completion
```

### 4. Constraints Layer
Boundaries and guidelines:
```yaml
constraints:
  tone: "Professional"          # Communication style
  format: "Markdown report"     # Output format
  do:                           # Things to include
    - "Cite sources"
    - "Provide examples"
  dont:                         # Things to avoid
    - "Make assumptions"
    - "Skip validation"
```

### 5. Context Layer
Background information:
```yaml
context:
  background: "Project context" # Situational info
  references:                   # Reference materials
    - "docs/architecture.md"
    - "Previous analysis"
```

### 6. Input Layer
The actual user request (typically added at runtime):
```yaml
input:
  user_request: "Analyze the authentication module"
```

---

## Integration with Executors

### SubagentExecutor

```python
from spine.orchestrator.executors import SubagentExecutor, SubagentConfig

config = SubagentConfig(
    agent_dir=project_path / ".claude" / "agents",
    role_scenarios={
        "architect": "scenarios/architecture.yaml",
        "reviewer": "scenarios/code-review.yaml",
    },
    default_scenario="scenarios/default.yaml",
    use_context_stacks=True,
)
executor = SubagentExecutor(config)
```

### ClaudeCodeExecutor

```python
from spine.orchestrator.executors import ClaudeCodeExecutor, ClaudeCodeConfig

config = ClaudeCodeConfig(
    model="opus",
    role_scenarios={"researcher": "scenarios/research.yaml"},
    use_context_stacks=True,
)
executor = ClaudeCodeExecutor(config)
```

---

## Benefits of Context Stacks

| Benefit | Description |
|---------|-------------|
| **Reproducibility** | Same scenario = same prompt structure |
| **Separation** | Task logic separate from prompt engineering |
| **Versioning** | YAML files can be version-controlled |
| **Reusability** | Share scenarios across projects |
| **Consistency** | Uniform AI behavior across tasks |

---

[← Back to Docs](README.md) | [Executors](executors.md)
