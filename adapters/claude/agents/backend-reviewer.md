---
name: backend-reviewer
description: Reviews C# / ABP / EF Core code against backend layering, contracts, data access, tenant behavior, and entity audit shape. Use after backend edits or before merge.
tools: Read, Grep, Glob, Bash
model: sonnet
---

# Backend Reviewer Adapter

Read and follow the canonical host-neutral specification:

`.ai/common/agents/specs/backend-reviewer.md`

Treat that specification as the complete review procedure. Load every rule file it names.

