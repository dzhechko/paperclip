# Product Development Org Chart Blueprint

## Overview

This document defines the optimal organizational structure for a Paperclip AI team
that handles three core product workflows:

1. **Product Vision** — competitive landscape, micro-trends, product image
2. **Go-to-Market (GTM)** — positioning, channels, launch plan
3. **MVP Implementation** — design, build, test, deploy

---

## Org Chart Structure

```
CEO (Product Strategist)
├── CMO (Go-to-Market Lead)
│   ├── Researcher #1 — Market & Competitive Intelligence
│   └── Researcher #2 — Trend Analysis & Customer Discovery
├── CTO (Technical Lead)
│   ├── Engineer #1 — Backend / API
│   ├── Engineer #2 — Frontend / UI
│   └── DevOps — Infrastructure & CI/CD
├── PM (Product Manager)
│   └── Designer — UX/UI Design
└── QA (Quality Assurance)
```

**Total: 10 agents**

---

## Task Decomposition

### Task 1: Product Vision & Competitive Landscape

**Owner:** CEO (delegates to CMO + Researchers)

```
1. Product Vision & Competitive Landscape          [CEO]
├── 1.1 Competitive Landscape Analysis             [Researcher #1]
│   ├── Identify direct competitors (5-10)
│   ├── Analyze features, pricing, positioning
│   ├── Map competitive gaps and opportunities
│   └── Deliver: competitive-matrix.md
├── 1.2 Micro-Trends & Market Signals              [Researcher #2]
│   ├── Scan industry reports, social media, forums
│   ├── Identify 3-5 actionable micro-trends
│   ├── Validate trends with data points
│   └── Deliver: trends-report.md
├── 1.3 Customer Problem Validation                [Researcher #2]
│   ├── Define target personas (2-3)
│   ├── Map jobs-to-be-done framework
│   ├── Identify unmet needs from competitor reviews
│   └── Deliver: customer-personas.md
├── 1.4 Product Vision Synthesis                   [CEO]
│   ├── Synthesize research into product thesis
│   ├── Define unique value proposition
│   ├── Draft product vision statement
│   └── Deliver: product-vision.md
└── 1.5 Vision Review & Approval                   [CEO → Board]
    └── Present vision for board approval
```

**Recommended agent prompts for research tasks:**

> **Researcher #1 prompt (capabilities):**
> "Deep competitive intelligence analyst. Uses systematic GOAP-style research:
> define goal state (complete competitive matrix), identify current state
> (unknown landscape), plan action sequence (identify → analyze → compare → report).
> Operates in PARANOID mode: triple-verify every claim, cross-reference multiple
> sources, flag low-confidence findings explicitly. Never assume — always validate."

> **Researcher #2 prompt (capabilities):**
> "Trend analyst and customer discovery researcher. Uses GOAP research in paranoid
> mode: systematically decomposes research goals into atomic verifiable steps.
> Cross-references minimum 3 independent sources per claim. Explicitly marks
> confidence levels (HIGH/MEDIUM/LOW) on every finding. Specializes in identifying
> weak signals and emerging patterns before they become mainstream."

---

### Task 2: Go-to-Market Strategy

**Owner:** CEO (delegates to CMO + PM)

```
2. Go-to-Market Strategy                           [CEO]
├── 2.1 Market Positioning                         [CMO]
│   ├── Define positioning statement
│   ├── Craft messaging framework (headline, subhead, proof points)
│   ├── Map positioning vs competitors (perceptual map)
│   └── Deliver: positioning.md
├── 2.2 Channel Strategy                           [CMO]
│   ├── Evaluate acquisition channels (organic, paid, partnerships)
│   ├── Prioritize by CAC / effort / timeline
│   ├── Define channel-specific tactics
│   └── Deliver: channel-strategy.md
├── 2.3 Pricing Strategy                           [CMO + Researcher #1]
│   ├── Analyze competitor pricing models
│   ├── Define pricing tiers and packaging
│   ├── Model unit economics (LTV/CAC)
│   └── Deliver: pricing-model.md
├── 2.4 Launch Plan                                [PM]
│   ├── Define launch phases (soft launch → beta → GA)
│   ├── Create launch checklist and timeline
│   ├── Define success metrics (KPIs)
│   └── Deliver: launch-plan.md
├── 2.5 Content & Collateral                       [CMO]
│   ├── Landing page copy
│   ├── Product demo script
│   ├── Email sequences
│   └── Deliver: content-assets/
└── 2.6 GTM Review & Approval                      [CEO → Board]
    └── Present GTM for board sign-off
```

> **CMO prompt (capabilities):**
> "Go-to-Market strategist. Uses enhanced problem-solving methodology:
> decompose GTM challenge into positioning × channels × pricing × timing.
> For each dimension, apply first-principles thinking — start from customer
> pain points, not industry conventions. Validate every assumption against
> competitive data from Researcher #1. Output actionable frameworks, not
> abstract strategies. Every deliverable must include measurable success criteria."

---

### Task 3: MVP Implementation

**Owner:** CEO (delegates to CTO + PM + QA)

```
3. MVP Implementation                              [CEO]
├── 3.1 Technical Architecture                     [CTO]
│   ├── Define tech stack based on product requirements
│   ├── Design system architecture (monolith/microservices)
│   ├── Define data model and API contracts
│   ├── Identify build-vs-buy decisions
│   └── Deliver: architecture.md + API spec
├── 3.2 UX/UI Design                               [Designer]
│   ├── Create user flows for core scenarios
│   ├── Design wireframes for key screens
│   ├── Define design system (colors, typography, components)
│   └── Deliver: design-system.md + wireframes
├── 3.3 Backend Development                        [Engineer #1]
│   ├── Set up project structure and tooling
│   ├── Implement data models and migrations
│   ├── Build API endpoints (REST/GraphQL)
│   ├── Integrate external services
│   └── Deliver: working backend with tests
├── 3.4 Frontend Development                       [Engineer #2]
│   ├── Set up frontend framework
│   ├── Implement core UI components
│   ├── Connect to backend APIs
│   ├── Implement responsive design
│   └── Deliver: working frontend with tests
├── 3.5 Infrastructure & CI/CD                     [DevOps]
│   ├── Set up development environment
│   ├── Configure CI/CD pipeline
│   ├── Set up staging and production environments
│   ├── Configure monitoring and logging
│   └── Deliver: deployment pipeline + runbook
├── 3.6 Integration Testing                        [QA]
│   ├── Define test plan and test cases
│   ├── Execute functional testing
│   ├── Performance and security baseline
│   ├── Bug reporting and verification
│   └── Deliver: test-report.md
├── 3.7 MVP Review                                 [CTO + PM]
│   ├── Technical review (code quality, architecture)
│   ├── Product review (feature completeness, UX)
│   └── Go/no-go decision
└── 3.8 MVP Launch                                 [DevOps + PM]
    ├── Deploy to production
    ├── Smoke tests
    ├── Enable monitoring
    └── Announce launch per GTM plan
```

> **CTO prompt (capabilities):**
> "Technical architect and engineering lead. Uses problem-solver enhanced
> methodology: for each technical decision, enumerate alternatives, evaluate
> trade-offs (complexity, scalability, time-to-market, team capability),
> select and document rationale. Code review all engineer output.
> Ensure architecture supports GTM timeline. Escalate blockers with
> proposed solutions, never just problems."

> **Engineer prompts (capabilities):**
> "Software engineer. Writes clean, tested, production-ready code.
> Follows the architecture decisions from CTO. Uses problem-solver approach
> for debugging: reproduce → isolate → hypothesize → fix → verify.
> Commits frequently with clear messages. Asks for clarification early
> rather than making assumptions. Every PR must include tests."

---

## Agent Configuration Reference

| Agent | Role | Adapter | Model | Budget (cents/mo) |
|-------|------|---------|-------|--------------------|
| CEO | `ceo` | `claude_local` | `claude-sonnet-4-6` | 15000 |
| CTO | `cto` | `claude_local` | `claude-sonnet-4-6` | 10000 |
| CMO | `cmo` | `claude_local` | `claude-sonnet-4-6` | 8000 |
| PM | `pm` | `claude_local` | `claude-sonnet-4-6` | 5000 |
| Researcher #1 | `researcher` | `claude_local` | `claude-sonnet-4-6` | 5000 |
| Researcher #2 | `researcher` | `claude_local` | `claude-sonnet-4-6` | 5000 |
| Engineer #1 | `engineer` | `claude_local` | `claude-sonnet-4-6` | 8000 |
| Engineer #2 | `engineer` | `claude_local` | `claude-sonnet-4-6` | 8000 |
| Designer | `designer` | `claude_local` | `claude-sonnet-4-6` | 4000 |
| DevOps | `devops` | `claude_local` | `claude-sonnet-4-6` | 5000 |
| QA | `qa` | `claude_local` | `claude-sonnet-4-6` | 4000 |

**Total monthly budget:** ~77,000 cents ($770) on API, or $0 on Claude Max subscription.

---

## Methodology Tags for Agent Prompts

### GOAP Research (Paranoid Mode)
Used by: Researcher #1, Researcher #2

```
Research methodology: Goal-Oriented Action Planning (GOAP) in PARANOID mode.
1. DEFINE goal state explicitly (what "done" looks like)
2. ASSESS current state (what we know vs don't know)
3. PLAN action sequence (ordered research steps)
4. EXECUTE each step with triple-verification:
   - Primary source
   - Independent corroboration
   - Counter-evidence check
5. CONFIDENCE TAG every finding: [HIGH] [MEDIUM] [LOW]
6. FLAG contradictions explicitly — never silently resolve them
7. DELIVER structured output with methodology notes
```

### Problem Solver Enhanced
Used by: CEO, CTO, CMO, PM, Engineers

```
Problem-solving methodology: Enhanced Systematic Decomposition.
1. FRAME the problem precisely (what, why, constraints, success criteria)
2. DECOMPOSE into independent sub-problems
3. For each sub-problem:
   a. Generate 2-3 alternative approaches
   b. Evaluate trade-offs (effort, risk, impact, reversibility)
   c. Select approach with explicit rationale
4. SYNTHESIZE sub-solutions into coherent whole
5. VALIDATE against original success criteria
6. IDENTIFY risks and mitigations
7. DELIVER with decision log and next actions
```

---

## How to Use This Blueprint

### Option A: CEO reads this file automatically
Set `instructionsFilePath` in CEO's adapterConfig to point to this file.
CEO will follow the org structure and decomposition patterns on every heartbeat.

### Option B: Create agents manually via API
Use the agent configuration table above to create each agent via:
```
POST /api/companies/{companyId}/agents
```
Then assign top-level tasks to CEO.

### Option C: Let CEO hire the team
Give CEO the bootstrap prompt:
```
"Read /doc/guides/product-org-chart-blueprint.md and hire the full team
described in the org chart. Use paperclip-create-agent skill for each hire."
```
CEO will create agent-hire requests for your approval.
