---
name: sql-reviewer
description: Reviews PostgreSQL queries, functions, and migrations against naming, entity audit shape, safety, and idempotency rules. Use after SQL edits or before merge.
tools: Read, Grep, Glob, Bash
model: sonnet
---

# SQL Reviewer Adapter

Read and follow the canonical host-neutral specification:

`agents/specs/sql-reviewer.md`

Treat that specification as the complete review procedure. Load every rule file it names.

