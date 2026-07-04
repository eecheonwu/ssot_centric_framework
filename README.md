# SSOT-Centric Agentic Software Engineering Framework

SSOT-Centric Agentic Software Engineering Framework\
Author: Emmanuel Echeonwu\
First Published: 2026\
https://doi.org/10.5281/zenodo.20745818

---

## 📚 Part of SCSE Research Programme

This repository is a **core implementation project** within the **SSOT-Centric Software Engineering (SCSE)** research programme.

**🔗 [Return to SCSE Central Research Hub](https://github.com/eecheonwu/scse-knowledge-base)** — The authoritative source for all SCSE research concepts, theory, and methodology.

For theoretical foundations and research context, see the [SCSE Research Handbook](https://github.com/eecheonwu/scse-knowledge-base).

---

## Overview

The SSOT-Centric Agentic Software Engineering Framework is an AI-native
software development lifecycle model that enables software engineering
artefacts, AI planning systems, coding agents, testing agents, and
continuous system evolution to operate from a shared system knowledge
foundation.

The framework introduces a Single Source of Truth (SSOT) as the central
intelligence layer that maintains alignment between requirements,
architecture decisions, implementation strategy, source code, testing,
and future system changes.

The framework separates:

- SSOT: System understanding and architectural truth

- Implementation Plan: Architecture-driven implementation strategy

- Task Plan: Execution-level engineering activities

- Source Code and Tests: Software implementation and validation

## Lifecycle

```text
Engineering Artefacts
(SRD, TDD, ADRs, C4, Technical Evaluation)

                |
                v

              SSOT
           (knowledge)

                |
                v

       Implementation Plan
   (Architecture-first approach)

                |
                v

          Task Plan
   (Execution implementation tasks)
(generated from implementation plan + SSOT)

                |
                v

        AI Coding Agents

                |
                v

       Source Code + Tests

                |
                v

       SSOT Synchronization
```

# Repository Structure

```text
ssot_centric_framework/

        ├── knowledge/
        │   └── SSOT containing System knowledge and architectural truth
        │
        ├── implementation_plan
        │   └── SSOT-derived implementation strategy
        │
        ├── task_plan
        │   └── SRD and TDD-derived execution tasks or from ssot-derived implementation plan
        │
        ├── src/
        │   └── Application source code
        │
        ├── tests/
        │   └── Test artefacts
        │
        └── priority.md
            └── AI agent implementation instructions
```

# Directory Responsibilities

## knowledge/

Contains the authoritative representation of the system.

Generated from:

- Software Requirements Document (SRD)

- Technical Design Document (TDD)

- Architecture Decision Records (ADRs)

- C4 diagrams

- Technical analysis

- Existing repository context

The SSOT maintains:

- Product knowledge

- Architecture knowledge

- System behaviour

- Engineering constraints

- Security knowledge

- Testing strategy

## implementation\_plan

The implementation plan is derived from the SSOT.

It defines:

- Architectural implementation strategy

- Development sequence

- Component evolution

- Dependencies

- Technical constraints

It answers:

> How should the system be built while preserving architectural intent?

## task\_plan

The task plan is derived from:

- Implementation plan

- SSOT

It defines:

- Coding tasks

- File changes

- Implementation steps

- Validation requirements

It answers:

> What specific engineering actions must be completed?

## src/

Contains the software implementation.

All changes must comply with:

- SSOT rules

- Implementation Plan

- Task Plan

## tests/

Contains:

- Unit tests

- Integration tests

- End-to-end tests

- Validation artefacts

# Priority of Truth

AI agents must follow this order:

1. SSOT
2. Implementation Plan
3. Task Plan
4. Existing Source Code
5. New Implementation Decisions

When conflicts occur:

- SSOT has highest authority.

- Implementation decisions must align with SSOT.

- Task execution must respect architecture.

- Changes affecting system behaviour must update SSOT.

# AI Coding Agent Context

The AI coding agent receives:

```text
/knowledge

implementation_plan.md

task_plan.md

priority.md
```

Workflow:

```text
Load System Context
        |
Review Implementation Strategy
        |
Review Execution Tasks
        |
Implement Changes
        |
Run Tests
        |
Synchronize SSOT
```

# Core Principles

## SSOT as System Memory

The SSOT is not only documentation. It is the continuously maintained
intelligence layer of the software system.

## Architecture Before Execution

AI agents should understand system intent before modifying
implementation details.

## Separation of Planning Responsibilities

Implementation Plan:

Defines strategic architecture-aware execution.

Task Plan:

Defines concrete engineering actions.

# Benefits

The framework provides:

- Better AI agent context awareness

- Reduced architectural drift

- Improved traceability

- More reliable autonomous development

- Continuous system knowledge evolution

# Future AI Agent Extensions

The framework supports specialized agents:

- Requirements Agent

- Architecture Agent

- Implementation Agent

- Testing Agent

- Security Agent

- Deployment Agent

- SSOT Synchronization Agent

# Citation

Echeonwu, E. C. (2026). SSOT-Centric Agentic Software Engineering Framework. Zenodo. https://doi.org/10.5281/zenodo.20745818

***
