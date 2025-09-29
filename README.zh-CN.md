# AI 代码编辑器提示词配置文件标准化提案

[English](./README.md) | 简体中文

## 提案概述

为 AI 代码编辑器创建一个统一的提示词配置文件标准，解决当前各编辑器使用不同路径和格式导致的配置碎片化问题。

## 当前问题

1. **路径不统一**：
   - Cursor: `.cursor/rules.md`
   - Trae: `.trae/rules/project_rules.md`
   - Codeium: `.codeium/context.md`
   - 其他编辑器各有自己的约定

2. **配置同步困难**：团队成员使用不同编辑器时需要维护多份相同内容
3. **版本控制冲突**：不同编辑器的配置文件需要分别添加到 `.gitignore`
4. **迁移成本高**：切换编辑器时需要重新配置提示词

## Proposed 标准

### 核心标准路径

```
.prompt/
  ├─ rules/
  │  ├─ project[.<editor-identifer>].md     # 项目级提示词（主要文件）
  │  ├─ context[.<editor-identifer>].md     # 上下文规则（可选）
  │  └─ code_style[.<editor-identifer>].md  # 代码风格规范（可选）
  ├─ config.json                            # 配置文件
  └─ README.md                              # 说明文档（可选）
```

其中，`[.<editor-identifer>]` 是编辑器特定配置。不添加则是全局使用的公共提示词文件。

### 配置示例

**基础配置：**

如果我们想配置项目级提示词，我们可以这样做：

```
.prompt/
  └─ rules/
     ├─ project.md                        # 项目级提示词（主要文件）
     ├─ project.vscode.md                 # 在 VSCode 编辑器中扩展 `project.md` 文件（可选）
     ├─ project.cursor.md                 # 在 Cursor 编辑器中扩展 `project.md` 文件（可选）
     └─ project.trae.md                   # 在 TRAE 编辑器中扩展 `project.md` 文件（可选）
```

**高级配置（多文件结构）：**

当提示词内容较多，需要拆分成多个文件时，可以创建对应目录，将提示词按照自定义目录结构进行分类。同样的，也以 `[.<editor-identifer>].md` 结尾。所有的文件都是可选的。

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

在这个示例中，`.prompt/rules/project/index.md` 与 `.prompt/rules/project.md` 作用相同，同时存在时，他们都将生效。

对于 VSCode 编辑器，在这个示例中，会加载以下 markdown 文件作为提示词：
- `.prompt/rules/project/[foo].vscode.md`
- `.prompt/rules/project/index.md`
- `.prompt/rules/project.md`
- `.prompt/rules/project.vscode.md`

### 文件加载规则

1. **全局文件**：`.prompt/rules/project.md`（所有编辑器共享）
2. **编辑器特定文件**：`.prompt/rules/project.{editor}.md`（按编辑器加载）
3. **目录结构**：支持子目录组织，`index.md` 作为目录入口文件
4. **合并策略**：所有匹配的文件内容会被合并，编辑器特定配置会扩展全局配置

## 配置文件格式 (`.prompt/config.json`)

```json
{
  "promptFileVersion": "1.0",
  "response_language": "zh-CN", // 指定模型返回的语言
  "defaultMergeStrategy": "append", // 默认合并策略：append|overwrite|smart
  "fileLoadingOrder": ["global", "editor-specific"], // 文件加载顺序
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

## 实施路径

### 阶段一：兼容层（立即可行）

创建转换工具和符号链接脚本：

```bash
# migrate-prompts.sh
#!/bin/bash

# 创建标准目录结构
mkdir -p .prompt/rules

# 迁移现有配置并创建符号链接
migrate_editor_config() {
  local editor=$1
  local source_path=$2
  local target_name=$3

  if [ -f "$source_path" ]; then
    # 迁移到新位置
    cp "$source_path" ".prompt/rules/${target_name}.${editor}.md"
    # 创建符号链接保持向后兼容
    ln -sf "../.prompt/rules/${target_name}.${editor}.md" "$source_path"
    echo "Migrated ${editor} configuration"
  fi
}

# 迁移各编辑器配置
migrate_editor_config "cursor" ".cursor/rules.md" "project"
migrate_editor_config "trae" ".trae/rules/project_rules.md" "project"
migrate_editor_config "codeium" ".codeium/context.md" "context"

# 创建基础配置文件
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

### 阶段二：编辑器适配

鼓励编辑器厂商原生支持标准路径：

```json
// 编辑器配置示例 (VSCode settings.json)
{
  "aiPrompts.standardPath": ".prompt/rules",
  "aiPrompts.loadOrder": [
    "project.md",
    "project.vscode.md",
    "context.md",
    "context.vscode.md"
  ],
  "aiPrompts.mergeStrategy": "smart"
}
```

### 阶段三：生态系统建设

1. **开发核心工具**：
   - 配置验证工具：`prompt-validator`
   - 文件合并工具：`prompt-merger`
   - 迁移辅助工具：`prompt-migrate`

2. **编辑器插件**：
   - VSCode 扩展：`vscode-prompt-config-integration`
   - Cursor 插件：`cursor-prompt-config-integration`

3. **模板库**：
   - 常见项目类型的配置模板
   - 各语言的最佳实践示例

## 配置文件详细规范

### 项目提示词模板 (`.prompt/rules/project.md`)

```markdown
# 项目提示词

## 项目概述
{{项目描述}}

## 编码规范
- 语言: {{编程语言}}
- 代码风格: {{代码风格指南}}
- 命名约定: {{命名规则}}

## 架构约束
- 设计模式: {{使用的模式}}
- 分层架构: {{架构层次}}
- 依赖管理: {{依赖规范}}

## 业务规则
{{具体业务逻辑和要求}}
```

### 编辑器特定扩展示例 (`.prompt/rules/project.vscode.md`)

```markdown
# VSCode 特定配置

## 编辑器集成
- 优先使用 VSCode 扩展: {{扩展列表}}
- 调试配置: {{调试设置}}
- 代码片段: {{自定义片段}}

## 性能优化
- 大文件处理策略: {{处理方式}}
- 内存管理: {{优化建议}}
```

### 完整 config.json 规范

```json
{
  "promptFileVersion": "string",          // 配置版本号
  "responseLanguage": "string",           // 默认响应语言
  "defaultMergeStrategy": "string",       // 默认合并策略
  "fileLoadingOrder": ["string"],         // 文件加载顺序
  "maxFileSizeKB": number,                // 最大文件大小限制
  "encoding": "string",                   // 文件编码格式
  "validationRules": {                    // 验证规则
    "maxLength": number,
    "allowedSections": ["string"],
    "requiredSections": ["string"]
  },
  "editorSpecific": {
    "[editor_name]": {
      "promptPriority": ["string"],       // 提示词优先级
      "maxTokenLimit": number,            // 最大 token 限制
      "temperature": number,              // 温度参数
      "modelPreferences": ["string"]      // 模型偏好
    }
  },
  "extensions": {                         // 扩展配置
    "customSections": ["string"],         // 自定义章节
    "templateVariables": {                // 模板变量
      "key": "value"
    }
  }
}
```

## 迁移指南

### 从现有系统迁移

1. **自动迁移**：
   ```bash
   # 使用迁移工具
   npx @prompt-standard/migrate --editor cursor --editor trae

   # 或使用交互式迁移
   npx @prompt-standard/migrate-interactive
   ```

2. **手动迁移**：
   ```bash
   # 1. 创建标准目录
   mkdir -p .prompt/rules

   # 2. 迁移并重命名现有配置
   mv .cursor/rules.md .prompt/rules/project.cursor.md
   mv .trae/rules/project_rules.md .prompt/rules/project.trae.md

   # 3. 创建符号链接（可选，保持兼容）
   ln -s ../.prompt/rules/project.cursor.md .cursor/rules.md
   ln -s ../.prompt/rules/project.trae.md .trae/rules/project_rules.md
   ```

3. **验证配置**：
   ```bash
   # 安装验证工具
   npm install -g @prompt-standard/validator

   # 验证配置完整性
   prompt-validate
   ```

### 验证工具功能

```bash
# 检查配置完整性
prompt-validate --check-completeness

# 验证语法正确性
prompt-validate --check-syntax

# 生成配置报告
prompt-validate --generate-report

# 检查编辑器兼容性
prompt-validate --check-editor vscode cursor
```

## 收益分析

### 对开发者
- ✅ **一致性**：跨编辑器统一体验，减少上下文切换
- ✅ **可维护性**：集中管理，单一事实来源
- ✅ **灵活性**：支持全局配置和编辑器特定扩展
- ✅ **协作性**：团队配置标准化，易于共享

### 对编辑器厂商
- ✅ **互操作性**：降低用户迁移成本，提高用户粘性
- ✅ **生态系统**：促进工具链集成和插件开发
- ✅ **标准遵从**：体现专业性和前瞻性，增强竞争力
- ✅ **扩展性**：支持自定义配置和高级功能

## 行动计划

### 立即行动（1-2周）
- [ ] 创建核心工具库：`@prompt-standard/core`
- [ ] 开发配置验证工具：`@prompt-standard/validator`
- [ ] 编写详细文档和示例
- [ ] 建立社区讨论组（Discord/GitHub Discussions）

### 短期目标（1-3月）
- [ ] 争取 2-3 个主流编辑器支持
- [ ] 创建流行项目的配置模板
- [ ] 开发 IDE 插件原型
- [ ] 建立自动化测试流程

### 长期愿景（6-12月）
- [ ] 成为事实标准，被多个编辑器原生支持
- [ ] 建立认证程序和兼容性测试
- [ ] 扩展支持更多 AI 开发工具
- [ ] 开发图形化配置管理工具

## 技术支持

### 开发工具
```bash
# 核心库安装
npm install @prompt-standard/core

# 验证工具安装
npm install -g @prompt-standard/validator

# 迁移工具安装
npm install -g @prompt-standard/migrate
```

### API 示例
```javascript
// 使用核心库
import { PromptLoader, PromptValidator } from '@prompt-standard/core'

const loader = new PromptLoader('.prompt/rules')
const prompts = await loader.loadForEditor('vscode')

const validator = new PromptValidator()
const results = await validator.validate(prompts)
```

## 呼吁参与

我们邀请以下各方参与：

### 对于编辑器开发者
- 实现原生标准支持
- 提供反馈和改进建议
- 参与标准规范的制定

### 对于开源项目
- 率先采用标准配置
- 贡献项目配置模板
- 分享使用经验和最佳实践

### 对于社区成员
- 测试和报告问题
- 参与文档编写和翻译
- 开发辅助工具和插件

### 对于企业用户
- 提供实际业务需求
- 分享企业级应用场景
- 参与标准推广和实施

**参与方式**：
- 📝 提交 Issue 和 Feature Request
- 🔄 提交 Pull Request
- 💬 加入社区讨论
- 🚀 分享你的配置案例

**GitHub 仓库**: https://github.com/aicode-standard/prompt-config
