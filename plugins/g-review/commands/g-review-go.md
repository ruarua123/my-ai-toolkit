---
description: Go 专项深度代码审查。覆盖 goroutine 生命周期、context 传播、错误包装惯用法、typed nil、接口语义、go.mod 兼容性。融合 beagle、ai-cortex@review-go、ECC golang-patterns 的全部精华。
argument-hint: "[文件路径 | 留空审查所有 Go 变更]"
---

# /g-review-go — Go 专项代码审查

**输入**：$ARGUMENTS

---

## 第 1 阶段：收集 Go 变更

```bash
if [ -n "$ARGUMENTS" ]; then
  git diff --name-only HEAD | grep "$ARGUMENTS"
else
  git diff --name-only HEAD | grep '\.go$'
fi
```

若无 Go 变更文件，停止并提示。

## 第 2 阶段：感知项目上下文

```bash
# 模块信息
cat go.mod | head -10
go version 2>/dev/null || true
# 包结构概览
go list ./... 2>/dev/null | head -20
```

读取每个变更 .go 文件的**完整内容**，重点理解接口定义、goroutine 启动点、错误传播路径。

## 第 3 阶段：静态分析工具

```bash
# 必须运行
go vet ./... 2>&1

# 增强分析
which staticcheck && staticcheck ./... 2>&1 | head -40 || \
  echo "⏭️ 建议安装: go install honnef.co/go/tools/cmd/staticcheck@latest"

which golangci-lint && golangci-lint run 2>&1 | head -40 || \
  echo "⏭️ 建议安装: https://golangci-lint.run/usage/install/"

# 竞态检测（编译阶段）
go build -race ./... 2>&1 | head -20 || true

# 安全漏洞
which govulncheck && govulncheck ./... 2>&1 | head -20 || \
  echo "⏭️ 建议安装: go install golang.org/x/vuln/cmd/govulncheck@latest"

# 调试代码扫描
grep -rn "fmt\.Println\|fmt\.Printf\|log\.Print\|TODO\|FIXME\|panic(" \
  --include="*.go" . 2>/dev/null | grep -v "_test.go" | grep -v "vendor/" | head -20
```

## 第 4 阶段：深度 Go 专项审查

**🔴 CRITICAL — 安全与错误处理**

逐文件检查：

| 问题类型 | 检查要点 |
|---------|---------|
| SQL 注入 | `database/sql` 中字符串格式化拼接 SQL（`fmt.Sprintf("SELECT...%s", userInput)`）|
| 命令注入 | `exec.Command` 接收未验证的用户输入 |
| 路径遍历 | 用户控制路径未经 `filepath.Clean` + 前缀校验 |
| 竞态条件 | `map`/`slice` 被多个 goroutine 访问无 `sync.RWMutex`/`sync.Map` |
| 错误丢弃 | 关键路径使用 `_` 忽略 error |
| 硬编码凭证 | 代码中出现 `password =`、`apiKey =`、`secret =` 字符串字面量 |
| 不安全 TLS | `InsecureSkipVerify: true` |

**🟠 HIGH — 并发与资源管理**

| 问题类型 | 检查要点 |
|---------|---------|
| Goroutine 泄漏 | `go func()` 启动但无 context 取消或 done channel 退出机制 |
| Channel 死锁 | 无缓冲 channel 发送无接收方；channel 关闭后继续发送 |
| WaitGroup 误用 | `wg.Add(1)` 在 goroutine 内部调用（竞态）|
| Mutex 未 defer | `mu.Lock()` 后提前 return 时锁未释放 |
| Context 滥用 | `context.Background()` 用于业务请求（应从调用方传入）|
| 错误未包装 | `return err` 丢失调用链信息（应用 `fmt.Errorf("op: %w", err)`）|
| 错误比较错误 | `err == io.EOF`（应用 `errors.Is(err, io.EOF)`）|

**🟡 MEDIUM — Go 惯用法**

| 问题类型 | 检查要点 |
|---------|---------|
| Typed nil 陷阱 | 返回 `(*MyError)(nil)` 赋给 `error` 接口，调用方 `!= nil` 会判断为非 nil |
| 返回接口 | 函数返回接口类型（隐藏实现，阻止内联优化）—— 特例：`error` 本身是接口 |
| 零值未设计 | 类型使用前必须调用 `New()`，未遵循 Go 零值可用原则 |
| 循环内 defer | `defer f.Close()` 在循环体内（资源积累到函数结束才释放）|
| 字符串拼接循环 | 循环内 `s += piece`（应用 `strings.Builder`）|
| Slice 未预分配 | 已知长度时 `make([]T, 0)` 而非 `make([]T, 0, n)` |
| 包级可变变量 | `var cache = map[string]..{}` 被多处修改 |
| 测试非 table-driven | 相同逻辑多个 Test 函数（应合并为 table-driven）|
| 公开 API 无 godoc | 导出函数/类型缺少 `// FuncName ...` 注释 |

**🟢 LOW — 风格规范**

| 问题类型 | 检查要点 |
|---------|---------|
| Context 非首位参数 | `func Foo(db *DB, ctx context.Context)` → 应将 ctx 放第一位 |
| 包名含非法字符 | 包名大写 / 含下划线 |
| 错误消息格式 | 错误消息以大写字母开头或以句点结尾 |
| 过早优化 | 无 benchmark 支撑的预优化 |
| 调试代码残留 | fmt.Println / fmt.Printf（非 log 包）|

## 第 5 阶段：go.mod 兼容性检查

```bash
# 检查是否有可升级的依赖
go list -u -m all 2>/dev/null | grep "\[" | head -20
# 检查是否存在 replace directive（可能影响生产环境）
grep "^replace" go.mod 2>/dev/null
```

## 第 6 阶段：输出完整报告

```markdown
# Go 代码审查报告

**审查时间**：<当前时间>
**Go 版本**：<go version 输出>
**模块**：<go.mod module 行>
**变更文件**：N 个 .go 文件（含/不含测试文件）

## 静态分析结果

| 工具 | 结果 |
|------|------|
| go vet | ✅ 通过 / ❌ N 个问题 |
| staticcheck | ✅ 通过 / ❌ N 个问题 / ⏭️ 未安装 |
| golangci-lint | ✅ 通过 / ❌ N 个问题 / ⏭️ 未安装 |
| 竞态编译 | ✅ 通过 / ❌ N 个问题 |
| govulncheck | ✅ 无已知漏洞 / ❌ N 个漏洞 / ⏭️ 未安装 |

## 发现问题

### 🔴 CRITICAL
（无 / 格式：[CRITICAL] 标题 \n 文件:行号 \n 问题描述 \n 修复代码示例）

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
| 并发正确性 | xx/25 |
| Go 惯用法 | xx/25 |
| 可测试性 | xx/25 |
| **总分** | **xx/100（等级 X）** |

## 审查结论

✅ APPROVE / ⚠️ WARNING / ❌ BLOCK

## 推荐工具安装

（根据未安装的工具动态生成）
- staticcheck: `go install honnef.co/go/tools/cmd/staticcheck@latest`
- govulncheck: `go install golang.org/x/vuln/cmd/govulncheck@latest`
- golangci-lint: https://golangci-lint.run/usage/install/
```

---

## 关联命令

- `/g-review` — 通用多语言审查
- `/g-plan` — 先规划 Go 重构方案
