---
description: 前端专项深度代码审查。覆盖 React/Vue/Next.js/TypeScript，深度检测 Hooks 规则、stale closure、重渲染、Bundle 优化、TypeScript 严格模式、可访问性(a11y)、XSS/CSRF。融合 dify、bamzc、ai-cortex@review-typescript/react/vue、ECC frontend-patterns 的全部精华。
argument-hint: "[文件路径 | 留空审查所有前端变更]"
---

# /g-review-fe — 前端专项代码审查

**输入**：$ARGUMENTS

---

## 第 1 阶段：收集前端变更

```bash
if [ -n "$ARGUMENTS" ]; then
  git diff --name-only HEAD | grep "$ARGUMENTS"
else
  git diff --name-only HEAD | grep -E '\.(ts|tsx|js|jsx|vue|css|scss|less)$'
fi
```

若无前端变更文件，停止并提示。

## 第 2 阶段：感知项目上下文

```bash
# 检测框架和依赖
cat package.json 2>/dev/null | python3 -c "
import json, sys
p = json.load(sys.stdin)
deps = {**p.get('dependencies', {}), **p.get('devDependencies', {})}
keys = ['react', 'vue', 'next', 'nuxt', 'angular', 'svelte', 'typescript', 'vite', 'webpack', 'eslint', 'zod', 'zustand', 'pinia']
for k in keys:
    if k in deps: print(f'{k}: {deps[k]}')
" 2>/dev/null || cat package.json 2>/dev/null | head -30

# TypeScript 配置
cat tsconfig.json 2>/dev/null | python3 -c "
import json, sys
t = json.load(sys.stdin)
opts = t.get('compilerOptions', {})
print('strict:', opts.get('strict'))
print('noImplicitAny:', opts.get('noImplicitAny'))
print('strictNullChecks:', opts.get('strictNullChecks'))
" 2>/dev/null || cat tsconfig.json 2>/dev/null | head -20

# ESLint 配置
ls .eslintrc* .eslint.config* eslint.config* 2>/dev/null | head -5
```

读取每个变更文件的**完整内容**，理解组件树、状态流、数据获取模式。

## 第 3 阶段：静态分析工具

```bash
# TypeScript 类型检查
which npx && npx tsc --noEmit 2>&1 | head -40 || echo "⏭️ TypeScript 未配置"

# ESLint（含 react-hooks、jsx-a11y 插件）
which npx && npx eslint . --ext .ts,.tsx,.js,.jsx,.vue --max-warnings 0 2>&1 | head -50 || \
  echo "⏭️ ESLint 未安装或未配置"

# 搜索高危模式
echo "=== 安全扫描 ==="
grep -rn "dangerouslySetInnerHTML\|v-html\|eval(\|new Function(" \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" --include="*.vue" \
  . 2>/dev/null | grep -v node_modules | head -20

echo "=== 调试代码扫描 ==="
grep -rn "console\.log\|console\.warn\|console\.error\|debugger\|TODO\|FIXME" \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" \
  . 2>/dev/null | grep -v node_modules | grep -v ".test." | grep -v ".spec." | head -20

echo "=== any 类型扫描 ==="
grep -rn ": any\|as any\|<any>" \
  --include="*.ts" --include="*.tsx" \
  . 2>/dev/null | grep -v node_modules | grep -v ".d.ts" | head -20
```

## 第 4 阶段：深度前端专项审查

**🔴 CRITICAL — 安全**

| 问题类型 | 检查要点 |
|---------|---------|
| XSS（React）| `dangerouslySetInnerHTML` 未通过 `DOMPurify.sanitize()` |
| XSS（Vue）| `v-html` 绑定未经净化的用户内容 |
| eval 执行 | `eval()` / `new Function()` 接收用户输入 |
| CSRF | 状态变更接口（POST/PUT/DELETE）无 CSRF Token |
| 硬编码密钥 | 前端代码中出现 `apiKey`, `secretKey`, `password` 字面量 |
| 敏感数据存储 | 将 Token/密码存入 `localStorage`（未加密）|

**🟠 HIGH — React Hooks 规则（检测到 React/Next.js 时）**

| 问题类型 | 检查要点 |
|---------|---------|
| 依赖数组不完整 | `useEffect/useCallback/useMemo` 中引用了外部变量但未列入依赖 |
| Stale Closure | `setInterval/setTimeout` 中捕获了旧的 state/props（应用 `useRef` 或函数式更新）|
| Hook 在条件中调用 | `if/else` 或循环内部调用 Hook（违反 Hooks 规则）|
| Effect 修改 DOM | 直接操作 `document.xxx`（应通过 `useRef`）|
| 竞态条件 | async Effect 未处理 cleanup（组件卸载后仍 setState）|

**🟠 HIGH — Vue Composition API（检测到 Vue/Nuxt 时）**

| 问题类型 | 检查要点 |
|---------|---------|
| reactive 解构 | `const { count } = reactive({count: 0})` 丢失响应性（应用 `toRefs`）|
| setup 外调用 | `ref()/reactive()` 在 `setup()` 外调用 |
| 计算属性副作用 | `computed()` 内部修改响应式数据 |
| watch 依赖不精确 | `watchEffect` 意外追踪过多依赖 |

**🟡 MEDIUM — 性能**

| 问题类型 | 检查要点 |
|---------|---------|
| 不必要重渲染 | 子组件每次父更新都重渲染（可用 `React.memo` / `shallowRef`）|
| useMemo 滥用 | 记忆化简单的原始值（开销大于收益）|
| Key 使用 index | `list.map((item, i) => <Item key={i} />)`（重排时状态错乱）|
| 大列表未虚拟化 | 超过 100 条数据直接渲染（应用 `react-window`/`vue-virtual-scroller`）|
| 未做代码分割 | 路由页面未用 `React.lazy()`/动态 `import()` |
| 整库引入 | `import _ from 'lodash'`（应用 `import debounce from 'lodash/debounce'`）|
| 图片缺少尺寸 | `<img>` 无 `width`/`height`（导致 CLS 布局偏移）|

**🟡 MEDIUM — TypeScript 规范**

| 问题类型 | 检查要点 |
|---------|---------|
| any 滥用 | `: any` / `as any`（应用 `unknown` + 类型守卫）|
| 非空断言滥用 | `value!.property`（应做条件判断）|
| 未用 discriminated union | 互斥状态用 `string` 而非 `type State = 'loading' \| 'success' \| 'error'` |
| 接口缺少注释 | 公开接口/类型无 JSDoc |
| 枚举滥用 | 数字枚举（优先用字符串字面量联合类型）|

**🟡 MEDIUM — 可访问性 (a11y)**

| 问题类型 | 检查要点 |
|---------|---------|
| 无标签交互元素 | `<button>` / `<a>` 无文本内容且无 `aria-label` |
| 图片缺 alt | `<img>` 无 `alt` 属性 |
| 表单无关联 | `<input>` 无对应 `<label>` 或 `aria-labelledby` |
| 键盘不可达 | `div onClick` 但无 `onKeyDown/onKeyPress` |
| 颜色对比度 | 文字与背景对比度 < 4.5:1（WCAG AA）|
| 焦点未捕获 | 模态框/弹层打开后焦点未 trap 在弹层内 |

**🟢 LOW — 代码规范**

| 问题类型 | 检查要点 |
|---------|---------|
| 调试代码 | `console.log` / `debugger` 残留 |
| 组件过大 | 单文件超 300 行（应拆分）|
| 魔法数字/字符串 | 未提取为常量 |
| 未用 import | 引入了未使用的包 |
| 注释中文乱码 | 编码问题 |

## 第 5 阶段：框架专项（按检测到的框架执行）

### Next.js 专项
```bash
grep -rn "getServerSideProps\|getStaticProps\|use client\|use server" \
  --include="*.ts" --include="*.tsx" . 2>/dev/null | head -20
```
- [ ] Server Component 中无 `useState`/`useEffect`（客户端 API）
- [ ] `getServerSideProps` 有权限验证
- [ ] `<Image>` 替代原生 `<img>`（自动优化）

### Nuxt 3 专项
- [ ] `defineProps` 用 `withDefaults` 设置默认值
- [ ] `useFetch`/`useAsyncData` 用于服务端数据获取
- [ ] Pinia store 的 `state` 是工厂函数（SSR 防串状态）

## 第 6 阶段：输出完整报告

```markdown
# 前端代码审查报告

**审查时间**：<当前时间>
**框架**：React <版本> / Vue <版本> / Next.js <版本>
**TypeScript**：是（strict: true/false）/ 否
**变更文件**：N 个（.tsx: x | .ts: x | .vue: x | .css: x）

## 静态分析结果

| 工具 | 结果 |
|------|------|
| TypeScript 类型检查 | ✅ 通过 / ❌ N 个错误 / ⏭️ 未配置 |
| ESLint | ✅ 通过 / ❌ N 个问题 / ⏭️ 未安装 |
| 安全扫描（dangerouslySetInnerHTML等）| ✅ 无风险 / ⚠️ N 处需审查 |

## 发现问题

### 🔴 CRITICAL（安全）
（无 / 格式：[CRITICAL] 标题 \n 文件:行号 \n 风险描述 \n 修复代码示例）

### 🟠 HIGH（Hooks / Composition API 规范）
（无 / 列出问题）

### 🟡 MEDIUM（性能 / TypeScript / a11y）
（无 / 列出问题）

### 🟢 LOW（代码规范）
（无 / 列出问题）

## 综合评分

| 维度 | 得分 |
|------|------|
| 安全性 | xx/20 |
| 框架规范（Hooks/Composition） | xx/20 |
| 渲染性能 | xx/20 |
| TypeScript 规范 | xx/20 |
| 可访问性（a11y） | xx/20 |
| **总分** | **xx/100（等级 X）** |

## 审查结论

✅ APPROVE / ⚠️ WARNING / ❌ BLOCK

## 推荐资源（针对本次发现问题）

（根据实际问题动态生成）
- React Hooks 规则：https://react.dev/reference/rules/rules-of-hooks
- Stale Closure：https://dmitripavlutin.com/react-hooks-stale-closures/
- DOMPurify：https://github.com/cure53/DOMPurify
- WCAG 对比度：https://webaim.org/resources/contrastchecker/
```

---

## 关联命令

- `/g-review` — 通用多语言审查（包含前端基础审查）
- `/g-plan` — 先规划前端重构方案
