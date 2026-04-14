---
name: g-review
version: "1.0.0"
description: >
  全语言代码审查插件。融合市面上 24+ 个顶级 code review skill 的精华，
  提供四条审查路径：通用、Java、Go、前端。
  包含 1 个核心技能、4 个专项 Agent、4 个 Command。
author: g-review-plugin
---

# g-review Plugin

> 你的专属代码质量守门人。集合 everything-claude-code、ai-cortex、dify、bamzc、decebals、beagle 等顶级 code review 项目的精华。

## 安装方式

将本插件目录复制到你的项目 `.claude/` 下，或全局安装到 `~/.claude/`：

```bash
# 方式一：复制到当前项目
cp -r plugins/g-review /path/to/your-project/.claude/

# 方式二：全局安装（所有项目可用）
cp -r plugins/g-review ~/.claude/
```

## 包含内容

```
g-review/
├── PLUGIN.md               ← 本文件（插件说明）
├── skills/
│   └── SKILL.md            ← 核心审查知识库（所有维度定义）
├── agents/
│   ├── code-reviewer.md    ← 通用审查 Agent
│   ├── java-reviewer.md    ← Java 专项 Agent
│   ├── go-reviewer.md      ← Go 专项 Agent
│   └── frontend-reviewer.md ← 前端专项 Agent
└── commands/
    ├── g-review.md          ← /g-review 通用命令
    ├── g-review-java.md     ← /g-review-java Java 专项命令
    ├── g-review-go.md       ← /g-review-go Go 专项命令
    └── g-review-fe.md       ← /g-review-fe 前端专项命令
```

## 命令速查

| 命令 | 用途 | 典型场景 |
|------|------|---------|
| `/g-review` | 通用多语言审查 | 多语言项目 / 快速 PR 审查 |
| `/g-review java` | Java 深度专项审查 | Spring Boot / JPA 项目 |
| `/g-review-go` | Go 深度专项审查 | 微服务 / CLI 工具 |
| `/g-review-fe` | 前端深度专项审查 | React / Vue / Next.js 项目 |

## 使用示例

```bash
# 审查本地所有变更（自动识别语言）
/g-review

# 审查 GitHub PR #42
/g-review 42

# 只审查 Java 变更，输出 Java 专项报告
/g-review-java

# 只审查前端变更，含 a11y + TypeScript 严格检查
/g-review-fe

# 只审查 Go 变更，含 goroutine 泄漏 + 竞态检测
/g-review-go
```

## 审查维度覆盖

| 维度 | 通用 | Java | Go | 前端 |
|------|:----:|:----:|:--:|:----:|
| 安全性（注入/XSS/CSRF） | ✅ | ✅ | ✅ | ✅ |
| 错误处理 | ✅ | ✅ | ✅ | ✅ |
| 资源管理 | ✅ | ✅ | ✅ | ✅ |
| 并发安全 | ✅ | ✅ | ✅ | ✅ |
| 可测试性 | ✅ | ✅ | ✅ | ✅ |
| 框架规范 | — | Spring/JPA | goroutine/ctx | Hooks/Composition |
| 性能 | ✅ | Stream/JVM | goroutine泄漏 | 重渲染/Bundle |
| 可访问性(a11y) | — | — | — | ✅ WCAG |
| 类型系统 | — | 泛型/Record | 接口/typed nil | TypeScript strict |
| 评分报告 | ✅ S/A/B/C/D/F | ✅ | ✅ | ✅ |

## 评分等级说明

| 等级 | 分值 | 含义 |
|------|------|------|
| S | 95-100 | 生产就绪，无任何 High+ 问题 |
| A | 85-94 | 最多 2 个 Medium 问题 |
| B | 70-84 | 有 Medium 问题，无 High |
| C | 55-69 | 有 High 问题 |
| D | 40-54 | 有多个 High 问题 |
| F | <40 | 存在 Critical 问题，必须阻断合并 |

## 数据来源与致谢

本插件融合了以下开源项目的精华：

- [everything-claude-code](https://github.com/affaan-m/everything-claude-code) — 整体架构参考、Go/Java/前端 agent 结构
- [ai-cortex](https://github.com/nesnilnehc/ai-cortex) — 原子型 skill 架构、编排模式
- [decebals/claude-code-java](https://github.com/decebals/claude-code-java) — Java 专项深度规范
- [existential-birds/beagle](https://github.com/existential-birds/beagle) — Go 专项规范
- [langgenius/dify](https://github.com/langgenius/dify) — 前端全框架审查
- [bamzc/claude-skills-frontend](https://github.com/bamzc/claude-skills-frontend) — 前端评分体系
- [coderabbitai/skills](https://github.com/coderabbitai/skills) — PR 审查流程
- [google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli) — 置信度过滤原则
