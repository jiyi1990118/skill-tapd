[English](README.md) | [中文](README.zh.md)

# Skill-TAPD

> This README is for skill maintainers and human readers — not the runtime entry point.
>
> - Skill runtime entry: [`skill.md`](./skill.md)
> - Modules: [`guides/`](./guides/) and [`reference/`](./reference/)
> - Runtime rules (workflows, MCP tools, state transitions) live in `skill.md` and are not repeated here.

This skill provides AI-powered TAPD project management capabilities: requirement analysis, sprint planning, bug triage, status transitions, and batch operations with MCP integration.

## When to Use

Use this skill when working with TAPD for:

- **Requirement analysis** — Extract content from TAPD URLs, parse prototype images, apply 6-dimensional analysis framework
- **Sprint planning** — Fetch iterations, analyze capacity, batch-assign stories
- **Bug triage** — Parse error screenshots, classify severity/priority, batch-update
- **Status transitions** — Forward-only state machine with developer validation
- **Batch operations** — Bulk update stories/bugs/tasks with confirmation workflow
- **Daily standups** — Generate progress reports from TAPD data
- **Comment management** — List and create comments on stories/bugs/tasks
- **Member lookup** — Find users and their assigned items

## Current Structure

```text
skill-tapd/
├── skill.md                        # Runtime entry: triggers, hard rules, workflows
├── README.md                       # Maintainer entry (this file)
├── README.zh.md                    # Chinese version
├── guides/
│   ├── requirement-analysis.md    # 6-dimensional analysis methodology
│   ├── vision-parser-quick.md     # Fast image parsing guide
│   ├── vision-parser.md           # Full 14-section vision parsing standard
│   ├── workflows.md               # Sprint planning, bug triage, daily reports
│   └── templates/                 # Ready-to-use templates (7 files)
└── reference/
    ├── mcp-tools.md               # Complete TAPD MCP tool documentation
    └── status-mapping.md          # Workspace-specific status transition rules
```

**Module loading**: `skill.md` references guides/ and reference/ by path. Claude loads them on demand when processing specific workflows.

## Key Features

- **MCP Integration** — Uses `@npm_xiyuan/mcp-tapd-radar` for authenticated TAPD API access
- **Vision Parser** — Extracts UI elements, text, layouts from prototypes and screenshots
- **State Transition Validation** — Forward-only flow with developer presence checks (`has_developer()`)
- **Batch Operations** — Efficient bulk updates with confirmation prompts
- **Token Efficiency** — Modular architecture saves ~31% tokens on average

## Requirements

- **MCP Server**: `@npm_xiyuan/mcp-tapd-radar` configured in Claude Desktop
- **TAPD Account**: Workspace ID and API credentials
- **Workspace ID**: Default `48801209` (configurable in skill.md)

## Maintenance Rules

When maintaining this skill:

- Keep `skill.md` as the entry point (triggers, hard rules, workflow index)
- Put detailed workflows and templates in `guides/`
- Put API reference and configuration in `reference/`
- If `skill.md` exceeds 1000 lines, consider moving sections to modules
- Update workflow index table (line 161-171) when adding new capabilities
- Keep vision parser standards in sync between `guides/vision-parser-quick.md` and `guides/vision-parser.md`
- Do not delete `guides/templates/` — they're referenced by `requirement-analysis.md`

## Module Descriptions

### skill.md (789 lines)
- Frontmatter: name, description, whenToUse
- Core principles: completeness, vision parsing, confirmation workflow
- State transition rules with `has_developer()` validation
- Workflow index table with module references
- MCP tool usage guidelines
- Error handling and fuzzy matching strategies

### guides/requirement-analysis.md
- 6-dimensional analysis framework (goal, users, scenarios, boundaries, value, acceptance)
- Story breakdown and task decomposition
- Acceptance criteria generation (Given/When/Then)

### guides/vision-parser-quick.md
- Fast 3-step image parsing guide (structure, content, tech hints)
- Optimized for quick prototype analysis

### guides/vision-parser.md
- Full 14-section vision parsing standard
- Used for detailed UI/UX specification extraction

### guides/workflows.md
- 13 workflow chapters (sprint planning, bug triage, daily reports, etc.)
- Step-by-step execution guides with MCP tool calls
- Confirmation prompts and error recovery

### reference/mcp-tools.md
- Complete TAPD MCP API tool documentation
- Parameter specs, return formats, usage examples

### reference/status-mapping.md
- Workspace-specific status sequences (e.g., 48801209: new→planned→developing→testing→done)
- Transition validation rules

## Common Pitfalls

- **Do not bypass `has_developer()` checks** — State transitions require validated developer assignments
- **Do not skip confirmation prompts** — All write operations (create/update/delete) require user approval
- **Do not use `tapd_get_*` for bulk operations** — Use `tapd_count_*` + `tapd_list_*` + batch tools instead
- **Do not ignore vision parser triggers** — Description with `<img>` tags or image URLs must invoke parser

## Current File Paths

```text
~/.claude/skills/skill-tapd/skill.md
~/.claude/skills/skill-tapd/guides/
~/.claude/skills/skill-tapd/reference/
```

## Related Projects

- [MCP TAPD Radar](https://github.com/jiyi1990118/mcp-tapd-radar) — The underlying MCP server
- [Skill Project Knowledge System](../skill-project-knowledge-system/) — Project knowledge base builder
