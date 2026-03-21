---
layout:      post
title:       "Agent Skills: The Infrastructure Layer Your AI Stack Is Missing"
description: "A deep dive into Anthropic's new Agent Skills open standard, explaining progressive disclosure, MCP integration, and why encoding organizational knowledge into skills is the next architectural frontier for enterprise AI."
date:        "2026-03-21"
author:      "Jamie Zhang"
image:       "/img/ai-01.jpg"
tags:        ["Agent Engineering", "Claude SDK", "Fintech AI", "Open Standard", "Staff Engineer", "MCP"]
categories:  ["AI"]
---

# Agent Skills: The Infrastructure Layer Your AI Stack Is Missing


---

## Preface

There is a moment in every mature engineering discipline when the conversation shifts from *"how do we make it work"* to *"how do we make it work reliably, at scale, across teams."* For agentic AI systems, that moment has arrived — and the answer being proposed by Anthropic, in partnership with the broader agent tooling ecosystem, is a deceptively simple but architecturally profound abstraction: the **Agent Skill**.

This is not about prompts. Prompts are atoms — necessary, but they don't scale. Skills are molecules: self-describing, composable, progressively disclosed bundles of domain knowledge, workflow logic, and executable code that any compliant agent can load on demand. If you've been in the agentic AI space for more than a year, you know the pain they're solving. You've written the same workflow prompt seventeen times across seventeen conversations. You've watched junior engineers reinvent analytical frameworks that already exist. You've shipped agents that are brilliant in demos and brittle in production because there was no systematic way to encode *how your organization does things*.

Skills address all three of those failures simultaneously. And given that the specification is now an **open standard** — adopted across Claude, Codex, Gemini CLI, OpenCode, and more — the window for building Skills-native tooling is now, not later.

---

## Section I — From Concept to Anatomy: What a Skill Actually Is

The first thing to understand about Agent Skills is their deliberate structural simplicity. A skill is a **folder**. That's the whole physical reality of it. Inside that folder lives a mandatory `SKILL.md` file, and optionally three subdirectories: `references/` for additional markdown documentation, `scripts/` for executable code, and `assets/` for templates, images, schemas, and data files.

### Skill Folder Structure

```
my-skill/                          ← root folder
├── SKILL.md                       ← required
├── references/                    ← optional
│   └── domain-rules.md
├── scripts/                       ← optional
│   ├── analyze.py
│   └── visualize.py
└── assets/                        ← optional
    ├── output-template.md
    └── schema.json
```

The `SKILL.md` begins with YAML front matter containing two required fields: `name` and `description`. The name follows lowercase-with-hyphens convention (e.g., `analyzing-marketing-campaign`), and the description is the agent's primary decision signal — it must explain both *what* the skill does and *when* to invoke it. Everything else in the file is free-form markdown: step-by-step instructions, edge case handling, acceptable inputs, expected outputs, and references to external files.

### Minimal Valid SKILL.md

```yaml
---
name: analyzing-marketing-campaign
description: >
  Performs weekly campaign performance analysis from marketing data.
  Use when analyzing campaign metrics, funnel performance, efficiency
  KPIs, or budget reallocation decisions.
---

## Input Requirements
Accept data from: CSV upload OR BigQuery (mcp__bigquery__query)
Required columns: date, campaign_name, impressions, clicks, conversions, cost

## Workflow
1. Data Quality Check — validate schema, flag anomalies
2. Funnel Analysis — CTR, CVR vs. benchmark targets
3. Efficiency Metrics — ROAS, CPA, net profit
4. Budget Reallocation — ONLY if user asks;
   read references/budget_reallocation_rules.md
```

What makes this deceptively powerful is the conditional loading in step 4. The budget rules file might be 5,000 tokens. They never enter the context window unless the user asks about reallocation. That discipline, multiplied across hundreds of skills in an enterprise deployment, is the difference between an agent that stays sharp across a 20-turn conversation and one that degrades into hallucination by turn 8.

> **"If you find yourself typing the same prompt across conversations, you should consider transforming that into a skill. The context window is a public good — treat it like one."**

---

## Section II — Progressive Disclosure: The Architectural Insight That Changes Everything

The concept of progressive disclosure deserves its own section because it's the insight that separates Skill design from naive prompt engineering. Most practitioners building agent systems eventually discover that loading everything into context upfront is self-defeating. Context windows have hard limits, and — more critically — there's a well-documented phenomenon of **context degradation**: as the window fills, models begin to lose coherence on earlier material.

The Skill architecture addresses this with a three-tier loading model:

| Tier               | What Loads                  | When                                                | Cost                          |
|:-------------------|:----------------------------|:----------------------------------------------------|:------------------------------|
| **1 — Always**     | Name + Description only     | Every conversation                                  | ~20–40 tokens per skill       |
| **2 — On Trigger** | Full `SKILL.md`             | When a user request matches the skill's description | ~500–2,000 tokens             |
| **3 — On Demand**  | References, scripts, assets | Only when the skill's instructions call for them    | Variable, loaded once         |
| **Execution**      | Script outputs (not source) | When scripts run via bash/code execution            | Results only, not source code |

This architecture enables something previously impossible to guarantee: you can have a library of **100+ specialized skills** available to an agent without meaningfully degrading its context budget for any given conversation. The agent carries a lightweight routing index, not the encyclopedia itself.

For 100 skills with 2,000-token average detailed instructions, loading everything upfront costs ~200,000 tokens per conversation. With progressive disclosure, any given conversation uses perhaps 3–5 active skills — a cost of 6,000–10,000 tokens. **The savings are real, and they compound.**

> **Editor's Insight:** Progressive disclosure is not just a performance optimization — it's an architectural forcing function. It makes you think carefully about what belongs in `SKILL.md` versus a reference file versus a script. That discipline produces better-organized skills and better-behaved agents.

---

## Section III — Ecosystem Position: Skills, MCP, Tools, and Subagents

Understanding where Skills fit in the broader agent architecture is essential for designing systems that compose correctly.

| Component       | Primary Role                         | Lives In             | Best For                                                    |
|:----------------|:-------------------------------------|:---------------------|:------------------------------------------------------------|
| **Tools**       | Low-level capability primitives      | Always in context    | File I/O, bash, web search, function calls                  |
| **MCP Servers** | External data & system access        | Connected at runtime | BigQuery, Notion, Salesforce, Google Drive                  |
| **Skills**      | Domain expertise & workflow encoding | Progressively loaded | Repeatable, organization-specific processes                 |
| **Subagents**   | Isolated execution & parallelism     | Spawned on demand    | Parallel tasks, context isolation, fine-grained permissions |

The analogy from the course is apt: tools are a hammer and a saw; a skill is knowledge of how to build a bookshelf. MCP brings the lumber yard's inventory system. Subagents are the specialized apprentices who can each work independently on different shelves simultaneously. None of these components competes with the others — they're layers in a stack, each solving a distinct problem.

### The Subagent × Skill Pattern

A critical implementation detail: **subagents do not inherit skills from parent agents.** This is intentional isolation. When you dispatch a code-reviewer subagent, you must explicitly declare which skills it has access to in its agent definition. This forces you to think carefully about what each agent actually needs — a forcing function against the lazy pattern of loading everything everywhere.

```yaml
# .claude/agents/code-reviewer.md (agent definition)
name: code-reviewer
description: Reviews code for quality, security, and conventions
tools: [Bash, Glob, Grep, Read]
skills:
  - reviewing-cli-command   # ← explicitly declared, not inherited
```

> **Editor's Insight:** The subagent × skill pattern is where I expect the most competitive differentiation to emerge in enterprise AI tooling. Teams that build well-curated, well-evaluated skill libraries — and deliberately assign them to specialized subagents — will achieve consistency and auditability that prompt-only architectures simply cannot match.

---

## Section IV — Best Practices: Writing Skills That Work in Production

### 1. Names and Descriptions Are Your Routing Logic

The description field is, functionally, a **semantic router**. It must answer two questions simultaneously: what does this skill do, and when should an agent trigger it? Include domain keywords, trigger phrases, and explicit conditions.

- ❌ **Weak:** `"analyzes data"`
- ✅ **Strong:** `"performs weekly campaign performance analysis; use when analyzing marketing metrics, funnel performance, ROAS, CPA, or budget reallocation decisions"`

### 2. Keep SKILL.md Under 500 Lines

This constraint forces decomposition. If your skill grows past 500 lines, you're encoding too much in one place. Extract detailed specifications into `references/`, move executable logic into `scripts/`, and move output templates into `assets/`. The `SKILL.md` should read like a well-structured runbook, not a novel.

### 3. Be Explicit About Step Order and Skip Conditions

Non-determinism is the enemy of production agents. Every workflow step should be numbered, and any step that might be skipped should explicitly state the skip condition.

```markdown
## Workflow
1. Validate input schema — fail fast with clear error if invalid
2. Run diagnostic script: `python scripts/diagnose.py {input_file}`
3. Generate visualizations — ONLY if user requests plots
4. Read summary.txt output and present findings
5. Create Word document — use docx skill; reference assets/report-template.md
```

### 4. Use Positive and Negative Examples for Code Patterns

When encoding coding conventions, include both the pattern you want and the anti-pattern you're rejecting:

```markdown
## Type Annotation Convention
✅ USE this (modern Annotated syntax):
    def add(name: Annotated[str, typer.Argument(help="Task name")]) -> None:

❌ NOT this (older decorator pattern):
    @app.command()
    def add(name: str = typer.Argument(..., help="Task name")) -> None:
```

### 5. Validate with the Skill Creator

Anthropic ships a meta-skill — `skill-creator` — that evaluates your custom skills against best practices. Running new skills through this evaluator before deployment is the equivalent of linting your code before committing.

### 6. Write Evaluation Tests

Treat skills like software. Define expected behaviors and test them:

```markdown
## Test Matrix: generating-practice-questions skill

Query: "Generate questions and save to markdown"
Expected:
  - Reads SKILL.md first (progressive disclosure)
  - Loads markdown_template.md from assets/ (not LaTeX template)
  - Generates questions in order: True/False → Explanatory → Coding → Application
  - Saves to specified filename with correct markdown formatting
  - Does NOT load LaTeX template (wrong format for this query)
```

### Production Deployment Checklist

- [ ] YAML front matter validates (name, description present; lowercase-hyphen format)
- [ ] Description answers both "what" and "when" — readable as a routing rule
- [ ] `SKILL.md` under 500 lines; large content externalized to `references/`
- [ ] Workflow steps are numbered; skip conditions explicit
- [ ] Scripts have error handling and documented dependencies
- [ ] File paths use forward slashes (cross-platform requirement)
- [ ] Evaluated against `skill-creator` best practices checklist
- [ ] Unit tests written for representative query × expected behavior pairs
- [ ] Human-reviewed output for at least 5 representative inputs
- [ ] Tested across all target model versions (Sonnet, Opus as applicable)

---

## Section V — The Real Benefits: Why Skills Are Strategic Assets

### 1. Organizational Knowledge Becomes Infrastructure

The most underappreciated benefit of Skills is that they turn tacit organizational knowledge into **explicit, versioned, auditable infrastructure**. Your best senior analyst's mental model for evaluating marketing campaign efficiency can be encoded in a skill, reviewed, iterated on, and deployed consistently across every analyst using the platform — including the junior one who started last month.

This is knowledge transfer at scale, with none of the fidelity loss that normally attends it. When the senior analyst leaves, the skill remains.

### 2. Portability Across the Entire Agent Ecosystem

Because Agent Skills are an open standard, a skill you author today for Claude AI is directly usable in Claude Code, the Claude Agent SDK, Codex, Gemini CLI, and more. Your investment in skill authorship isn't vendor-locked — it's **portable capital**.

### 3. Predictability in a Non-Deterministic System

Agents without skills are inherently variable — the same prompt produces meaningfully different outputs across sessions, models, and context lengths. Skills introduce the functional equivalent of unit tests for agent behavior: defined inputs, defined workflows, inspectable outputs.

You can still get variability in *how* the output is expressed, but the *structure* of the analysis, the steps performed, the metrics computed — these become **reproducible**.

### 4. Context Efficiency Compounds

In enterprise deployments where token costs matter, the efficiency gains from progressive disclosure compound quickly:

| Scenario                                        | Tokens per Conversation |
|:------------------------------------------------|:------------------------|
| 100 skills, all loaded upfront                  | ~200,000 tokens         |
| 100 skills, progressive disclosure (3–5 active) | ~6,000–10,000 tokens    |
| **Efficiency gain**                             | **~95% reduction**      |

### 5. The Composability Multiplier

Individual skills combine into compound workflows. A well-designed skill library lets you build new capabilities by composing existing ones:

```
analyzing-marketing-campaign
    + brand-guidelines
    + pptx (built-in)
= Automated weekly marketing deck with brand-consistent styling, 
  data-driven insights, and budget reallocation recommendations
```

Each skill does one thing well. Together, they do something neither could do alone.

---

## Section VII — Impact on the AI Staff Engineer's Career Arc

For AI Staff Engineers — and those on the path to that role — the emergence of Agent Skills represents both an opportunity and a clarifying signal. Let me be direct.

### Skills Elevate What "Engineering" Means in Agent Systems

The early phase of agentic AI was dominated by prompt engineering — a craft that, while real, had a relatively low ceiling and a short half-life. Skills shift the locus of value toward **systems thinking, organizational knowledge architecture, and software engineering discipline**.

Designing a skill library for a financial institution requires the same intellectual muscles as designing a microservice architecture: thinking about interface boundaries, separation of concerns, composability, versioning, and failure modes. These are Staff Engineer problems, not junior developer problems.

### The Competency Stack for AI Staff Engineers

```
Level 5 — Organizational Change
  Translating domain expert knowledge into skill specifications
  Managing human workflows around skill authorship and review

Level 4 — Cross-Ecosystem Architecture
  Authoring portable skills across Claude, Codex, Gemini CLI
  Designing skills-as-organizational-assets strategy

Level 3 — Evaluation Engineering  ← Wide open gap
  Writing behavioral unit tests for agent workflows
  Human feedback integration, regression detection across model versions
  Building evaluation harnesses with quantitative metrics

Level 2 — Agent System Design
  Main agent / subagent topology design
  Permission modeling and context budget management
  Failure isolation and graceful degradation patterns

Level 1 — Skill Architecture  ← Entry requirement
  Skill library design, naming conventions, progressive disclosure
  Composability across multi-skill workflows
  MCP integration for enterprise data access
```

### The Evaluation Engineering Gap Is Wide Open

Running skills through `skill-creator` checks structural best practices, but it doesn't tell you whether the skill's analytical output is correct for your domain. Writing evaluation harnesses that test skills against expected behaviors — across model versions, across edge cases, with human feedback in the loop — is currently a problem with **no standardized solution**.

Staff Engineers who build credible capability here will be extremely valuable. This is the 2026 equivalent of being the person who figured out how to write good integration tests before the rest of the industry caught up.

### The Domain Expert Translation Role

The best skills are not written by engineers alone. They require deep collaboration with domain experts: the senior compliance officer who knows exactly how KYC decisions should be made, the head of treasury who understands the firm's hedging philosophy, the lead data scientist with a decade of time series experience.

Staff Engineers who can **bridge the gap between domain expertise and precise skill specification** are rare and disproportionately valuable. This is a genuinely new competency that doesn't map cleanly onto any prior engineering role.

### A Note on Career Risk

AI Staff Engineers who do not develop fluency in agentic system design — skills, subagents, MCP, evaluation — risk finding their expertise in standalone LLM integration become commoditized. The trajectory is clear:

- **Commoditizing:** Calling an LLM API, basic prompt engineering, single-turn chat integrations
- **Appreciating:** Composable agent system design, skill library architecture, evaluation engineering, cross-ecosystem portability

The economic value is shifting from "knowing how to call an API" to "knowing how to build reliable, auditable, composable agent systems." Engage with that shift now, not when it has become table stakes.

---

## Closing — The Open Standard Bet: Why Now Is the Right Time

The decision to make Agent Skills an open standard is strategically significant beyond the technical architecture. It signals that the industry is converging on a shared abstraction layer for agent capabilities — analogous to how REST became the lingua franca for service APIs and containers became the deployment primitive for cloud-native applications.

The teams and organizations that invest in skill library development now are not just solving today's problems. They're building **assets that appreciate in value as the ecosystem grows** — because any skill you write today will be usable in agent runtimes that don't yet exist. The portability is structural, not incidental.

For agentic AI professionals, the message is clear:
1. Understand the skill architecture deeply
2. Develop a point of view on how your organization should structure its skill library
3. Start building — and start evaluating
4. Own the evaluation problem; it's the unsolved piece

The organizations that figure out evaluation will be the ones that can trust their agents at scale.

**That trust is the whole game.**

---

## References

| Resource               | Location                                             |
|:-----------------------|:-----------------------------------------------------|
| Course: "Agent Skills" | Anthropic & DeepLearning.AI, taught by Elie Schoppik |
| Open Skills Standard   | github.com/anthropic/skills                          |
| Claude Agent SDK       | `pip install claude-agent-sdk`                       |
| Model Context Protocol | anthropic.com/mcp                                    |
| Anthropic Messages API | docs.anthropic.com — Code Execution Tool, Files API  |

