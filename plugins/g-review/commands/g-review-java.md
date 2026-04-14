---
description: Java 专项深度代码审查。覆盖 Java 17+、Spring Boot、JPA、并发模型、JVM 生态。融合 decebals/claude-code-java、ai-cortex@review-java、ECC java-coding-standards 的全部精华。
argument-hint: "[文件路径 | 留空审查所有 Java 变更]"
---

# /g-review-java — Java 专项代码审查

**输入**：$ARGUMENTS

---

## 第 1 阶段：收集 Java 变更

```bash
# 获取变更的 Java 文件
if [ -n "$ARGUMENTS" ]; then
  # 指定文件模式
  git diff --name-only HEAD | grep "$ARGUMENTS"
else
  git diff --name-only HEAD | grep '\.java$'
fi
```

若无 Java 变更文件，停止并提示。

## 第 2 阶段：感知项目上下文

```bash
# 检测 Java 版本、框架、构建工具
cat pom.xml 2>/dev/null | grep -E '<java\.version>|<spring-boot\.version>|<hibernate' | head -10
cat build.gradle 2>/dev/null | grep -E "sourceCompatibility|java|spring|hibernate" | head -10
# 检测是否使用 Spring Boot
grep -rl "@SpringBootApplication\|@RestController\|@Service\|@Repository" --include="*.java" . 2>/dev/null | head -5
```

读取每个变更 Java 文件的**完整内容**。

## 第 3 阶段：静态分析工具

```bash
# Checkstyle
which mvn && mvn checkstyle:check -q 2>&1 | head -30 || echo "⏭️ Maven 未找到"

# SpotBugs
which mvn && mvn spotbugs:check -q 2>&1 | head -30 || echo "⏭️ SpotBugs 未配置"

# 代码气味快速扫描
grep -rn "System\.out\.print\|e\.printStackTrace()\|TODO\|FIXME\|catch (Exception" \
  --include="*.java" . 2>/dev/null | grep -v "test\|Test" | head -20

# 检测潜在的 N+1（循环中调用 Repository）
grep -n "for\|stream\(\)" --include="*.java" -rn . 2>/dev/null | head -10
```

## 第 4 阶段：深度 Java 专项审查

**🔴 CRITICAL — 安全与事务**

检查每个变更文件：

| 问题类型 | 检查要点 |
|---------|---------|
| SQL 注入 | JPQL/原生 SQL 是否有字符串拼接 |
| 事务缺失 | 写操作方法是否缺少 `@Transactional` |
| 事务穿透 | 同一 Bean 内方法相互调用 `@Transactional` 失效 |
| 反序列化 | ObjectInputStream 是否反序列化不受信对象 |
| 硬编码凭证 | `password =` / `apiKey =` 等字面量 |

**🟠 HIGH — JVM/框架规范**

| 问题类型 | 检查要点 |
|---------|---------|
| JPA N+1 | Service 层循环中调用 repository.findById() |
| try-with-resources | InputStream/Connection/PreparedStatement 是否正确关闭 |
| 并发安全 | synchronized 范围 / volatile 语义 / ThreadLocal 清理 |
| Optional 滥用 | 直接 `.get()` 而非 `.orElseThrow()` |
| 宽泛异常捕获 | `catch (Exception e)` 无重抛且只打印堆栈 |
| 泛型原始类型 | `List list` 而非 `List<String> list` |

**🟡 MEDIUM — 现代 Java 实践（Java 17+）**

| 问题类型 | 检查要点 |
|---------|---------|
| 可用 Record 替代 | 只含字段和 getter 的 DTO 类 |
| 可用 Sealed 替代 | `instanceof` 链可以改为 sealed + pattern matching |
| Stream 副作用 | `forEach` 内修改外部可变集合 |
| 装箱拆箱性能 | `Stream<Integer>` 应换为 `IntStream` |
| Spring 分层混乱 | Controller 包含业务逻辑 / Service 依赖 HttpServletRequest |

**🟢 LOW — 规范风格**

| 问题类型 | 检查要点 |
|---------|---------|
| 命名规范 | 类：PascalCase / 方法/字段：camelCase / 常量：UPPER_SNAKE_CASE |
| 调试代码 | System.out.println / e.printStackTrace() |
| Javadoc | 公开 API 方法缺少说明 |
| 魔法数字 | 未定义为 `static final` 常量 |

## 第 5 阶段：Spring Boot 专项（检测到 Spring 注解时执行）

```bash
grep -rn "@Transactional\|@Service\|@Repository\|@Controller\|@RestController\|@Lazy\|@Scope" \
  --include="*.java" . 2>/dev/null | head -30
```

额外检查：
- Bean Scope：单例 Bean 是否注入了 prototype scope 的 Bean（应使用 `@Lookup` 或 `ObjectProvider`）
- Controller 层是否直接调用 DAO（应通过 Service 层）
- `@Value` 属性是否有默认值（`@Value("${key:defaultValue}")`）
- `@Async` 方法是否在同一 Bean 内调用（代理穿透）

## 第 6 阶段：输出完整报告

```markdown
# Java 代码审查报告

**审查时间**：<当前时间>
**Java 版本**：<从 pom.xml/build.gradle 检测>
**框架**：Spring Boot <版本> / 纯 Java
**变更文件**：N 个 .java 文件

## 静态分析结果

| 工具 | 结果 |
|------|------|
| Checkstyle | ✅ / ❌ N 个问题 / ⏭️ 未配置 |
| SpotBugs | ✅ / ❌ N 个问题 / ⏭️ 未配置 |

## 发现问题

### 🔴 CRITICAL（安全 + 事务）
（无 / 列出：位置 + 分类 + 描述 + 修复建议 + 代码示例）

### 🟠 HIGH（JVM/框架规范）
（无 / 列出问题）

### 🟡 MEDIUM（现代 Java 实践）
（无 / 列出问题）

### 🟢 LOW（规范风格）
（无 / 列出问题）

## 综合评分

| 维度 | 得分 |
|------|------|
| 安全性（含事务） | xx/25 |
| Spring/JPA 规范 | xx/25 |
| 并发正确性 | xx/25 |
| 现代 Java 实践 | xx/25 |
| **总分** | **xx/100（等级 X）** |

## 审查结论

✅ APPROVE / ⚠️ WARNING / ❌ BLOCK

## 推荐阅读（针对本次发现问题）

（根据实际问题动态推荐）
- JPA N+1：https://vladmihalcea.com/n-plus-1-query-problem/
- try-with-resources：JLS §14.20.3
- Java 17 Records：JEP 395
```

---

## 关联命令

- `/g-review` — 通用多语言审查（包含 Java 基础审查）
- `/g-plan` — 先规划 Java 重构方案
