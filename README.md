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
5. **Monorepo Challenges**: Different projects within a monorepo require different prompt configurations

## Proposed Standard

### Core Standard Path

```
.prompt/
  ├─ rules/
  │  ├─ project[/[<custom-filename>]][.<editor-identifer>].md     # Project-level prompts (primary file)
  │  ├─ context[/[<custom-filename>]][.<editor-identifer>].md     # Context rules (optional)
  │  └─ code_style[/[<custom-filename>]][.<editor-identifer>].md  # Code style guidelines (optional)
  ├─ config.json                            # Configuration file
  └─ README.md                              # Documentation (optional)
```

The `[.<editor-identifer>]` indicates editor-specific configurations. When omitted, it represents globally shared prompt files.

### Monorepo Support with Path-Specific Rules

For monorepo projects, configure path-specific rules using glob patterns in `.prompt-config.yaml`:

```yaml
version: 1.0

prompt:
  rules:
    - scope: "packages/frontend/**"  # Frontend packages
      files:
        - .prompt/frontend-rules.md
        - docs/frontend/ai-context.md

    - scope: "packages/backend/**"   # Backend packages
      files:
        - .prompt/backend-rules.md
        - "!docs/frontend/**"  # Exclude frontend docs

    - scope: "**/legacy/**"          # Any legacy directory
      files:
        - .prompt/legacy-overrides.md

  default:  # Default rules for unmatched paths
    files:
      - .prompt/general.md
      - docs/ai-guidelines.md
```

#### Rules Evaluation
1. **Specificity**: Exact paths > wildcard patterns > default rules
2. **Exclusion**: Negative patterns (`!path`) exclude files
3. **Proximity**: Closest `.promptconfig.yaml` wins in nested configurations
4. **Merge**: Arrays are merged, objects are deep-merged

### Configuration Examples

**Basic Configuration:**

If we want to configure project-level prompts, we can do this:

```
.prompt/
  └─ rules/
     ├─ project.md                        # Project-level prompts (primary file)
     ├─ project.vscode.md                 # Enhance `project.md` in VSCode editor (optional)
     ├─ project.cursor.md                 # Enhance `project.md` in Cursor editor (optional)
     └─ project.trae.md                   # Enhance `project.md` in TRAE editor (optional)
```

**Advanced Configuration (Multi-file Structure):**

When prompt content is extensive and needs to be split into multiple files, you can create corresponding directories to organize prompts according to custom directory structures. Similarly, files should end with `[.<editor-identifer>].md`. All files are optional.

```
.prompt/
  └─ rules/
     ├─ project/
     │  ├─ [foo].vscode.md
     │  ├─ [foo].cursor.md
     │  ├─ [foo].trae.md
     │  └─ index.md
     ├─ project.md
     ├─ project.vscode.md
     ├─ project.cursor.md
     └─ project.trae.md
```

In this example, `.prompt/rules/project/index.md` serves the same purpose as `.prompt/rules/project.md`. When both exist, they will both take effect.

For the VSCode editor, in this example, the following markdown files will be loaded as prompts:
- `.prompt/rules/project/[foo].vscode.md`
- `.prompt/rules/project/index.md`
- `.prompt/rules/project.md`
- `.prompt/rules/project.vscode.md`

### File Loading Rules

1. **Global Files**: `.prompt/rules/project.md` (shared by all editors)
2. **Editor-Specific Files**: `.prompt/rules/project.{editor}.md` (loaded by specific editors)
3. **Directory Structure**: Supports subdirectory organization, with `index.md` as the directory entry file
4. **Merge Strategy**: Content from all matching files will be merged, with editor-specific configurations extending global configurations
5. **Path-Specific Rules**: For monorepos, uses glob patterns to apply different prompts to different paths

## Configuration File Format (`.prompt/config.json`)

```json
{
  "promptFileVersion": "1.0",
  "response_language": "en-US", // Specify the language for model responses
  "defaultMergeStrategy": "append", // Default merge strategy: append|overwrite|smart
  "fileLoadingOrder": ["global", "editor-specific"], // File loading order
  "supportedEditors": ["vscode", "cursor", "trae", "codeium"],
  "editorSpecific": {
    "vscode": {
      "promptPriority": ["project.md", "project.vscode.md"],
      "maxTokenLimit": 4000
    },
    "cursor": {
      "promptPriority": ["project.md", "project.cursor.md"],
      "maxTokenLimit": 8000
    },
    "trae": {
      "promptPriority": ["project.md", "project.trae.md"],
      "maxTokenLimit": 6000
    },
    "[editor_name]": {
      ...
    }
  }
}
```

## Implementation Path

### Phase 1: Compatibility Layer (Immediately Feasible)

Create conversion tools and symbolic link scripts:

```bash
# migrate-prompts.sh
#!/bin/bash

# Create standard directory structure
mkdir -p .prompt/rules

# Migrate existing configurations and create symbolic links
migrate_editor_config() {
  local editor=$1
  local source_path=$2
  local target_name=$3

  if [ -f "$source_path" ]; then
    # Migrate to new location
    cp "$source_path" ".prompt/rules/${target_name}.${editor}.md"
    # Create symbolic link for backward compatibility
    ln -sf "../.prompt/rules/${target_name}.${editor}.md" "$source_path"
    echo "Migrated ${editor} configuration"
  fi
}

# Migrate various editor configurations
migrate_editor_config "cursor" ".cursor/rules.md" "project"
migrate_editor_config "trae" ".trae/rules/project_rules.md" "project"
migrate_editor_config "codeium" ".codeium/context.md" "context"

# Create basic configuration file
if [ ! -f ".prompt/config.json" ]; then
  cat > ".prompt/config.json" << EOF
{
  "promptFileVersion": "1.0",
  "response_language": "",
  "defaultMergeStrategy": "append",
  "supportedEditors": ["vscode", "cursor", "trae", "codeium"]
}
EOF
fi
```

### Phase 2: Editor Adaptation

Encourage editor vendors to natively support standard paths:

```json
// Editor configuration example (VSCode settings.json)
{
  "aiPrompts.standardPath": ".prompt/rules",
  "aiPrompts.loadOrder": [
    "project.md",
    "project.vscode.md",
    "context.md",
    "context.vscode.md"
  ],
  "aiPrompts.mergeStrategy": "smart",
  "aiPrompts.scopeAware": true, // Enable monorepo path-aware loading
  "aiPrompts.debug": false // Show debug info about loaded prompts
}
```

### Editor Implementation Example

```javascript
// Pseudocode for scope matching
function getApplicablePrompts(filePath, config) {
  const rules = config.rules.filter(rule =>
    micromatch.isMatch(filePath, rule.scope)
  );

  // Sort by specificity (more specific patterns first)
  rules.sort((a, b) => compareSpecificity(a.scope, b.scope));

  return rules.flatMap(rule => rule.files);
}
```

### Phase 3: Ecosystem Development

1. **Develop Core Tools**:
   - Configuration validation tool: `prompt-validator`
   - File merging tool: `prompt-merger`
   - Migration assistance tool: `prompt-migrate`
   - Scope debug tool: `prompt-debug` (shows which prompts apply to which paths)

2. **Editor Plugins**:
   - VSCode extension: `vscode-prompt-config-integration`
   - Cursor plugin: `cursor-prompt-config-integration`

3. **Template Library**:
   - Configuration templates for common project types
   - Best practice examples for various languages
   - Monorepo configuration examples

## Detailed Configuration Specifications

### Project Prompts Template (`.prompt/rules/project.md`)

```markdown
# Project Prompts

## Project Overview
{{Project description}}

## Coding Standards
- Language: {{Programming language}}
- Code Style: {{Code style guidelines}}
- Naming Conventions: {{Naming rules}}

## Architecture Constraints
- Design Patterns: {{Patterns used}}
- Layered Architecture: {{Architecture layers}}
- Dependency Management: {{Dependency specifications}}

## Business Rules
{{Specific business logic and requirements}}
```

### Editor-Specific Extension Example (`.prompt/rules/project.vscode.md`)

```markdown
# VSCode Specific Configuration

## Editor Integration
- Preferred VSCode Extensions: {{Extension list}}
- Debug Configuration: {{Debug settings}}
- Code Snippets: {{Custom snippets}}

## Performance Optimization
- Large File Handling Strategy: {{Handling approach}}
- Memory Management: {{Optimization suggestions}}
```

### Complete config.json Specification

```json
{
  "promptFileVersion": "string",          // Configuration version number
  "responseLanguage": "string",           // Default response language
  "defaultMergeStrategy": "string",       // Default merge strategy
  "fileLoadingOrder": ["string"],         // File loading order
  "maxFileSizeKB": number,                // Maximum file size limit
  "encoding": "string",                   // File encoding format
  "validationRules": {                    // Validation rules
    "maxLength": number,
    "allowedSections": ["string"],
    "requiredSections": ["string"]
  },
  "editorSpecific": {
    "[editor_name]": {
      "promptPriority": ["string"],       // Prompt priority
      "maxTokenLimit": number,            // Maximum token limit
      "temperature": number,              // Temperature parameter
      "modelPreferences": ["string"]      // Model preferences
    }
  },
  "extensions": {                         // Extension configuration
    "customSections": ["string"],         // Custom sections
    "templateVariables": {                // Template variables
      "key": "value"
    }
  }
}
```

## Migration Guide

### Migrating from Existing Systems

1. **Automatic Migration**:
   ```bash
   # Use migration tool
   npx @prompt-standard/migrate --editor cursor --editor trae

   # Or use interactive migration
   npx @prompt-standard/migrate-interactive
   ```

2. **Manual Migration**:
   ```bash
   # 1. Create standard directory
   mkdir -p .prompt/rules

   # 2. Migrate and rename existing configurations
   mv .cursor/rules.md .prompt/rules/project.cursor.md
   mv .trae/rules/project_rules.md .prompt/rules/project.trae.md

   # 3. Create symbolic links (optional, for compatibility)
   ln -s ../.prompt/rules/project.cursor.md .cursor/rules.md
   ln -s ../.prompt/rules/project.trae.md .trae/rules/project_rules.md
   ```

3. **Validation**:
   ```bash
   # Install validation tool
   npm install -g @prompt-standard/validator

   # Validate configuration integrity
   prompt-validate
   ```

### Validation Tool Features

```bash
# Check configuration completeness
prompt-validate --check-completeness

# Validate syntax correctness
prompt-validate --check-syntax

# Generate configuration report
prompt-validate --generate-report

# Check editor compatibility
prompt-validate --check-editor vscode cursor

# Debug path matching (for monorepos)
prompt-debug --path packages/frontend/src/index.js
```

## Benefits Analysis

### For Developers
- ✅ **Consistency**: Unified experience across editors, reduced context switching
- ✅ **Maintainability**: Centralized management, single source of truth
- ✅ **Flexibility**: Support for global configurations and editor-specific extensions
- ✅ **Collaboration**: Standardized team configurations, easy sharing
- ✅ **Monorepo Support**: Path-specific rules for different projects in a monorepo

### For Editor Vendors
- ✅ **Interoperability**: Reduce user migration costs, increase user retention
- ✅ **Ecosystem**: Promote toolchain integration and plugin development
- ✅ **Standards Compliance**: Demonstrate professionalism and foresight, enhance competitiveness
- ✅ **Extensibility**: Support for custom configurations and advanced features

## Action Plan

### Immediate Actions (1-2 weeks)
- [ ] Create core tool library: `@prompt-standard/core`
- [ ] Develop configuration validation tool: `@prompt-standard/validator`
- [ ] Write detailed documentation and examples
- [ ] Establish community discussion groups (Discord/GitHub Discussions)
- [ ] Add monorepo path-matching support to core library

### Short-term Goals (1-3 months)
- [ ] Gain support from 2-3 mainstream editors
- [ ] Create configuration templates for popular projects
- [ ] Develop IDE plugin prototypes
- [ ] Establish automated testing processes
- [ ] Add monorepo examples and best practices

### Long-term Vision (6-12 months)
- [ ] Become a de facto standard, natively supported by multiple editors
- [ ] Establish certification programs and compatibility testing
- [ ] Extend support to more AI development tools
- [ ] Develop graphical configuration management tools
- [ ] Advanced monorepo support with inheritance and override rules

## Technical Support

### Development Tools
```bash
# Core library installation
npm install @prompt-standard/core

# Validation tool installation
npm install -g @prompt-standard/validator

# Migration tool installation
npm install -g @prompt-standard/migrate

# Debug tool installation
npm install -g @prompt-standard/debug
```

### API Examples
```javascript
// Using core library
import { PromptLoader, PromptValidator } from '@prompt-standard/core'

const loader = new PromptLoader('.prompt/rules')
const prompts = await loader.loadForEditor('vscode')

const validator = new PromptValidator()
const results = await validator.validate(prompts)

// For monorepo path-aware loading
const pathSpecificPrompts = await loader.loadForPath('packages/frontend/src/index.js', 'vscode')
```

## Call for Participation

We invite participation from:

### For Editor Developers
- Implement native standard support
- Provide feedback and improvement suggestions
- Participate in standard specification development
- Add monorepo path-aware prompt loading

### For Open Source Projects
- Adopt standard configurations early
- Contribute project configuration templates
- Share usage experiences and best practices
- Provide monorepo use cases

### For Community Members
- Test and report issues
- Participate in documentation writing and translation
- Develop auxiliary tools and plugins
- Contribute to monorepo support features

### For Enterprise Users
- Provide real business requirements
- Share enterprise-level application scenarios
- Participate in standard promotion and implementation
- Contribute large-scale monorepo requirements

**Participation Methods**:
- 📝 Submit Issues and Feature Requests
- 🔄 Submit Pull Requests
- 💬 Join community discussions
- 🚀 Share your configuration examples
- 🏗️ Contribute monorepo solutions

**GitHub Repository**: https://github.com/aicode-standard/prompt-config

Join the discussion in [GitHub Issues](https://github.com/aicode-standard/prompt-config/issues)!
