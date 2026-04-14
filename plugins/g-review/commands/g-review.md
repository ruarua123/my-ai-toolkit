---
description: 通用代码审查，自动识别语言，覆盖安全、质量、性能、可维护性四大维度。支持本地变更审查和 PR 审查两种模式。
argument-hint: "[pr编号 | pr链接 | 留空审查本地变更]"
---

# /g-review — 通用代码审查

**输入**：$ARGUMENTS

---

## 模式选择

- 如果 `$ARGUMENTS` 包含 PR 编号、PR 链接或 `--pr`：→ 进入 **PR 审查模式**
- 否则：→ 进入 **本地变更审查模式**

---

## 本地变更审查模式

### 第 1 阶段：收集变更

```bash
git diff --name-only HEAD
```

若无变更文件，停止并提示"没有检测到变更，请先暂存代码（git add）。"

### 第 2 阶段：语言路由

检测变更文件的语言分布：

| 检测到的文件类型 | 追加的专项审查 |
|----------------|--------------|
| `*.go` | 调用 go-reviewer agent 补充 Go 专项 |
| `*.java` | 调用 java-reviewer agent 补充 Java 专项 |
| `*.ts` / `*.tsx` / `*.js` / `*.jsx` / `*.vue` | 调用 frontend-reviewer agent 补充前端专项 |

### 第 3 阶段：通用审查

读取每个变更文件的**完整内容**（不只看 diff）。

运行静态分析：

```bash
# 调试代码残留扫描
grep -rn "console\.log\|fmt\.Println\|System\.out\.print\|TODO\|FIXME\|HACK" . \
  --include="*.go" --include="*.java" --include="*.ts" --include="*.tsx" --include="*.js" \
  2>/dev/null | grep -v node_modules | grep -v ".git" | head -30
```

按照以下维度审查（严格执行置信度过滤，只上报 >80% 确信的问题）：

**🔴 CRITICAL — 安全（发现即阻断）**
- 硬编码凭证（API Key、密码、Token）
- 注入攻击（SQL / 命令 / 路径遍历）
- XSS（未转义用户输入渲染到 HTML）
- 认证绕过（受保护路由无鉴权）
- 日志泄露敏感数据

**🟠 HIGH — 代码质量**
- 错误处理缺失或被忽略
- 资源泄漏（文件/连接/内存）
- 并发不安全（共享状态无保护）
- 函数超 50 行 / 文件超 800 行

**🟡 MEDIUM — 可维护性**
- 可测试性问题（全局状态、硬编码依赖）
- 命名不表达意图
- 重复代码（超 3 处相同逻辑）
- 公开 API 缺少注释

**🟢 LOW — 规范风格**
- 调试代码残留
- 不可变性可改进处
- 性能小问题

### 第 4 阶段：输出报告

```markdown
# 代码审查报告 — 通用

**时间**：<当前时间>
**变更文件**：N 个（Go: x | Java: x | TS/JS: x | 其他: x）

## 静态分析

| 工具 | 结果 |
|------|------|
| 调试代码扫描 | ✅ 无残留 / ⚠️ N 处 |
| 特定语言工具 | 见下方语言专项 |

## 发现问题

### 🔴 CRITICAL
（无 / 列出每个问题：文件:行号 + 描述 + 修复建议）

### 🟠 HIGH
（无 / 列出问题）

### 🟡 MEDIUM
（无 / 列出问题）

### 🟢 LOW
（无 / 列出问题）

## 综合评分

| 维度 | 得分 |
|------|------|
| 安全性 | xx/25 |
| 代码质量 | xx/25 |
| 可维护性 | xx/25 |
| 性能/规范 | xx/25 |
| **总分** | **xx/100 （等级 X）** |

## 审查结论

✅ APPROVE — 无 Critical/High 问题，可合并
⚠️ WARNING — 有 Medium 问题，建议修复后合并
❌ BLOCK — 存在 Critical 或 High 问题，必须修复

## 优先修复建议

1. （最高优先级）
2. （次优先级）
```

---

## PR 审查模式

### 第 1 阶段：获取 PR 信息

解析 `$ARGUMENTS` 得到 PR 编号（支持纯数字、GitHub URL、分支名）：

```bash
# 获取 PR 基本信息
gh pr view <编号> --json number,title,body,author,baseRefName,headRefName,changedFiles,additions,deletions

# 获取 diff
gh pr diff <编号>
```

若 `gh` 未安装或 PR 不存在，回退到本地变更审查模式。

### 第 2 阶段：上下文收集

1. 读取 `CLAUDE.md` / `README.md` / `.claude/` 目录了解项目约定
2. 解析 PR 描述中的目标、关联 issue、测试计划
3. 按文件类型分类（源码/测试/配置/文档）

### 第 3 阶段：执行审查

与本地模式相同的审查流程，但基于 PR diff。

### 第 4 阶段：发布评审

```bash
# APPROVE
gh pr review <编号> --approve --body "<审查摘要>"

# REQUEST CHANGES
gh pr review <编号> --request-changes --body "<问题摘要及修复要求>"
```

保存审查报告到 `.claude/reviews/pr-<编号>-review.md`。

---

## 与其他命令配合

- `/g-review-java` — 聚焦 Java 深度审查
- `/g-review-go` — 聚焦 Go 深度审查  
- `/g-review-fe` — 聚焦前端深度审查
- `/g-plan` — 先规划再实现
