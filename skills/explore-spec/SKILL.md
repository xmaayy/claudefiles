---
name: explore-spec
description: "Use this when the user wants to explore options and design and implementation."
---

# Brainstorming Ideas Into Designs

## Overview

Help turn ideas into fully formed designs and specs through natural collaborative dialogue.

Start by understanding the current project context, then ask questions one at a time to refine the idea. Once you understand what you're building, present the design in small sections (200-300 words), checking after each section whether it looks right so far.

## Key Principles

- **One question at a time** - Don't overwhelm with multiple questions
- **Multiple choice preferred** - Easier to answer than open-ended when possible
- **YAGNI ruthlessly** - Remove unnecessary features from all designs
- **Explore alternatives** - Always propose 2-3 approaches before settling
- **Incremental validation** - Present design in sections, validate each
- **Be flexible** - Go back and clarify when something doesn't make sense

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
- The following can be expressed in code:
  - Database Tables / Queries
  - Function declarations
  - Types
  - API Routes and Parameters
  - Any surface between services
- Functions, tests, etc should be conveyed through descriptions and cases
  - You should be able to describe the intention thoroughly in plain language.
  - Explain the what and why without giving an exact code implementation
  - Allow the implementer to implement rather than just copying code like a monkey

**Documentation:**
- Write the validated design to `docs/plans/YYYY-MM-DD-<topic>-design.md`
- Use use `~/.claude/skills/explore-spec/elements-of-style.md` as a style guide for your writing.
