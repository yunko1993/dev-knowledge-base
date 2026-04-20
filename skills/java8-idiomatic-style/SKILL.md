---
name: java8-idiomatic-style
description: >-
  Use when writing, refactoring, reviewing, or modernizing Java code that should target Java 8 style.
  Prefer safe IntelliJ IDEA inspection suggestions and Java 8 idioms such as Map.computeIfAbsent,
  getOrDefault, removeIf, streams for clear transformations, method references, Optional where it
  improves null-handling, and String.trim().isEmpty() over length checks. Avoid overusing Optional,
  streams, or clever one-liners when plain code is clearer.
---

# Java 8 Idiomatic Style

目标：让 Codex 写出的 Java 代码更接近 IDEA 会推荐的现代 Java 8 风格，减少 review 时被 IDE 提出一堆“可替换为更简洁写法”的问题。

## 核心原则

优先采用安全、清晰、IDEA 认可的 Java 8 写法。

但不要为了“现代”而强行炫技。可读性、业务语义和调试便利优先。

## 优先采用的写法

### 1. Map 初始化与分组

遇到“先 get，null 时 new，再 put”的模式，优先使用 `computeIfAbsent`。

推荐：

```java
List<Book> books =
        booksByCategory.computeIfAbsent(categoryCode, key -> new ArrayList<>());
books.add(book);
```

不推荐：

```java
List<Book> books = booksByCategory.get(categoryCode);
if (books == null) {
    books = new ArrayList<>();
    booksByCategory.put(categoryCode, books);
}
books.add(book);
```

### 2. 默认值读取

简单默认值读取优先使用 `getOrDefault`。

```java
List<Student> students = studentsByClass.getOrDefault(classId, Collections.emptyList());
```

### 3. 条件删除

遍历集合并按条件删除时，优先使用 `removeIf`。

```java
students.removeIf(student -> student.getStudentNo() == null);
```

### 4. 字符串空值判断

项目里如果已经引入 Apache Commons Lang，优先使用 `StringUtils.isBlank/isNotBlank`。

没有工具类时，在 Java 8 中可用：

```java
value == null || value.trim().isEmpty()
```

不要写：

```java
value == null || value.trim().length() == 0
```

注意：`String.isBlank()` 是 Java 11，不是 Java 8。

### 5. Stream 与 Collectors

筛选、映射、分组、去重、收集结果时，可以使用 Stream。

推荐用于：

1. `filter + collect`
2. `map + collect`
3. `groupingBy`
4. `toMap` 且明确重复 key 合并策略
5. 简单 `distinct` / `sorted`

示例：

```java
Map<Long, List<Student>> studentsByClass = students.stream()
        .filter(student -> student.getClassId() != null)
        .collect(Collectors.groupingBy(Student::getClassId));
```

使用 `toMap` 时必须处理重复 key 风险：

```java
Map<Long, Student> studentMap = students.stream()
        .collect(Collectors.toMap(
                Student::getId,
                Function.identity(),
                (oldValue, newValue) -> oldValue
        ));
```

### 6. Optional

`Optional` 适合表达“这个值可能不存在”的局部转换。

可以用于：

1. 避免短链路 null 判断。
2. 给返回值表达可选语义。
3. 简单默认值转换。

谨慎使用：

1. 不要把实体字段定义成 `Optional`。
2. 不要把方法参数定义成 `Optional`。
3. 不要为了替换一个简单 `if` 写出更绕的 Optional 链。
4. 有业务异常、复杂分支、日志记录时，普通 `if` 更清楚。

推荐：

```java
return Optional.ofNullable(value)
        .map(String::trim)
        .filter(StringUtils::isNotBlank)
        .orElse(null);
```

不推荐：

```java
Optional.ofNullable(item).ifPresent(i -> doManySideEffects(i));
```

## Review 时主动检查

写完 Java 代码后，主动用 IDEA inspection 的视角自查：

1. 是否有 `map.get()` 后 null 初始化，可以改为 `computeIfAbsent`。
2. 是否有 `trim().length() == 0`，可以改为 `trim().isEmpty()` 或 `StringUtils.isBlank`。
3. 是否有手写集合过滤/转换，适合改为 Stream。
4. 是否有遍历删除，适合改为 `removeIf`。
5. 是否有 `if (x == null) return default`，适合改为 `Objects.requireNonNullElse` 不行，因为这是 Java 9；Java 8 用普通 if 或 `Optional`。
6. 是否引入了 Java 9+ API，必须避免。

## 不要过度现代化

不要为了少几行代码牺牲可读性。

保留普通 `for` / `if` 的场景：

1. 逻辑有多次状态变更。
2. 需要清晰打日志。
3. 有多个异常分支。
4. 业务规则很多，Stream 会变成很长链式调用。
5. 需要调试每一步中间值。

## 与注释风格配合

使用本 skill 改写代码后，如果引入了 `computeIfAbsent`、复杂 Stream、分组、合并策略或 Optional 链，应配合 `code-comment-style` 补一句业务目的或风险说明。

例如：

```java
// 按分类编码聚合同类图书，后续统一生成分类维度的展示数据。
List<Book> books =
        booksByCategory.computeIfAbsent(categoryCode, key -> new ArrayList<>());
```
