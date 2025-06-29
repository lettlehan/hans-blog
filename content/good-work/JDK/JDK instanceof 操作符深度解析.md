---
title: JDK instanceof 操作符深度解析
date: {{ .Date }}
tags: [Java, JDK, 模式匹配]
description: 全面解析Java instanceof操作符及其演进，包含传统用法、模式匹配、性能分析和最佳实践
---

## 1. 基础语法

### 1.1 传统用法

```java
if (obj instanceof String) {
    String str = (String) obj;
    System.out.println(str.length());
}
```

### 1.2 模式匹配（JDK16+）

```java
if (obj instanceof String str) {
    System.out.println(str.length());
}
```

## 2. 类型模式匹配

### 2.1 基本模式

```java
public void process(Object obj) {
    if (obj instanceof Integer i && i > 0) {
        System.out.println("正整数: " + i);
    }
}
```

### 2.2 switch表达式结合（JDK21+）

```java
String formatted = switch (obj) {
    case Integer i -> String.format("int %d", i);
    case Long l    -> String.format("long %d", l);
    case null      -> "null";
    default        -> obj.toString();
};
```

## 3. 性能分析 {#performance}

### 3.1 字节码对比

| 方式                | 字节码指令数 | 性能影响 |
|---------------------|-------------|---------|
| 传统instanceof      | 5-7         | 较高     |
| 模式匹配            | 3-5         | 较低     |

### 3.2 优化建议

1. **优先使用模式匹配语法**：减少类型转换操作
2. **避免多层嵌套检查**：改用switch表达式简化逻辑
3. **缓存频繁检查结果**：对热点路径考虑缓存instanceof结果

## 4. 最佳实践

### 4.1 防御性编程

```java
public void safeProcess(Object obj) {
    if (!(obj instanceof Number num)) {
        throw new IllegalArgumentException("需要数字类型");
    }
    // 使用num...
}
```

### 4.2 模式变量作用域

```java
if (obj instanceof String s1) {
    // s1在此可见
} else if (obj instanceof Integer i) {
    // i在此可见，s1不可见
}
```

## 5. 常见陷阱

### 5.1 空值处理

```java
// 传统安全写法
if (obj != null && obj instanceof String s) {
    // ...
}

// JDK16+简化写法
if (obj instanceof String s) {
    // 自动处理null
}
```

### 5.2 模式变量遮蔽

```java
String s = "外部";
if (obj instanceof String s) {  // 编译错误
    // 变量名s被遮蔽
}
```

## 6. 高级用法

### 6.1 记录模式（JDK21+）

```java
record Point(int x, int y) {}

if (obj instanceof Point(int x, int y)) {
    System.out.println(x + "," + y);
}
```

### 6.2 泛型类型检查

```java
public <T> void checkList(Object obj) {
    if (obj instanceof List<?> list) {
        // 原始类型检查
    }
    if (obj instanceof List<String> list) {
        // 具体类型检查（运行时擦除）
    }
}
```

## 7. 实际案例

### 7.1 解析JSON节点

```java
public void parseJson(JsonNode node) {
    if (node instanceof ObjectNode objNode) {
        // 处理对象节点
    } else if (node instanceof ArrayNode arrNode) {
        // 处理数组节点
    }
}
```

### 7.2 处理异构集合

```java
List<Object> mixedList = ...;
for (Object item : mixedList) {
    if (item instanceof String s) {
        processString(s);
    } else if (item instanceof Number n) {
        processNumber(n);
    }
}
```

## 8. 版本兼容性

| JDK版本 | 特性支持               |
|--------|-----------------------|
| 1.0+   | 基础instanceof         |
| 16+    | 模式匹配               |
| 21+    | switch模式匹配         |
| 21+    | 记录模式               |

{{< details "点击查看实现原理" >}}
instanceof的实现依赖于Java虚拟机的checkcast指令，模式匹配在编译阶段会进行类型推断和变量绑定优化。
{{< /details >}}

{{< button href="#performance" >}}跳转到性能分析{{< /button >}}
