---
name: creating-skills.md
description: Guide for creating properly structured Claude Code Skills following official naming conventions and best practices.
version: 1.0.0
author: Kombinacija
tags: [claude, skills, documentation]
---

# Creating Claude Code Skills

This guide explains how to create properly structured Claude Code Skills that can be discovered and used effectively by Claude Code.

## Directory Structure

```
.claude/skills/
└── skill-name/
    ├── SKILL.md           # Required: instruction file (case-sensitive)
    └── *.js             # Helper files
    └── *.md             # Documentation files
```

## Naming Conventions

### Directory Names
- **Format**: kebab-case (hyphen-separated lowercase words)
- **Examples**: `python-naming-standard`, `playwright-testing`, `cost-relations`
- **Max length**: 80 characters
- **No spaces**: Use hyphens, not spaces

### File Names

1. **Instruction File**: MUST be named `SKILL.md` (case-sensitive)
   - ❌ Invalid: `skill.md`, `Skill.md`, `SKILL.MD`
   - ✅ Valid: `SKILL.md`

2. **Skill Name** (used in `name` field of SKILL.md):
   - **Format**: lowercase, numbers, hyphens only
   - **Max length**: 64 characters
   - **No reserved words**: `anthropic`, `claude`
   - **Examples**: `playwright-testing`, `testing-code`, `processing-pdfs`

3. **Helper Files**:
   - **Naming**: kebab-case.js or PascalCase.js
   - **Examples**: `shared-code.js`, `helpers.js`, `date-utils.js`

## SKILL.md Front Matter

The `SKILL.md` file contains front matter with these required fields:

```yaml
---
name: Skill Name
description: Brief description of what the skill does and when to use it
version: 1.0.0
author: Your Name
tags: [category1, category2]
---
```

## Required Fields

### name
- **Type**: string
- **Required**: Yes
- **Description**: Human-readable skill name
- **Max length**: 64 characters
- **Allowed**: lowercase, numbers, hyphens
- **Forbidden**: spaces, underscores, reserved words

### description
- **Type**: string
- **Required**: Yes
- **Description**: What the skill teaches or provides
- **Max length**: 1024 characters
- **Allowed**: Can contain markdown, code blocks, links

### version
- **Type**: string
- **Required**: No
- **Description**: Skill version (semantic versioning like 1.0.0)
- **Format**: X.Y.Z (e.g., 1.0.0, 2.1.0)

### author
- **Type**: string
- **Required**: Yes
- **Description**: Who created this skill
- **Format**: Your name or organization

### tags
- **Type**: array of strings
- **Required**: No
- **Description**: Categories for discovery (e.g., [testing, playwright, documentation])

## Best Practices

1. **Be Specific**: Skills should solve a clear, well-defined problem
2. **Use Proper Naming**: Follow kebab-case convention
3. **Include Examples**: Show how to use the skill with code
4. **Document Dependencies**: Mention any other skills or files required
5. **Version Appropriately**: Use semantic versioning (X.Y.Z)
6. **Write Clear Front Matter**: Make SKILL.md metadata easy to understand

## Template

See `.claude/skills/creating-skills/SKILL.md` for a complete example of a properly structured skill.
