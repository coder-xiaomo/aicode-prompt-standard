# AI Code Editor Prompt Configuration Standardization Proposal

English | [简体中文](./README.zh-CN.md)

## Proposal Overview

Create a unified standard for AI code editor prompt configuration files to address the fragmentation caused by different editors using varied paths and formats.

## Current Problems

1. **Inconsistent Paths**:
   - Cursor: `.cursor/rules.md`
   - Trae: `.trae/rules/project_rules.md`
   - Codeium: `.codeium/context.md`
   - Other editors have their own conventions

2. **Configuration Synchronization Difficulties**: Team members using different editors need to maintain multiple copies of the same content
3. **Version Control Conflicts**: Configuration files from different editors need to be separately added to `.gitignore`
4. **High Migration Costs**: Switching editors requires reconfiguring prompts

## Proposed Standard

### Core Standard Path

```
.aicode/
├── prompts/
│   ├── project.md          # Project-level prompts (primary file)
│   ├── context.md          # Context rules
│   ├── style.md            # Code style guidelines
│   └── editor-specific/    # Editor-specific configurations (optional)
├── config.json             # Configuration file
└── README.md               # Documentation
```

### Configuration File Format (`config.json`)

```json
{
  "version": "1.0.0",
  "prompts": {
    "project": ".aicode/prompts/project.md",
    "context": ".aicode/prompts/context.md",
    "style": ".aicode/prompts/style.md"
  },
  "editorSpecific": {
    "cursor": {
      "rulesFile": ".cursor/rules.md",
      "source": ".aicode/prompts/project.md"
    },
    "trae": {
      "rulesFile": ".trae/rules/project_rules.md",
      "source": ".aicode/prompts/project.md"
    }
  }
}
```

## Implementation Path

### Phase 1: Compatibility Layer (Immediately Feasible)

Create conversion tools and installation scripts:

```bash
# install-aicode-prompts.sh
#!/bin/bash
mkdir -p .aicode/prompts

# Detect installed editors and create symbolic links
if [ -d ".cursor" ]; then
    ln -sf ../.aicode/prompts/project.md .cursor/rules.md
fi

if [ -d ".trae" ]; then
    mkdir -p .trae/rules
    ln -sf ../../.aicode/prompts/project.md .trae/rules/project_rules.md
fi
```

### Phase 2: Editor Adaptation

Encourage editor vendors to support standard paths:

```json
// Editor configuration example
{
  "promptSources": [
    ".aicode/prompts/project.md",  // Standard path (priority)
    ".cursor/rules.md"            // Legacy path (fallback)
  ]
}
```

### Phase 3: Ecosystem Development

1. Create validation tools to check configuration integrity
2. Develop configuration generation plugins for VS Code and other editors
3. Establish template libraries and best practices

## Detailed Configuration Specifications

### `project.md` Template

```markdown
# Project Prompts

## Project Overview
{{Project description}}

## Coding Standards
- Language: {{Programming language}}
- Style: {{Code style}}
- Constraints: {{Technical constraints}}

## Business Rules
{{Specific business logic}}
```

### Complete `config.json` Specification

```json
{
  "version": "string",
  "name": "string",
  "prompts": {
    "project": "string",
    "context": "string",
    "style": "string",
    "testing": "string"
  },
  "mergeStrategy": "overwrite|append|smart",
  "editorSupport": {
    "cursor": "boolean",
    "trae": "boolean",
    "codeium": "boolean"
  }
}
```

## Migration Guide

### Migrating from Existing Systems

1. **Incremental Migration**:
   ```bash
   # 1. Create standard directory
   mkdir -p .aicode/prompts

   # 2. Move existing configuration
   cp .cursor/rules.md .aicode/prompts/project.md

   # 3. Create symbolic links
   ln -s .aicode/prompts/project.md .cursor/rules.md
   ```

2. **Validation Tool**:
   ```bash
   npx aicode-validate
   ```

## Benefits Analysis

### For Developers

- ✅ **Consistency**: Unified experience across editors
- ✅ **Maintainability**: Single source of truth
- ✅ **Collaboration**: Standardized team configurations

### For Editor Vendors

- ✅ **Interoperability**: Reduce user migration costs
- ✅ **Ecosystem**: Promote toolchain integration
- ✅ **Standards Compliance**: Demonstrate professionalism and foresight

## Action Plan

1. **Immediate Actions** (1-2 weeks):
   - Create reference implementation and conversion tools
   - Write detailed documentation and examples
   - Establish community discussion groups

2. **Short-term Goals** (1-3 months):
   - Gain support from 2-3 mainstream editors
   - Create configuration templates for popular projects
   - Develop IDE plugins

3. **Long-term Vision** (6-12 months):
   - Become a de facto standard
   - Establish certification programs
   - Extend support to more AI development tools

## Call for Participation

We invite participation from:
- **Editor Developers**: Implement standard support
- **Open Source Projects**: Early adoption of the standard
- **Community Members**: Provide feedback and contributions
- **Enterprise Users**: Share real-world requirements
