---
title: JDK instanceof 操作符深度解析
date: {{ .Date }}
author: 韓小han
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

## 3. 性能分析 

### 3.1 字节码对比

**不同方式的性能表现**

| 方式                | 字节码指令数 | 性能影响 |
|---------------------|-------------|---------|
| 传统instanceof      | 5-7         | 较高     |
| 模式匹配            | 3-5         | 较低     |

## 4. 最佳实践

### 4.2 模式变量作用域

```java
if (obj instanceof String s1) {
    // s1在此可见
} else if (obj instanceof Integer i) {
    // i在此可见，s1不可见
}
```

## 5. 版本兼容性

**各版本特性支持**

| JDK版本 | 特性支持               |
|--------|-----------------------|
| 1.0+   | 基础instanceof         |
| 16+    | 模式匹配               |
| 21+    | switch模式匹配         |
| 21+    | 记录模式               |

<details>
<summary>点击查看实现原理</summary>
instanceof的实现依赖于Java虚拟机的checkcast指令，模式匹配在编译阶段会进行类型推断和变量绑定优化。
</details>

[跳转到性能分析](#performance)
