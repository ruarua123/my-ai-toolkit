---
name: g-review
description: >
  全语言代码审查核心技能。融合 ai-cortex、everything-claude-code、decebals/claude-code-java、
  existential-birds/beagle、langgenius/dify、bamzc/frontend-code-review 等顶级技能的最佳实践。
  支持通用审查、Java、Go、前端四条审查路径，提供结构化评分报告与置信度过滤。
origin: g-review-plugin
version: "1.0.0"
---

# g-review 代码审查核心技能

> 集合 24+ 个顶级 code review skill 的精华，覆盖通用、Java、Go、前端四条审查路径。

---

## 激活时机

- 写完或修改代码后，提交前
- PR 合并前的质量把关
- 入职新项目，理解现有代码规范
- `/g-review`、`/g-review-java`、`/g-review-go`、`/g-review-fe` 命令被调用时

---

## 置信度过滤原则（来自 ECC code-reviewer agent）

**只报告你 >80% 确信是真实问题的内容**：

- ✅ 上报：安全漏洞、会造成 bug 或数据丢失的问题
- ✅ 上报：违反项目既有规范的问题
- ❌ 跳过：纯风格偏好（除非违反项目约定）
- ❌ 跳过：未修改代码中的问题（CRITICAL 安全问题除外）
- ❌ 合并：同类问题合并上报（如"5 处缺少错误处理"而非 5 条独立 finding）

---

## 通用审查维度（所有语言共享）

### 🔴 CRITICAL — 安全（必须修复）

| 维度 | 说明 |
|------|------|
| 硬编码凭证 | API Key、密码、Token 出现在源码中 |
| SQL 注入 | 字符串拼接 SQL 而非参数化查询 |
| XSS | 未转义用户输入渲染到 HTML/JSX |
| 路径遍历 | 用户控制的文件路径未经净化 |
| CSRF | 状态变更接口缺少 CSRF 保护 |
| 认证绕过 | 受保护路由缺少鉴权检查 |
| 日志泄露 | 将 Token、密码、PII 写入日志 |

### 🟠 HIGH — 代码质量（应当修复）

| 维度 | 说明 |
|------|------|
| 错误处理缺失 | 忽略错误返回值，无 try/catch 兜底 |
| 函数过长 | 单函数超过 50 行 |
| 文件过长 | 单文件超过 800 行 |
| 嵌套过深 | 超过 4 层嵌套 |
| 资源泄漏 | 文件/连接/内存未在所有路径释放 |
| 并发不安全 | 共享状态无同步保护 |

### 🟡 MEDIUM — 可维护性（建议修复）

| 维度 | 说明 |
|------|------|
| 可测试性 | 无法依赖注入，存在全局状态 |
| 命名语义 | 变量/函数名不表达意图 |
| 重复代码 | 超过 3 处相同逻辑未抽象 |
| 缺少文档 | 公开 API 无注释说明 |
| TODO/FIXME | 遗留未完成标记 |

### 🟢 LOW — 规范与风格（可改进）

| 维度 | 说明 |
|------|------|
| 调试代码 | console.log / fmt.Println / System.out.println 残留 |
| 不可变性 | 可以用 const/final/val 的地方使用了可变变量 |
| 性能意识 | 热路径中的不必要分配/计算 |

---

## 语言专属审查维度

### Java 专属（来自 decebals、ai-cortex、ECC java-coding-standards）

```
✅ JVM/JPMS 模块边界与 deprecated API 迁移
✅ Spring Bean 生命周期、@Transactional 边界、@Lazy 滥用
✅ JPA N+1 问题（循环查询替代 JOIN FETCH）
✅ try-with-resources / AutoCloseable 正确使用
✅ synchronized/volatile/Executor 并发模型 (JMM)
✅ Stream 流中副作用、装箱拆箱性能
✅ Optional 正确用法（不用 Optional.get()，用 orElseThrow）
✅ Java 17+ Record、Sealed Class、Pattern Matching 使用
✅ 自定义领域异常（非宽泛 catch Exception）
✅ 泛型原始类型使用（禁止 raw type）
```

### Go 专属（来自 beagle、ai-cortex、ECC golang-patterns）

```
✅ Goroutine 生命周期管理（泄漏检测、context 取消机制）
✅ Channel 关闭时机与 unbuffered channel 死锁
✅ sync.WaitGroup / sync.RWMutex / sync.Map 正确使用
✅ 错误包装惯用法：fmt.Errorf("op: %w", err)
✅ errors.Is/errors.As（不用 == 比较错误）
✅ Panic 仅用于不可恢复错误（不作为正常流程）
✅ Context 传播：ctx 作为第一个参数，避免 context.Background() 滥用
✅ typed nil 陷阱（接口变量含非 nil 类型但 nil 值）
✅ 接口/值接收器语义（指针 vs 值）
✅ go.mod 版本管理与导出 API 向后兼容
✅ 接受接口，返回具体类型（Accept interfaces, return structs）
✅ 零值可用性设计
```

### 前端专属（来自 dify、bamzc、ai-cortex、ECC frontend-patterns）

```
✅ React Hooks 规则：依赖数组完整性，stale closure 检测
✅ Vue Composition API：ref/reactive 正确使用
✅ 不必要重渲染（useMemo/useCallback/React.memo 时机）
✅ 大列表虚拟化（react-window/vue-virtual-scroller）
✅ Key 稳定性（禁止 array index 作为 key）
✅ Bundle 优化：tree-shaking、动态 import、懒加载
✅ TypeScript 严格模式：禁止 any、正确用 unknown/discriminated union
✅ 可访问性(a11y)：ARIA 属性、语义化 HTML、键盘导航
✅ XSS：dangerouslySetInnerHTML 使用前必须 DOMPurify
✅ 浏览器兼容性：polyfill、CSS 前缀
✅ 表单校验：Zod schema 优先
✅ 状态管理：Context 滥用检测，Zustand/Pinia 正确范式
```

---

## 输出格式规范

每条 Finding 必须包含：

```
[严重度] 问题标题
文件：path/to/file.ext:行号
分类：Security / Concurrency / ErrorHandling / Performance / ...
描述：（1-2 句话说明为什么这是问题）
建议：（具体的修复代码或方向）
```

### 评分卡（来自 bamzc 评分体系）

| 等级 | 分值 | 含义 |
|------|------|------|
| S | 95-100 | 无任何 High+ 问题，生产就绪 |
| A | 85-94 | 最多 2 个 Medium 问题 |
| B | 70-84 | 有 Medium 问题，无 High |
| C | 55-69 | 有 High 问题 |
| D | 40-54 | 有多个 High 问题 |
| F | <40 | 存在 Critical 问题，必须阻断 |

### 最终决策

| 决策 | 条件 |
|------|------|
| ✅ APPROVE | 无 Critical/High 问题 |
| ⚠️ WARNING | 仅有 Medium/Low 问题（谨慎合并） |
| ❌ BLOCK | 存在 Critical 或 High 问题 |
