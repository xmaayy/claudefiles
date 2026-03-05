---
name: explore-spec
description: "Use this when the user wants to explore options and design and implementation."
---

# Brainstorming Ideas Into Designs

## Overview

Help turn ideas into fully formed designs and specs through natural collaborative dialogue.

Start by understanding the current project context, then ask questions one at a time to refine the idea. Once you understand what you're building, present the design in small sections (200-300 words), checking after each section whether it looks right so far.

## Key Principles
- **No implementation code in designs** - Describe what to build, not how. Function signatures and types are fine. Method bodies, loops, and orchestration logic are not.
- **One question at a time** - Don't overwhelm with multiple questions
- **Multiple choice preferred** - Easier to answer than open-ended when possible
- **YAGNI ruthlessly** - Remove unnecessary features from all designs
- **Explore alternatives** - Always propose 2-3 approaches before settling
- **Incremental validation** - Present design in sections, validate each
- **Be flexible** - Go back and clarify when something doesn't make sense
- **Reference, don't restate** - If something already exists in the codebase, point to the file path. Don't describe how it works.
- **Name for actual scope** - Don't name things after one use case when the contract is broader (e.g. `file_url` not `presigned_s3_url` if any HTTP URL works).

## The Process

**Understanding the idea:**
- Check out the current project state first (files, docs, recent commits). Take your time exploring, the more you understand the state of the project the better.
- Dont be afraid to ask clarification about implementation details or intent of existing code.
- Ask questions one at a time to refine the idea.
- Prefer multiple choice questions when possible, but open-ended is fine too. Prefer the answer that aligns with the existing implementation.
- Only one question per message - if a topic needs more exploration, break it into multiple questions but give the context of your question.
- Focus on understanding: purpose, constraints, success criteria

**Exploring approaches:**
- Propose 2-3 different approaches with trade-offs
- Present options conversationally with your recommendation and reasoning
- Lead with your recommended option and explain why
- When proposing where code lives, check it against the project's architecture decision tree (CLAUDE.md) first.

**Presenting the design:**
- Once you believe you understand what you're building, present the design
- Break it into sections of 200-300 words
- Ask after each section whether it looks right so far
- Cover: architecture, components, data flow, error handling, testing
- Be ready to go back and clarify if something doesn't make sense

## After the Design

**Finalization:**
- Make use of ascii (for simple things) or graphviz (for complicated things)
  - Graphs can convey complex logic more comprehensively and in less space than equivalent words
  - Dont overuse diagrams though
- The following can be expressed in code (declarations only, no implementations):
    - Database column/table definitions
    - Function signatures (name, params, return type)
    - Types and schemas
    - API route paths and parameters
    - Protocol/interface surfaces between services
  - Everything else is described in plain language. No method bodies, no loops, no orchestration logic. If you find yourself writing `for`, `if`, `await`, or `return` inside a function
  body, stop and describe the behaviour instead.
  - When referencing existing code, give the file path. Don't re-explain what it does.
- Functions, tests, etc should be conveyed through descriptions and cases
  - You should be able to describe the intention thoroughly in plain language.
  - Explain the what and why without giving an exact code implementation
  - Allow the implementer to implement rather than just copying code like a monkey

**Key Behaviours / Acceptance Scenarios:**
  - Include testable behaviours scenarios that verify the feature works end-to-end
  - Scenarios must be verifiable through the public API (HTTP requests, polling, response assertions)
  - Do not include scenarios for internal behaviour (worker internals, event emission, storage layout). Those are unit/integration test concerns.
  - Do not include input validation scenarios. Those are unit tests.
  - Do not include non-scenarios ("existing test X already covers this" is not a scenario, just don't include it)
  - Use concrete test fixtures with real URLs and known content, not placeholder variables like {canary}

**Documentation:**
- Write the validated design to `docs/plans/YYYY-MM-DD-<topic>-design.md`
- Use use `~/.claude/skills/explore-spec/elements-of-style.md` as a style guide for your writing.
