# agent-platform-skills

Skills for wiring agents together on **Gemini Enterprise Agent Platform** — the parts
that are easy to get wrong and slow to diagnose: which registration mode can render
A2UI, what authenticates on each path, how Agent Identity changes outbound calls, and
how to tell a rejected credential from a missing permission.

Derived from running deployments. Where a claim was verified by observation rather
than taken from documentation, the skill marks it; where documentation and observed
behaviour disagree, both are recorded.

## Skills

| Skill | Use it for |
| :--- | :--- |
| [`agent-platform-architecture`](skills/agent-platform-architecture/SKILL.md) | **Designing.** The platform constraints that govern how agents fit together, and the four architectures they permit. One agent or several, Agent Identity or not, whether you need Agent Gateway, how to expose an agent in Gemini Enterprise. |
| [`agent-platform-a2a`](skills/agent-platform-a2a/SKILL.md) | **Debugging.** 401 vs 403, identifying the real caller from audit logs, Agent Identity principal forms, reading permissions out of errors, and the conditions under which a broken deployment reports success. |

The split is design versus operations, so that neither crowds out the other: the
architecture skill states the constraints and the designs they permit, and the
debugging skill covers failure modes and how to identify them.

## Install

Both installers read the `skills/` directory at the repo root.

```bash
# Claude Code (global)
npx -y skills add https://github.com/alanblythe/agent-platform-skills -y --agent claude-code -g

# Project scope instead of global — drop the -g and run from the project directory
npx -y skills add https://github.com/alanblythe/agent-platform-skills -y --agent claude-code

# Antigravity / Gemini CLI
npx -y skills add https://github.com/alanblythe/agent-platform-skills -y --agent antigravity -g
```

To use it without installing, point your agent at
[`skills/agent-platform-a2a/SKILL.md`](skills/agent-platform-a2a/SKILL.md) directly — it
is a single self-contained Markdown file.

## Layout

```
plugin.json               # Claude Code plugin manifest (duplicated in .claude-plugin/)
.claude-plugin/plugin.json
gemini-extension.json     # Antigravity / Gemini extension manifest
skills/
  README.md
  agent-platform-a2a/
    SKILL.md              # YAML frontmatter (name, description, metadata) + body
```

Each skill is a directory under `skills/` containing a `SKILL.md` whose frontmatter
carries `name` and a `description` written as trigger phrases — that description is what
an agent matches against when deciding whether the skill is relevant, so it names the
symptoms ("A2UI renders as raw JSON", "401 between agents") rather than summarising the
contents.

## Scope

Deliberately not covered: writing agent code, evaluation, and scaffolding. Those are
handled by the [`google-agents-cli`](https://github.com/google/agents-cli) skills, which
this complements rather than replaces — install both.

## License

Apache-2.0
