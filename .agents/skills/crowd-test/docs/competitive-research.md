# CrowdTest Competitive Research — Technical Deep Dive

> Date: 2026-03-22
> Scope: ArkSim, Channel-Labs/synthetic-conversation-generation, Tencent PersonaHub, Playwright Test Agents

## Executive Summary

We analyzed four open-source projects across three capability domains:

| Domain | Project | What It Does | What It Doesn't Do |
|--------|---------|-------------|-------------------|
| Evaluation | ArkSim | LLM-as-judge metrics, error dedup, HTML reports | No browser automation, no persona generation |
| Persona Generation | Channel-Labs | Prompt-driven persona diversity, decoupled pipeline | No evaluation, no UI testing, doesn't scale past ~50 personas |
| Persona Generation | PersonaHub | 1B personas from web data, MinHash+embedding dedup | No agent execution, no testing, no feedback |
| Browser Automation | Playwright Agents | 21 MCP browser tools, accessibility snapshots | No personas, no user simulation, no product feedback |

**Key finding**: No project combines persona generation + browser automation + structured product feedback. This is CrowdTest's opportunity.

---

## 1. ArkSim (ArkLex AI)

**Repo**: https://github.com/ArkLexAI/arksim
**License**: Apache 2.0 | **Language**: Python 3.10-3.13

### 1.1 Architecture

Four modules with clear separation of concerns:

```
arksim/
  scenario/           # Test case definitions (persona + goal + knowledge)
  simulation_engine/  # Generates synthetic multi-turn conversations
  evaluator/          # LLM-as-judge scoring (turn-level + conversation-level)
  utils/html_report/  # Standalone HTML report generation
  cli.py              # Orchestration: simulate | evaluate | simulate-evaluate
```

### 1.2 Persona/Scenario Data Structure

The `Scenario` is the central entity — a persona + test case:

```python
class Scenario(BaseModel):
    scenario_id: str
    user_id: str
    goal: str                          # What the simulated user wants to accomplish
    agent_context: str                 # Context about the agent being tested
    knowledge: list[KnowledgeItem]     # Ground truth for faithfulness checks
    user_profile: str                  # Free-text persona narrative
    origin: dict                       # Raw structured attributes (metadata only)
```

The `user_profile` is a **pre-rendered narrative string** — structured attributes (Big Five personality, demographics, spending behavior, loyalty level, etc.) live in `origin` as provenance but are NOT consumed by the simulation. Example:

```
"You are Jasmine Rivera who is open to experience, conscientious, introverted..."
```

**Implication for CrowdTest**: ArkSim expects personas to be pre-authored. It has no generation pipeline. CrowdTest needs its own.

### 1.3 Simulation Execution Flow

```
Load scenarios.json
  → For each scenario, spawn N conversations (default: 5)
    → For each turn (up to max_turns=5):
       1. Render simulated user prompt via Jinja2 template
          (injects user_profile, goal, knowledge, agent_context)
       2. LLM generates simulated user message
       3. If message contains "###STOP###" → end (goal satisfied)
       4. Send user message to agent-under-test
       5. Collect agent response (including tool calls)
       6. Append to conversation history
```

Concurrent execution via `asyncio.create_task` with configurable worker limits.

### 1.4 Evaluation System (Most Reusable Component)

Two-phase parallel evaluation:

**Phase 1 — Turn-level metrics** (all turns, parallel via ThreadPoolExecutor):

| Metric | Scale | What It Measures |
|--------|-------|-----------------|
| Helpfulness | 1-5 | How well the response addresses user needs |
| Coherence | 1-5 | Logical flow and consistency |
| Verbosity | 1-5 | Conciseness (inverted) |
| Relevance | 1-5 | How on-topic the response is |
| Faithfulness | 1-5 | Whether response contradicts provided knowledge |

If any metric < 3.0, triggers qualitative `AgentBehaviorFailureMetric`:

```
Failure categories (with severity):
  - false information      → critical
  - unsafe action          → critical
  - unsafe state           → critical
  - disobey user request   → high
  - lack of specific info  → medium
  - failure to clarify     → medium
  - repetition             → low
```

**Phase 2 — Conversation-level goal completion** (parallel):

```python
overall_agent_score = 0.75 * turn_success_ratio + 0.25 * goal_completion_score
```

**Phase 3 — Error deduplication** (sequential):
All behavior failure reasons across conversations → LLM deduplicates into unique root-cause errors → each gets a severity level.

```python
class UniqueError(BaseModel):
    unique_error_id: str
    behavior_failure_category: str
    unique_error_description: str
    severity: str                     # "critical", "high", "medium", "low"
    occurrences: list[Occurrence]     # Which conversations/turns hit this
```

### 1.5 Custom Metrics Extension Point

```python
class QuantitativeMetric(abc.ABC):
    def __init__(self, name, score_range=(1,5), additional_input=None, description=""): ...
    def score(self, score_input: ScoreInput) -> QuantResult: ...

class QualitativeMetric(abc.ABC):
    def __init__(self, name, description="", label_colors=None): ...
    def evaluate(self, score_input: ScoreInput) -> QualResult: ...
```

Custom metrics loaded dynamically from Python files specified in config via `custom_metrics_file_paths`.

### 1.6 Configuration (Single YAML)

```yaml
agent_config:
  agent_type: chat_completions | a2a | custom
  agent_name: <name>
  api_config: { endpoint, headers, body }

scenario_file_path: ./scenarios.json
num_conversations_per_scenario: 5
max_turns: 5
output_file_path: ./simulation.json

output_dir: ./evaluation
metrics_to_run: [faithfulness, helpfulness, ...]
custom_metrics_file_paths: []
generate_html_report: true
numeric_thresholds: { overall_score: 0.7 }
qualitative_failure_labels: { agent_behavior_failure: ["false information"] }

model: gpt-4o
provider: openai
num_workers: 50
```

### 1.7 Report Generation

Standalone HTML file (no server needed):
1. Build structured data objects (`ReportSummary`, `ConvoRow`, `TurnRow`, `ErrorRow`)
2. Serialize to JSON, inject into HTML template via `{{FINAL_REPORT_DATA}}` placeholders
3. Includes: performance metrics summary, per-conversation scores, turn-by-turn drill-down, unique error cards

### 1.8 CI/CD Integration

- `numeric_thresholds` and `qualitative_failure_labels` → CLI exits code 1 on failure
- `arksim simulate-evaluate config.yaml` as single CI command

### 1.9 Reusable for CrowdTest

| Pattern | Applicability |
|---------|--------------|
| `QuantitativeMetric` / `QualitativeMetric` ABC | Adapt for product feedback scoring (usability, clarity, accessibility) |
| Two-phase parallel evaluation | Turn-level → session-level aggregation |
| Error deduplication via LLM | Cross-persona bug dedup (unique product issues) |
| Threshold gates + exit codes | CI/CD pipeline integration |
| Standalone HTML report template | Report generation pattern |
| Jinja2 simulated user prompt | Persona prompt templating |

### 1.10 Limitations

- No browser automation or UI testing
- No dynamic persona generation
- No visual/screenshot analysis
- No product feedback from user perspective
- Tests chat interfaces only

---

## 2. Channel-Labs / synthetic-conversation-generation

**Repo**: https://github.com/Channel-Labs/synthetic-conversation-generation
**License**: MIT | **Language**: Python (~400 lines)

### 2.1 Architecture

Clean two-phase decoupled pipeline:

```
src/synthetic_conversation_generation/
  persona_generator.py          # Phase 1: Generate diverse personas
  conversation_generator.py     # Phase 2: Simulate conversations
  data_models/
    character_card.py           # Persona schema
    conversation.py             # Conversation + Message
    assistant.py                # Target system definition
    inference_endpoint.py       # Generic HTTP client for target
  llm_queries/
    user_persona_query.py       # Persona generation prompt + schema
    user_message_query.py       # User message generation prompt
    conversation_completion_query.py  # Natural stopping detection
```

### 2.2 Persona Data Structure (CharacterCard)

```python
@dataclass
class CharacterCard:
    name: str          # "Alex Rivera"
    description: str   # Physical/mental traits overview
    personality: str   # Personality description
    scenario: str      # Why they're interacting with the assistant
    summary: str       # ~10 word summary
```

Five string fields. The `scenario` field ties the persona to the target assistant's domain, ensuring contextual relevance.

### 2.3 Diversity Mechanism (Prompt-Driven Accumulation)

The core innovation — and limitation:

```python
for i in range(args.num_personas):
    persona = persona_generator.generate_persona()
    new_personas.append(persona)
    persona_generator.previous_personas.append(persona)  # accumulates
```

The diversity prompt (from `user_persona_query.py`):

```
Create a distinct, realistic, and well-defined user persona...

### Instructions
1. Review the assistant definition and previous user personas.
2. Invent a new persona...that is likely to seek out the defined assistant...
3. Develop the persona based on filling gaps in the existing persona collection.

### Previous User Personas
{json.dumps([asdict(persona) for persona in self.previous_personas], indent=4)}
```

**The entire list of all previously generated personas is dumped into the prompt as JSON.** The LLM is instructed to "fill gaps" in the existing collection.

**Scaling limitation**: Prompt grows linearly with persona count. Hits context limits at ~50-100 personas depending on verbosity.

**No formal diversity metric**, no embedding similarity check, no rejection sampling. Diversity is entirely LLM-compliance-dependent.

### 2.4 Conversation Generation

Three LLM calls per turn:

```python
for i in range(self.max_conversation_turns):
    user_message = user_message_generator.query()         # 1. LLM generates user msg
    conversation.messages.append(user_message)

    assistant_message = self.assistant_endpoint.get_assistant_message(conversation)  # 2. HTTP call to target
    conversation.messages.append(assistant_message)

    completion_checker = ConversationCompletionQuery(...)
    is_complete = completion_checker.query()               # 3. LLM decides if done
    if is_complete:
        break
```

User message generation emphasizes natural speech:
```
- Use natural human speech patterns (varied sentence length,
  occasional grammatical imperfections, contractions)
- Mimic human behavior when chatting with AI (concise, direct
  questions, sometimes abrupt topic changes)
```

### 2.5 Natural Stopping Point Detection

Uses a separate (potentially more expensive) model — default `o3` while user messages use `gpt-4o`.

Five evaluation criteria:
1. Has the primary user need been addressed satisfactorily?
2. Is the user frustrated or has the conversation reached a dead end?
3. Are there closure signals (gratitude, goodbyes)?
4. Does the conversation feel complete based on natural patterns?
5. Would a typical user naturally respond again?

### 2.6 Structured Output Pattern

For **OpenAI**: native JSON Schema (`response_format.type = "json_schema"`, `strict: True`).
For **Anthropic**: tool-use API as structured output — defines a fake tool `json_extractor` and forces the model to call it via `tool_choice`.

Temperature: `1.0` for generation. Seed: `42` for reproducibility (OpenAI).

### 2.7 Generic Inference Endpoint

```yaml
# endpoint config (YAML)
url: "https://api.example.com/chat"
body:
  model: "gpt-4o"
  messages: []
headers:
  Authorization: "Bearer ${API_KEY}"
response_path: "choices.0.message.content"
```

Supports `${ENV_VAR}` interpolation. The `response_path` extracts content from arbitrary JSON responses. This means the target system can be any HTTP API.

### 2.8 Reusable for CrowdTest

| Pattern | Applicability |
|---------|--------------|
| Decoupled persona → action pipeline | Generate personas, review, then run browser tests |
| YAML intermediate format | Human-editable persona files between phases |
| Accumulative diversity prompt | Baseline approach for <50 personas (augment with embeddings for more) |
| Natural stopping via expensive model | Detect when a simulated user would stop using the product |
| Generic HTTP endpoint abstraction | Configurable target product testing |
| Dual structured output (OpenAI/Anthropic) | Multi-provider LLM support |

### 2.9 Limitations

- No browser automation or UI testing
- No product feedback extraction
- No diversity metrics (relies solely on LLM compliance)
- No parallel execution (sequential generation)
- Linear context scaling caps at ~50-100 personas
- No evaluation framework

---

## 3. Tencent PersonaHub

**Repo**: https://github.com/tencent-ailab/persona-hub
**Paper**: "Scaling Synthetic Data Creation with 1,000,000,000 Personas" (2024)
**License**: CC-BY-NC-4.0 | **Language**: Python (minimal code)

### 3.1 What's Actually Open-Sourced

The repo is **not** the persona generation pipeline. It contains:

```
code/
  prompt_templates.py    # 4 downstream task templates
  vllm_synthesize.py     # Batch synthesis via vLLM
  openai_synthesize.py   # Synthesis via OpenAI API
data/
  persona.jsonl          # 200K sample personas
  instruction.jsonl      # 50K synthesized instructions
  math.jsonl             # 50K math problems
  ...
```

The actual Text-to-Persona pipeline, deduplication code, and elite filtering are **proprietary**. Only the methodology is described in the paper.

### 3.2 Persona Generation Method (Text-to-Persona)

**Source corpus**: RedPajama v2 (large-scale web text).

**Method**: Given a web text snippet, LLM infers "who is likely to [read | write | like | dislike] this text" and outputs a 1-2 sentence persona description.

**Key insight**: Personas are derived as *inferred readers/authors* of existing web content, not invented from scratch. This anchors each persona in real-world knowledge domains.

### 3.3 Persona Data Format

**Basic format** (200K sample):
```json
{"persona": "A Political Analyst specialized in El Salvador's political landscape."}
```

**Elite format** (370M on HuggingFace `proj-persona/PersonaHub`):
```json
{
  "persona": "A software developer looking for a way to simplify the integration of GPRS technology into embedded system designs...",
  "general domain (top 1 percent)": "Computer Science",
  "specific domain (top 1 percent)": "Embedded Systems",
  "general domain (top 0.1 percent)": null,
  "specific domain (top 0.1 percent)": null
}
```

Personas are **unstructured natural language strings** (1-2 sentences) focusing on knowledge, experience, interest, personality, and profession.

### 3.4 Deduplication at Billion Scale

Two-stage pipeline:

**Stage 1 — MinHash (surface-form)**:
- 1-gram tokenization
- Signature size: 128
- Jaccard similarity threshold: 0.9

**Stage 2 — Embedding-based (semantic)**:
- Model: OpenAI `text-embedding-3-small`
- Cosine similarity threshold: 0.9 (adjustable)

**Result**: ~1.016 billion unique personas after dedup + quality filters.

### 3.5 Elite Persona Filtering

- Each persona classified into general + specific domain categories
- Ranked as top 1% or top 0.1% within domain
- 370M out of ~1B survived the elite filter (~37%)
- Ranking mechanism likely uses LLM classifier (not disclosed)

### 3.6 Downstream Synthesis Templates

The `{persona}` injection pattern:

```python
# Instruction generation
instruction_template = """Guess a prompt that the following persona may ask
you to do: {persona}"""

# Knowledge article
knowledge_template = """Assume the persona and write a Quora article:
{persona}"""
```

Output schema for all tasks:
```json
{
  "input persona": "string",
  "synthesized text": "string",
  "description": "string"
}
```

**vLLM synthesis parameters**: Temperature 0.6, Top-p 0.95, Max tokens 2048, Tensor parallel across 4 GPUs.

### 3.7 Reusable for CrowdTest

| Pattern | Applicability |
|---------|--------------|
| Text-to-Persona from product pages | Feed product URL/docs → "who would use this?" → relevant personas |
| MinHash + embedding dedup | Two-tier dedup for ensuring persona diversity at scale |
| Domain classification as quality signal | Filter personas for product relevance |
| Template-driven `{persona}` injection | Clean extensible synthesis pattern |
| HuggingFace 370M elite personas | Pre-built persona pool for bootstrapping |

### 3.8 CrowdTest Persona Format Recommendation

PersonaHub's flat text string is too unstructured for browser automation. CrowdTest should use **structured JSON with explicit dimensions** while including a natural language summary:

```json
{
  "id": "persona_001",
  "name": "Maria Chen",
  "tech_level": "intermediate",
  "purpose": "price comparison",
  "age": 34,
  "industry": "healthcare",
  "personality": {
    "patience": 0.3,
    "exploration_tendency": 0.8,
    "attention_to_detail": 0.6
  },
  "summary": "A busy healthcare worker who shops online for deals but gets frustrated with slow sites",
  "narrative": "You are Maria Chen, a 34-year-old nurse who..."
}
```

### 3.9 Limitations

- No agent execution or testing
- No browser automation
- No structured feedback schemas
- No multi-turn product interaction
- No persona memory or return-user simulation
- Personas are text-only, not executable agents
- Core generation pipeline is proprietary

---

## 4. Playwright Test Agents

**Docs**: https://playwright.dev/docs/test-agents
**MCP Server**: `@playwright/mcp`
**Version**: v1.56+ | **Language**: TypeScript

### 4.1 Architecture: Tools, Not Brains

Playwright's AI testing has two layers:

**Layer A — MCP Server** (`@playwright/mcp`): Exposes browser control to any LLM via Model Context Protocol. The browser automation primitive.

**Layer B — Test Agents** (v1.56+): Three pre-built agent definitions (Planner, Generator, Healer) that sit on top of MCP. These are `.agent.md` files consumed by coding agents like Claude Code.

**Key insight**: Playwright does NOT embed an AI model. It exposes structured browser tools and relies on external LLMs for reasoning.

### 4.2 MCP Browser Tools (21 Core)

| Category | Tools |
|---|---|
| Navigation | `browser_navigate`, `browser_navigate_back`, `browser_tabs` |
| Interaction | `browser_click`, `browser_hover`, `browser_type`, `browser_press_key`, `browser_drag`, `browser_select_option`, `browser_file_upload`, `browser_handle_dialog` |
| Observation | `browser_snapshot`, `browser_console_messages`, `browser_network_requests`, `browser_take_screenshot` |
| Execution | `browser_evaluate`, `browser_run_code`, `browser_wait_for` |
| Management | `browser_close`, `browser_resize` |
| Optional | `browser_pdf_save`, `browser_mouse_move_xy`, `browser_mouse_click_xy` (vision mode) |

### 4.3 Accessibility Snapshots vs Screenshots

**Critical design decision**: The MCP server uses **structured accessibility snapshots** rather than pixel-based vision input. The LLM sees:

```
- textbox "What needs to be done?"
- checkbox "Toggle Todo"
- text "1 item left"
```

This is far more token-efficient and deterministic than screenshot-based approaches. Vision mode (`--caps=vision`) is optional.

### 4.4 Three Pre-Built Agents

**Planner** (Sonnet):
- Explores the app via browser tools
- Maps user journeys and critical paths
- Outputs structured markdown test plan to `specs/`

**Generator** (Sonnet):
- Takes a spec, executes each step with real browser tools
- Records what happens, then writes Playwright test code:

```typescript
test('Add Valid Todo', async ({ page }) => {
  const todoInput = page.getByRole('textbox', { name: 'What needs to be done?' });
  await todoInput.click();
  await todoInput.fill('Buy groceries');
  await todoInput.press('Enter');
  await expect(page.getByText('Buy groceries')).toBeVisible();
});
```

**Healer** (Sonnet):
- Runs full test suite, identifies failures
- Debugs each failure with snapshots and selector analysis
- Fixes tests or marks unfixable with `test.fixme()`

### 4.5 Configuration

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

Key flags: `--browser` (chromium/firefox/webkit), `--headless`, `--device "iPhone 15"`, `--viewport-size`, `--storage-state` (cookies/localStorage from JSON), `--init-page <path>`.

**Page initialization hook** (relevant for persona setup):
```typescript
export default async ({ page }) => {
  await page.context().grantPermissions(['geolocation']);
  await page.context().setGeolocation({ latitude: 37.7749, longitude: -122.4194 });
  await page.setViewportSize({ width: 1280, height: 720 });
};
```

### 4.6 Reusable for CrowdTest

| Pattern | Applicability |
|---------|--------------|
| MCP tool surface (21 browser tools) | Use `@playwright/mcp` directly as browser automation layer |
| Accessibility snapshots over screenshots | Token-efficient, deterministic page observation |
| `--init-page` hooks | Per-persona browser context setup (locale, geolocation, viewport) |
| `--device` emulation | Test mobile experiences per persona device preferences |
| `--storage-state` | Pre-authenticated sessions per persona |
| Intent classification on tool calls | Extend with persona-driven intent (exploratory, goal-directed, confused) |

### 4.7 What CrowdTest Should NOT Copy

- The Planner/Generator/Healer split is QA-focused, not user-simulation-focused
- The test-plan-to-code pipeline generates test scripts, not product feedback
- No persona concept whatsoever

---

## 5. Cross-Project Synthesis

### 5.1 Component Mapping for CrowdTest

```
CrowdTest Pipeline:
  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐    ┌──────────────┐
  │   Persona    │ →  │    Browser        │ →  │    Feedback       │ →  │    Report     │
  │   Generator  │    │    Automation     │    │    Evaluation     │    │    Output     │
  └──────┬───────┘    └────────┬─────────┘    └────────┬─────────┘    └──────┬───────┘
         │                     │                       │                     │
  PersonaHub method     Playwright MCP          ArkSim metrics        ArkSim HTML
  Channel-Labs dedup    Accessibility snap      Error dedup           report pattern
  Dimension matrix      Init-page hooks         LLM-as-judge
```

### 5.2 Recommended Architecture

**Phase 1 — Persona Generation** (borrow from PersonaHub + Channel-Labs):
- Dimension matrix: `tech_level × purpose × age × industry × personality`
- Product-aware: feed product URL → Text-to-Persona → relevant personas
- Diversity: accumulative prompt for <20, embedding cosine check for >20
- Output: structured JSON with both fields and narrative summary

**Phase 2 — Browser Testing** (borrow from Playwright):
- Use `@playwright/mcp` as the browser automation layer
- Accessibility snapshots for token-efficient page observation
- Per-persona init hooks (device, locale, viewport, auth state)
- Persona-driven exploration (not scripted test plans)

**Phase 3 — Feedback Evaluation** (borrow from ArkSim):
- Adapt quantitative metrics: usability, clarity, accessibility, performance (1-5)
- Adapt qualitative failure categories: broken UI, confusing flow, missing info
- LLM error deduplication across personas for unique product issues
- Session-level scoring: `overall_score = weighted(metric_scores)`

**Phase 4 — Reporting** (borrow from ArkSim):
- Standalone HTML report with embedded JSON data
- Per-persona drill-down + cross-persona aggregation
- Unique issue cards with severity and occurrence count
- CI/CD threshold gates

### 5.3 Gaps None of These Projects Fill

| Gap | Impact on CrowdTest |
|-----|-------------------|
| No product feedback schemas | CrowdTest must define usability, UX, accessibility metrics from scratch |
| No visual regression detection | Need screenshot comparison for UI bug detection |
| No return-user simulation | No project models memory or multi-session journeys |
| No A/B variant testing | CrowdTest opportunity: test two versions with same personas |
| No accessibility-specific personas | Model users with disabilities (screen reader, color blindness, motor impairment) |
| No cultural/locale-aware testing | Personas that test i18n, RTL layouts, date formats |

### 5.4 Competitive Positioning

```
                    Persona Generation
                         ▲
                         │
            PersonaHub ● │              ● CrowdTest (target)
           Channel-Labs ● │
                         │
  ◄──────────────────────┼──────────────────────► Browser Automation
                         │                 Playwright ●
                         │
              ArkSim ●   │
                         │
                         ▼
                    Evaluation/Feedback
```

CrowdTest occupies the intersection that no existing project covers: the top-right quadrant where persona generation meets browser automation with structured evaluation.
