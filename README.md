# 🚀 Skill-TAPD

[![npm version](https://img.shields.io/npm/v/@npm_xiyuan/skill-tapd.svg)](https://www.npmjs.com/package/@npm_xiyuan/skill-tapd)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Skill](https://img.shields.io/badge/Claude-Skill-8A2BE2)](https://claude.ai)

> **Professional TAPD Project Management Skill for Claude AI** - Transform your TAPD workflow with AI-powered requirement analysis, sprint planning, and bug triage.

## ✨ Features

- 🎯 **Smart Requirement Analysis** - 6-dimensional requirement understanding with automated completeness checks
- 🖼️ **Vision Parser Integration** - Automatically extracts UI elements, text, and layouts from prototypes and screenshots
- 📋 **Sprint Planning Automation** - Capacity analysis and intelligent task allocation
- 🐛 **Intelligent Bug Triage** - Auto-classify severity/priority with screenshot analysis
- 📊 **Daily Standup Reports** - Generate progress reports and identify blockers
- ⚡ **Token-Efficient** - Modular architecture saves 31% tokens on average
- 🔧 **MCP Integration** - Built on Model Context Protocol for seamless TAPD API access

## 🎥 Quick Demo

```
You: "Help me analyze this requirement [with prototype image]"

Claude: 
1. 🖼️ Parses the prototype (UI elements, text, layout)
2. 🎯 Analyzes 6 dimensions (goal, users, scenarios, boundaries, value, acceptance)
3. 📝 Breaks down into Story → Tasks
4. ✅ Generates Given/When/Then acceptance criteria
5. 💾 Creates TAPD records with user confirmation
```

## 📦 Installation

### For Claude Desktop

```bash
# Install via npm
npm install -g @npm_xiyuan/skill-tapd

# Or clone the repository
git clone git@github.com:jiyi1990118/skill-tapd.git
cd skill-tapd
```

Add to your Claude configuration:

```json
{
  "skills": [
    {
      "name": "skill-tapd",
      "path": "/path/to/skill-tapd/skill.md"
    }
  ]
}
```

### For Claude Web

Upload `skill.md` to your Claude conversation to activate the skill.

## 🚀 Quick Start

### 1. Requirement Analysis (with Prototype)

```
"Analyze this requirement [attach prototype image]"
```

Claude will:
- Extract all UI elements from the prototype
- Understand business goals and user scenarios
- Check requirement completeness
- Break down into tasks
- Generate acceptance criteria

### 2. Sprint Planning

```
"Help me plan Sprint 5"
```

Claude will:
- Fetch open iterations and unassigned stories
- Analyze team capacity
- Recommend stories to include/exclude
- Highlight risks and dependencies

### 3. Bug Triage

```
"Triage this bug [with screenshot]"
```

Claude will:
- Extract error info from screenshot
- Analyze severity and priority
- Recommend owner based on module
- Generate triage report

## 📖 Documentation

- [Requirement Analysis Guide](guides/requirement-analysis.md) - 6-dimensional analysis methodology
- [Vision Parser](guides/vision-parser-quick.md) - Image parsing for prototypes and screenshots
- [Workflows](guides/workflows.md) - Sprint planning, bug triage, daily standups
- [Templates](guides/templates/) - Ready-to-use templates for stories, bugs, tasks
- [MCP Tools Reference](reference/mcp-tools.md) - Complete TAPD API tool documentation

## 🎯 Use Cases

### Product Managers
- 📝 Transform rough ideas into structured requirements
- 🖼️ Convert prototypes into detailed specs
- ✅ Generate comprehensive acceptance criteria

### Developers
- 📋 Break down stories into development tasks
- 🔍 Understand requirements with visual context
- 📊 Track sprint progress

### QA Engineers
- 🐛 Analyze bugs with screenshot context
- ✅ Generate test cases from requirements
- 📈 Monitor quality metrics

### Scrum Masters
- 🏃 Plan sprints with capacity analysis
- 📊 Generate daily standup reports
- ⚠️ Identify blockers and risks

## 🏗️ Architecture

```
skill-tapd/
├── skill.md                 # Main entry point (3.5KB)
├── guides/
│   ├── requirement-analysis.md    # 6D methodology
│   ├── vision-parser-quick.md     # Fast image parsing
│   ├── vision-parser.md           # Full 14-section standard
│   ├── workflows.md               # Sprint, triage, reports
│   └── templates/                 # 7 ready-to-use templates
└── reference/
    └── mcp-tools.md               # Complete API reference
```

**Token Efficiency**:
- Initial load: 600 tokens
- Requirement analysis: 2000 tokens (saves 23%)
- With image parsing: 2600 tokens (saves 37%)
- Using templates: 1400 tokens (saves 56%)

## 🔧 Requirements

- **Claude Desktop** or **Claude Web**
- **TAPD Account** with MCP server configured
- **MCP TAPD Server** (install via `@npm_xiyuan/mcp-tapd-radar`)

## 🌟 Why Skill-TAPD?

| Feature | Traditional | With Skill-TAPD |
|---------|-------------|-----------------|
| Requirement Analysis | Manual, inconsistent | AI-guided, 6D framework |
| Prototype Understanding | Manual annotation | Auto vision parsing |
| Sprint Planning | Spreadsheets | AI capacity analysis |
| Bug Triage | Manual classification | AI severity/priority detection |
| Documentation | Fragmented | Structured templates |

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

```bash
git clone git@github.com:jiyi1990118/skill-tapd.git
cd skill-tapd
# Make your changes
git commit -m "feat: your feature"
git push
```

## 📝 License

MIT © [jiyi1990118](https://github.com/jiyi1990118)

## 🔗 Related Projects

- [MCP TAPD Radar](https://github.com/jiyi1990118/mcp-tapd-radar) - The underlying MCP server
- [Claude Skills](https://docs.anthropic.com/claude/docs/skills) - Official Claude skills documentation

## 💬 Community & Support

- 🐛 [Report Issues](https://github.com/jiyi1990118/skill-tapd/issues)
- 💡 [Feature Requests](https://github.com/jiyi1990118/skill-tapd/issues/new)
- 📧 [Contact](https://github.com/jiyi1990118)

---

**Made with ❤️ for Product Managers, Developers, and Agile Teams**

⭐ Star us on GitHub if you find this helpful!
