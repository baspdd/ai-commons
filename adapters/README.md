# Host Adapter Templates

These files document how each host maps the canonical base. They are templates, not a
runtime installation and not a second source of truth.

When generating a project adapter, prefix canonical paths with `.ai/common/`. For example:

- `rules/RULE_ROUTER.md` becomes `.ai/common/rules/RULE_ROUTER.md`
- `skills/core/reasoning/SKILL.md` becomes `.ai/common/skills/core/reasoning/SKILL.md`
- `agents/specs/backend-reviewer.md` becomes
  `.ai/common/agents/specs/backend-reviewer.md`

Do not commit absolute checkout paths into generated adapters.
