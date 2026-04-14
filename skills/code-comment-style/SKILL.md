---
name: code-comment-style
description: >-
  Use when writing, refactoring, reviewing, or documenting code where comments should be improved.
  Prefer structured, intent-focused comments with Chinese business-context style when appropriate,
  including section headers for major phases, tagged comments for risks/performance/business constraints,
  Javadoc for public methods, and concise inline notes for non-obvious logic.
  Avoid noisy comments that only repeat syntax.
---

# Code Comment Style

目标：让代码注释像一份“小型交接文档”，人和 AI 下次读到时能快速理解业务意图、风险点和不能乱改的地方。

## 核心风格

优先写“为什么”和“这里有什么坑”，而不是复述“代码在做什么”。

推荐风格：

1. 用分段标题标出主流程。
2. 用短注释解释关键业务规则。
3. 用“核心 / 排雷 / 性能 / 业务 / 兼容”这类标签提醒维护者。
4. 对公共方法、复杂校验、批处理、查重、缓存、状态回写写清楚输入、输出和副作用。
5. 不给简单赋值、普通 getter/setter、显而易见的 if/for 贴废话注释。

## 什么时候必须补注释

看到下面场景时，主动补关键注释：

1. 复杂业务校验、逐级断言、状态流转。
2. 为了前端展示、SQL 分组统计、后续批处理而写的特殊字段。
3. 清空旧值、强制置空、绕过默认 ORM 策略等容易被误删的“排雷逻辑”。
4. 本地缓存、批量分组、查重、重试、锁、并发、性能优化。
5. 复杂 SQL、动态 SQL、非直观 join、数据修复脚本。
6. 日期时间、编码、路径、代理、操作系统差异、兼容性处理。
7. 外部接口、第三方服务、文件导入导出、权限差异。
8. AI 或维护者很可能“看起来能简化，其实不能简化”的代码。

## 推荐注释模板

### 1. 主流程分段

适合长方法、导入流程、校验流程、任务状态流转。

```java
// ==========================================
// 1. 验证：资源目录（逐级断言，精准报错）
// ==========================================
```

### 2. 排雷核心

适合容易被未来维护者误删的关键逻辑。

```java
// 【排雷核心】：校验开始前必须清空旧解析结果，避免上一次错误状态残留到本次保存。
```

### 3. 性能核心

适合缓存、批量处理、避免数据库 IO 的逻辑。

```java
// 【性能核心】：从 JVM 本地缓存读取目录节点，避免导入校验时逐行打数据库。
```

### 4. 业务断言

适合“为什么必须这样判断”的业务规则。

```java
// 业务要求必须选择到叶子分类，否则后续任务分解无法定位具体资源责任单位。
```

### 5. 副作用说明

适合会回写实体、改变状态、影响前端展示或统计的逻辑。

```java
// 重复错误也要写入 errorJson，否则前端只有 isDuplicate 标记时无法展示具体原因。
```

## Javadoc 风格

公共方法、服务入口、复杂私有方法建议写 Javadoc。

写清楚：

1. 方法解决什么业务问题。
2. 参数是否允许为空。
3. 返回值代表什么。
4. 是否会修改入参对象或回写状态。

示例：

```java
/**
 * 校验单条需求明细，并把解析结果与异常状态回写到 item。
 *
 * @param item 当前需要校验的明细项，会被回写目录ID、异常状态和错误JSON
 * @param allItemsInTask 当前任务下的所有明细；批量导入时用于任务内查重，单条校验可为空
 * @return 校验结果，包含字段级错误和解析出的目录ID
 */
```

## 注释语言

1. 业务系统、排障文档、中文团队项目：优先中文注释。
2. 开源库、英文既有项目：沿用英文注释。
3. 文件里已有明确风格：跟随原风格，不强行切换语言。

## 不要这样写

避免这种只复述语法的注释：

```java
// 设置状态为1
item.setValidationStatus(1);
```

如果状态值本身不直观，应该写成：

```java
// 1 表示校验通过；这里清空 errorJson，避免前端继续展示历史错误。
item.setValidationStatus(1);
```

## 修改代码时的执行规则

1. 新增复杂逻辑时，同步补关键注释。
2. 改动已有逻辑时，检查附近注释是否已经过期。
3. 发现误导性注释时，优先更新或删除。
4. 不为了“显得注释多”而堆注释。
5. 注释要能帮助未来 AI agent 继续排查、修改或验证。


