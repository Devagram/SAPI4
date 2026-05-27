# Project skills

Project-scoped skills for working on this repo via Claude Code. Each subdirectory contains a `SKILL.md` with a frontmatter `name`/`description` and a body Claude reads when the skill is invoked.

## Available

| Skill | Purpose |
|---|---|
| `sapi4-build` | Stage source to `C:\temp\SAPI4`, build C++ binaries + D server |
| `sapi4-run` | Start the D server + nginx, confirm listening |
| `sapi4-stop` | Stop the server + nginx |
| `sapi4-smoke-test` | End-to-end check: enumerate voices, synthesize a phrase, confirm WAV |

## Adding a skill

```
.claude/skills/<skill-name>/
└── SKILL.md
```

`SKILL.md` format:

```markdown
---
name: skill-name
description: One-line summary used by Claude to decide when to invoke
---

# Body
Markdown instructions, code blocks, decision trees, etc.
```

Keep skills focused on operational tasks specific to this project. Anything generic (running tests, code review, etc.) belongs in user-level skills, not here.

## Naming

Prefix project skills with `sapi4-` to avoid colliding with built-in skill names (`run`, `verify`, `init`, etc.). When Claude sees `sapi4-build` it knows to look here rather than guessing.
