---
name: refactor-planner
description: Plans behavior-preserving refactors with caller tracing, risk analysis, and verification steps. Use before non-trivial refactors; this agent does not edit code.
tools: Read, Grep, Glob, Bash
model: sonnet
---

# Refactor Planner Adapter

Read and follow the canonical host-neutral specification:

`.ai/common/agents/specs/refactor-planner.md`

Treat that specification as the complete planning procedure. Load every rule file it names.

