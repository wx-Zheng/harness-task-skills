# Anthropic Harness Methodology Reference

> Source: https://www.anthropic.com/engineering/harness-design-long-running-apps
>
> Implementation: See `SKILL.md` in this directory for the executable skill that implements this methodology.

## Core Problem

| Problem | Description |
|---------|-------------|
| Self-evaluation bias | Models confidently praise their own output regardless of quality |
| Context degradation | Long tasks cause coherence loss as context fills up |
| Quality ceiling | Simple prompt engineering hits performance limits |
| Scale complexity | Full apps need multi-stage coordination without human intervention |

## Architecture: Three-Agent System

```
User prompt → [Planner] → Full specification
                            ↓
                       [Generator] ⇄ [Evaluator]
                       (implements)    (tests + scores)
                            ↑              ↓
                        Fix iteration ← Eval results
```

## Design Principles

### 1. Separate Generation from Evaluation (strongest lever)
- Never let the same agent both generate and judge
- Split into independent Generator and Evaluator, each independently tunable
- Similar to GAN adversarial approach

### 2. Concrete Criteria Replace Subjective Judgment
- Don't ask "is this good?" — encode principles as scoreable dimensions
- Each dimension has clear indicators and evidence requirements

### 3. Structured Files as Handoff Protocol
- Agents communicate via files, not conversation
- Avoids lossy summarization, enables clean session boundaries

### 4. Task Decomposition + Strategic Guidance
- Break work into manageable sprints
- Maintain high-level direction without over-specifying implementation

## Sprint Contract Negotiation

Before each sprint:
1. Generator proposes implementation plan + acceptance criteria
2. Evaluator reviews feasibility
3. Both agree on testable outcomes
4. Prevents spec-implementation misalignment

> **Implementation note**: In the SKILL.md implementation, contract negotiation is handled by the orchestrator (step 3a-0) rather than by the Generator/Evaluator agents directly, since the skill uses subagents that cannot negotiate with each other. The orchestrator reviews the contract for feasibility and presents concerns to the user before launching the Generator.

## Context Management

| Strategy | Advantage | Cost |
|----------|-----------|------|
| Context reset | Clean start, prevents anxiety | Token cost, orchestration |
| Context compression | Maintains continuity | Doesn't fully eliminate issues |
| Detailed spec | Precise implementation guidance | Cascading errors if spec wrong |
| High-level spec | Execution flexibility | Needs contracts to bridge gaps |
| Multi-agent | Higher quality, specialized roles | 20x token cost |

## Key Insight

**Scaffold should be removed as models improve.** Don't over-engineer — continuously simplify.
