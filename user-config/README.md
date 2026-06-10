# Claude User Config Compatibility Archive

This folder contains legacy Claude adapter templates. It is retained for migration only;
new projects should use project-local `.claude` adapters that point to `.ai/common`.

Canonical content does not live here:

- Rules: `rules/`
- Skills: `skills/<name>/SKILL.md`
- Agent behavior: `agents/specs/`

Paths in these templates are relative to the common base and must be prefixed by the
installer. Do not copy the files directly to a user profile and do not persist an absolute
checkout path.

For other hosts use:

- Codex: `adapters/codex/`
