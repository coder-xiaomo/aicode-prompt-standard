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

##  proposed 标准

### 核心标准路径

```
.aicode/
├── prompts/
│   ├── project.md          # 项目级提示词（主要文件）
│   ├── context.md          # 上下文规则
│   ├── style.md            # 代码风格规范
│   └── editor-specific/    # 编辑器特定配置（可选）
├── config.json             # 配置文件
└── README.md               # 说明文档
```

### 配置文件格式 (`config.json`)

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

## 实施路径

### 阶段一：兼容层（立即可行）

创建转换工具和安装脚本：

```bash
# install-aicode-prompts.sh
#!/bin/bash
mkdir -p .aicode/prompts

# 检测已安装的编辑器并创建符号链接
if [ -d ".cursor" ]; then
    ln -sf ../.aicode/prompts/project.md .cursor/rules.md
fi

if [ -d ".trae" ]; then
    mkdir -p .trae/rules
    ln -sf ../../.aicode/prompts/project.md .trae/rules/project_rules.md
fi
```

### 阶段二：编辑器适配

鼓励编辑器厂商支持标准路径：

```json
// 编辑器配置示例
{
  "promptSources": [
    ".aicode/prompts/project.md",  // 标准路径（优先）
    ".cursor/rules.md"            // 传统路径（回退）
  ]
}
```

### 阶段三：生态系统建设

1. 创建验证工具检查配置完整性
2. 开发 VS Code 等编辑器的配置文件生成插件
3. 建立模板库和最佳实践

## 配置文件详细规范

### `project.md` 模板

```markdown
# 项目提示词

## 项目概述
{{项目描述}}

## 编码规范
- 语言: {{编程语言}}
- 风格: {{代码风格}}
- 限制: {{技术限制}}

## 业务规则
{{具体业务逻辑}}
```

### `config.json` 完整规范

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

## 迁移指南

### 从现有系统迁移

1. **渐进式迁移**：
   ```bash
   # 1. 创建标准目录
   mkdir -p .aicode/prompts

   # 2. 移动现有配置
   cp .cursor/rules.md .aicode/prompts/project.md

   # 3. 创建符号链接
   ln -s .aicode/prompts/project.md .cursor/rules.md
   ```

2. **验证工具**：
   ```bash
   npx aicode-validate
   ```

## 收益分析

### 对开发者

- ✅ **一致性**：跨编辑器统一体验
- ✅ **可维护性**：单一事实来源
- ✅ **协作性**：团队配置标准化

### 对编辑器厂商

- ✅ **互操作性**：降低用户迁移成本
- ✅ **生态系统**：促进工具链集成
- ✅ **标准遵从**：体现专业性和前瞻性

## 行动计划

1. **立即行动**（1-2周）：
   - 创建参考实现和转换工具
   - 编写详细文档和示例
   - 建立社区讨论组

2. **短期目标**（1-3月）：
   - 争取 2-3 个主流编辑器支持
   - 创建流行项目的配置模板
   - 开发 IDE 插件

3. **长期愿景**（6-12月）：
   - 成为事实标准
   - 建立认证程序
   - 扩展更多 AI 开发工具支持

## 呼吁参与

我们邀请以下各方参与：
- **编辑器开发者**：实现标准支持
- **开源项目**：率先采用标准
- **社区成员**：提供反馈和贡献
- **企业用户**：分享实际需求
