# CLAUDE.md for claude-ops

## Project Overview

`claude-ops` is a meta-operational workspace for developing, refining, and managing AI-assisted workflows. This project serves as a laboratory for agentic system design—a place to experiment with prompt engineering, test autonomous workflows, and maintain reusable operational patterns that can be deployed across other projects.

## Core Intent

This is where we:
- **Develop agentic flows**: Design multi-step autonomous workflows that chain together prompts, tools, and decision-making logic
- **Refine operational agents**: Iterate on and improve existing AI assistants and automation patterns
- **Manage active plans**: Maintain living documentation of ongoing strategic initiatives and their execution patterns
- **Extract learnings**: Capture what works (and what doesn't) to build a knowledge base of effective AI collaboration patterns

## Working with Claude in this Project

### Context Architecture

When working in this repository, Claude should:

1. **Think in systems, not scripts**: Each "agent" or "flow" represents a repeatable pattern, not a one-off solution. Design for reusability and composition.

2. **Maintain operational memory**: Reference and build upon patterns established in previous sessions. Each refinement should acknowledge what came before and explain the evolution.

3. **Document decision rationale**: Every flow or agent should include:
   - The problem it solves
   - The workflow pattern it implements
   - Key decision points and their reasoning
   - Known limitations and improvement opportunities

4. **Favor incremental refinement**: Rather than large rewrites, make targeted improvements that preserve what works while addressing specific gaps.

### Prompt Engineering Principles

When developing or refining agents:

**Be explicit about reasoning patterns**
- Define when to use sequential vs. parallel thinking
- Specify confidence thresholds for decisions
- Clarify when to seek human input vs. proceed autonomously

**Structure for composability**
- Break complex flows into discrete, reusable components
- Define clear input/output contracts between stages
- Make dependencies explicit

**Optimize for iteration velocity**
- Prefer small, testable changes over monolithic updates
- Include validation criteria at each stage
- Design for fast feedback loops


### Interaction Model

**When starting a session:**
- Review active plans to understand current priorities
- Check recent changes to understand evolving patterns
- Ask clarifying questions about intent before proposing solutions

**During development:**
- Think step-by-step through implications
- Challenge assumptions constructively
- Suggest improvements beyond the immediate request when relevant
- Flag potential issues early

**When refining:**
- Start by understanding what's working well (preserve it)
- Identify specific gaps or friction points
- Propose targeted improvements with clear rationale
- Consider second-order effects on related flows

**For meta-work (improving this system itself):**
- Be especially critical and creative
- Question established patterns
- Propose experiments to test new approaches
- Think about scalability and maintainability

## Quality Standards

Good work in this repository:
- **Makes future work easier**: Creates leverage for subsequent tasks
- **Reduces cognitive load**: Simplifies complex operations into clear patterns
- **Fails gracefully**: Includes error handling and recovery paths
- **Documents decisions**: Explains not just what but why
- **Enables autonomy**: Reduces need for human intervention without sacrificing control


## Key Principles

1. **Clarity over cleverness**: Straightforward patterns beat clever hacks
2. **Documentation is design**: If it can't be clearly explained, it needs refinement
3. **Test in production**: Real-world usage reveals true effectiveness
4. **Iterate relentlessly**: First version is rarely the final version
5. **Build leverage**: Each improvement should compound

---

When in doubt, prioritize: What creates the most leverage for future work while maintaining clarity and reliability?