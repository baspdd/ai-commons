# AI Commons

Host-neutral rules, skills, agent specifications, and MCP declarations shared by
Claude, Codex, and other coding agents.

This repository is the canonical source for shared AI behavior. Project-specific
architecture and guardrails must stay in each project's `.ai/` directory.

## Architecture

```text
ai-commons/                         # Shared repository
|-- rules/                          # Canonical engineering rules
|-- skills/                         # Canonical reusable skills
|-- agents/specs/                   # Host-neutral agent behavior
|-- manifests/                      # Host-neutral MCP declarations
|-- adapters/                       # Portable host adapter templates
`-- user-config/                    # Legacy migration templates

project/
|-- .ai/
|   |-- common/                     # Git submodule -> ai-commons
|   |-- PROJECT.md                  # Project architecture and guardrails
|   |-- agents/                     # Project-specific agent specifications
|   `-- skills/                     # Project-specific skills
|-- .agents/skills/                 # Codex project skill discovery
|-- .claude/                        # Claude adapter -> .ai
`-- .codex/                         # Codex adapter -> .ai
```

Dependency flow:

```text
.claude --+
.codex  --+--> .ai/PROJECT.md + .ai/agents + .ai/skills
.agents ---------------------------+
                                    v
                              .ai/common
                                    |
                                    v
                           ai-commons repository
```

Host adapters may point into `.ai`. Project-specific canonical content may point into
`.ai/common`. Shared canonical content must never point back into a host adapter or a
specific project.

## Ownership

- Edit engineering rules only under `rules/`.
- Edit reusable skill behavior only in canonical `SKILL.md` files.
- Edit shared agent behavior only under `agents/specs/`.
- Treat `adapters/`, `agents/*.md`, and `user-config/` as generated or compatibility
  surfaces.
- Do not store user profiles, drive letters, or machine-specific checkout paths.
- Do not copy project-specific architecture into this repository.

## Project Installation

Add this repository as a submodule:

```bash
git submodule add https://github.com/baspdd/ai-commons.git .ai/common
git commit -m "chore: add shared AI common submodule"
```

Clone a project with its common AI base:

```bash
git clone --recurse-submodules <project-url>
```

Initialize an existing clone:

```bash
git submodule update --init --recursive
```

Update the project to the latest common commit:

```bash
git submodule update --remote .ai/common
git add .ai/common
git commit -m "chore: update shared AI common"
```

The parent project records an exact `ai-commons` commit. Updating `main` in this
repository does not silently change existing projects.

## MCP

`manifests/mcp.json` is the canonical declaration. The Codex adapter translates it to:

- Codex: `adapters/codex/config.toml`

It uses `codegraph serve --mcp` from `PATH`; no user-specific executable path is stored.
