---
title: JDK 8新特性全解析
date: {{ .Date }}
tags: [Java, JDK8, Lambda, Stream, 函数式编程]
description: 深入解析Java 8引入的重要特性及其实际应用，包括Lambda表达式、Stream API、新日期时间API等
toc: true
---

## 1. Lambda表达式

Lambda表达式是Java 8引入的最重要特性，它允许我们将函数作为方法参数传递，使代码更加简洁和灵活。

### 1.1 基本语法

```java
// 基本语法: (参数) -> { 表达式 }

// 无参数示例
Runnable r1 = () -> System.out.println("Hello Lambda!");

// 单参数示例 (可省略参数括号)
Consumer<String> c1 = s -> System.out.println(s);

// 多参数示例
Comparator<Integer> c2 = (a, b) -> a.compareTo(b);

// 带代码块的Lambda
Runnable r2 = () -> {
    System.out.println("多行代码");
    System.out.println("使用代码块");
};
```

### 1.2 变量捕获

Lambda表达式可以捕获其外部作用域中的变量，但这些变量必须是effectively final（事实上的最终变量）。

```java
String prefix = "User: ";  // 事实上的final变量
Consumer<String> printer = name -> System.out.println(prefix + name);
printer.accept("John");  // 输出: User: John

// 以下代码会编译错误，因为Lambda中使用的外部变量必须是effectively final
String counter = "Count: ";
Consumer<Integer> counter_printer = n -> {
    // counter = "Total: ";  // 这行会导致编译错误
    System.out.println(counter + n);
};
```

## 2. 函数式接口

函数式接口是只包含一个抽象方法的接口，可以使用Lambda表达式创建其实例。Java 8在`java.util.function`包中提供了许多标准函数式接口。

### 2.1 常用函数式接口

<div class="grid cards" markdown>

-   **Consumer\<T\>**
    - 接受一个输入参数并且没有返回值
    - `void accept(T t)`
    - 例如：打印、发送消息

-   **Supplier\<T\>**
    - 不接受参数但返回一个结果
    - `T get()`
    - 例如：工厂方法、生成随机值

-   **Function\<T, R\>**
    - 接受一个输入参数，返回一个结果
    - `R apply(T t)`
    - 例如：类型转换、数据映射

-   **Predicate\<T\>**
    - 接受一个参数，返回布尔值
    - `boolean test(T t)`
    - 例如：过滤、验证

</div>

### 2.2 自定义函数式接口

使用`@FunctionalInterface`注解可以确保接口是函数式接口。

```java
@FunctionalInterface
public interface Converter<F, T> {
    T convert(F from);
    
    // 可以包含默认方法
    default void printInfo() {
        System.out.println("This is a converter");
    }
}

// 使用Lambda实现自定义函数式接口
Converter<String, Integer> stringToInt = s -> Integer.parseInt(s);
Integer value = stringToInt.convert("123");  // 返回整数123
```

## 3. 方法引用

方法引用提供了一种更简洁的Lambda表达式写法，通过引用已有方法来创建函数式接口实例。

### 3.1 方法引用类型

```java
// 静态方法引用: ClassName::staticMethod
Function<String, Integer> parser = Integer::parseInt;

// 实例方法引用: instance::method
String text = "Hello";
Supplier<Integer> lengthSupplier = text::length;

// 特定类型的实例方法引用: ClassName::method
Function<String, Integer> lengthFunc = String::length;

// 构造方法引用: ClassName::new
Supplier<ArrayList<String>> listCreator = ArrayList::new;
```

### 3.2 实际应用

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

// 使用Lambda表达式
names.forEach(name -> System.out.println(name));

// 使用方法引用，更加简洁
names.forEach(System.out::println);

// 排序示例
List<Person> people = getPeopleList();
// Lambda表达式
people.sort((p1, p2) -> p1.getName().compareTo(p2.getName()));
// 方法引用
people.sort(Comparator.comparing(Person::getName));
```

## 4. 默认方法

Java 8允许在接口中定义默认方法（带有实现的方法），这使得接口可以演化而不破坏现有实现。

### 4.1 基本用法

```java
public interface Vehicle {
    // 抽象方法
    void accelerate();
    
    // 默认方法
    default void brake() {
        System.out.println("Default brake implementation");
    }
    
    // 静态方法
    static Vehicle create() {
        return new Car();
    }
}

// 实现类可以选择覆盖默认方法
public class Car implements Vehicle {
    @Override
    public void accelerate() {
        System.out.println("Car is accelerating");
    }
    
    // 可以选择不覆盖brake()方法
}
```

### 4.2 多重继承问题

当一个类实现多个接口，且这些接口包含相同签名的默认方法时，会产生冲突。

```java
public interface A {
    default void hello() {
        System.out.println("Hello from A");
    }
}

public interface B {
    default void hello() {
        System.out.println("Hello from B");
    }
}

// 必须明确指定使用哪个默认方法实现
public class C implements A, B {
    @Override
    public void hello() {
        // 选择使用A的实现
        A.super.hello();
        // 或自定义实现
        System.out.println("Hello from C");
    }
}
```

## 5. Stream API

Stream API提供了一种函数式编程方式来处理集合数据，支持串行和并行操作，使数据处理更加高效和简洁。

### 5.1 创建Stream

```java
// 从集合创建
List<String> list = Arrays.asList("a", "b", "c");
Stream<String> streamFromList = list.stream();

// 从数组创建
String[] array = {"a", "b", "c"};
Stream<String> streamFromArray = Arrays.stream(array);

// 使用Stream.of
Stream<String> streamFromValues = Stream.of("a", "b", "c");

// 创建无限流
Stream<Integer> infiniteStream = Stream.iterate(0, n -> n + 1);
Stream<Double> randomStream = Stream.generate(Math::random);
```

### 5.2 中间操作

中间操作返回一个新的Stream，可以链式调用多个中间操作。

```java
List<String> names = Arrays.asList("John", "Jane", "Adam", "Tom", "Alice");

// filter: 过滤元素
Stream<String> filtered = names.stream().filter(name -> name.startsWith("J"));

// map: 转换元素
Stream<Integer> lengths = names.stream().map(String::length);

// sorted: 排序
Stream<String> sorted = names.stream().sorted();

// distinct: 去重
Stream<String> distinct = names.stream().distinct();

// limit: 限制数量
Stream<String> limited = names.stream().limit(3);

// skip: 跳过元素
Stream<String> skipped = names.stream().skip(2);

// flatMap: 扁平化嵌套集合
List<List<String>> nestedList = Arrays.asList(
    Arrays.asList("a", "b"), 
    Arrays.asList("c", "d")
);
Stream<String> flatStream = nestedList.stream().flatMap(Collection::stream);
```

### 5.3 终端操作

终端操作会触发Stream的计算并返回结果。

```java
List<String> names = Arrays.asList("John", "Jane", "Adam", "Tom", "Alice");

// forEach: 遍历元素
names.stream().forEach(System.out::println);

// collect: 收集结果到集合
List<String> filteredList = names.stream()
    .filter(name -> name.length() > 3)
    .collect(Collectors.toList());

// reduce: 归约操作
Optional<String> concatenated = names.stream()
    .reduce((a, b) -> a + ", " + b);

// count: 计数
long count = names.stream().count();

// anyMatch/allMatch/noneMatch: 匹配操作
boolean anyStartsWithJ = names.stream().anyMatch(name -> name.startsWith("J"));
boolean allLongerThan2 = names.stream().allMatch(name -> name.length() > 2);
boolean noneStartsWithZ = names.stream().noneMatch(name -> name.startsWith("Z"));

// findFirst/findAny: 查找元素
Optional<String> first = names.stream().findFirst();
Optional<String> any = names.stream().findAny();

// min/max: 最小/最大值
Optional<String> shortest = names.stream()
    .min(Comparator.comparing(String::length));
```

### 5.4 并行流

Stream API支持并行处理，可以充分利用多核处理器。

```java
// 创建并行流
Stream<String> parallelStream = names.parallelStream();

// 或将顺序流转换为并行流
Stream<String> parallel = names.stream().parallel();

// 并行处理示例
long count = names.parallelStream()
    .filter(name -> name.length() > 3)
    .count();
```

## 6. Optional类

Optional类是一个容器对象，可以包含或不包含非空值，用于避免空指针异常。

### 6.1 创建Optional

```java
// 创建空Optional
Optional<String> empty = Optional.empty();

// 创建包含值的Optional（值不能为null）
Optional<String> opt = Optional.of("Hello");

// 创建可能包含null的Optional
String nullableValue = null;
Optional<String> optOrNull = Optional.ofNullable(nullableValue);
```

### 6.2 使用Optional

```java
Optional<String> opt = Optional.of("Hello");

// isPresent: 检查是否存在值
if (opt.isPresent()) {
    System.out.println("Value found");
}

// get: 获取值（如果为空会抛出NoSuchElementException）
String value = opt.get();

// orElse: 提供默认值
String result = opt.orElse("Default");

// orElseGet: 提供默认值的Supplier
String lazyResult = opt.orElseGet(() -> computeDefaultValue());

// orElseThrow: 值不存在时抛出异常
String valueOrException = opt.orElseThrow(() -> new RuntimeException("Value not found"));

// ifPresent: 值存在时执行操作
opt.ifPresent(System.out::println);

// filter: 过滤值
Optional<String> filtered = opt.filter(s -> s.length() > 3);

// map: 转换值
Optional<Integer> length = opt.map(String::length);

// flatMap: 转换为Optional
Optional<String> upperCase = opt.flatMap(s -> Optional.of(s.toUpperCase()));
```

### 6.3 实际应用

```java
// 传统方式处理可能为null的值
public String getCarInsuranceName(Person person) {
    if (person != null) {
        Car car = person.getCar();
        if (car != null) {
            Insurance insurance = car.getInsurance();
            if (insurance != null) {
                return insurance.getName();
            }
        }
    }
    return "Unknown";
}

// 使用Optional的方式
public String getCarInsuranceNameWithOptional(Optional<Person> person) {
    return person.flatMap(Person::getCarOptional)
                .flatMap(Car::getInsuranceOptional)
                .map(Insurance::getName)
                .orElse("Unknown");
}
```

## 7. 新的日期时间API

Java 8引入了新的日期时间API（`java.time`包），解决了旧API的设计缺陷，提供了不可变、线程安全的日期时间类。

### 7.1 核心类

<div class="grid cards" markdown>

-   **LocalDate**
    - 表示日期（年月日）
    - 不包含时间和时区信息
    - 例如：2023-05-15

-   **LocalTime**
    - 表示时间（时分秒纳秒）
    - 不包含日期和时区信息
    - 例如：14:30:45.123

-   **LocalDateTime**
    - 表示日期和时间
    - 不包含时区信息
    - 例如：2023-05-15T14:30:45.123

-   **ZonedDateTime**
    - 表示带时区的日期和时间
    - 例如：2023-05-15T14:30:45.123+08:00[Asia/Shanghai]

</div>

### 7.2 基本用法

```java
// 创建当前日期时间
LocalDate today = LocalDate.now();
LocalTime now = LocalTime.now();
LocalDateTime dateTime = LocalDateTime.now();
ZonedDateTime zonedDateTime = ZonedDateTime.now();

// 创建特定日期时间
LocalDate date = LocalDate.of(2023, 5, 15);

// 日期计算
LocalDate tomorrow = today.plusDays(1);
LocalDate lastMonth = today.minusMonths(1);
LocalDate firstDayOfMonth = today.withDayOfMonth(1);

// 日期比较
boolean isBefore = date1.isBefore(date2);
Period period = Period.between(date1, date2);

// 格式化
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
String formattedDate = today.format(formatter);
LocalDate parsedDate = LocalDate.parse("2023-05-15", formatter);
```

## 8. JavaScript引擎Nashorn

Java 8引入了新的JavaScript引擎Nashorn，替代了旧的Rhino引擎，提供了更好的性能和更多的功能。

```java
// 创建ScriptEngine
ScriptEngine engine = new ScriptEngineManager().getEngineByName("nashorn");

// 执行JavaScript代码
engine.eval("print('Hello from JavaScript!');");

// 在JavaScript中调用Java
engine.eval("var ArrayList = Java.type('java.util.ArrayList')");
engine.eval("var list = new ArrayList()");
engine.eval("list.add('Item 1')");

// 在Java中调用JavaScript函数
engine.eval("function greet(name) { return 'Hello, ' + name; }");
Invocable invocable = (Invocable) engine;
String result = (String) invocable.invokeFunction("greet", "John");
```

## 9. Base64编码

Java 8内置了Base64编码的支持，不再需要使用第三方库。

```java
// 基本编码
String text = "Hello, World!";
String encoded = Base64.getEncoder().encodeToString(text.getBytes());
byte[] decoded = Base64.getDecoder().decode(encoded);

// URL安全编码
String urlEncoded = Base64.getUrlEncoder().encodeToString(text.getBytes());
byte[] urlDecoded = Base64.getUrlDecoder().decode(urlEncoded);

// MIME编码
String mimeEncoded = Base64.getMimeEncoder().encodeToString(text.getBytes());
byte[] mimeDecoded = Base64.getMimeDecoder().decode(mimeEncoded);
```

## 10. 其他改进

### 10.1 集合API增强

```java
// Map接口的新方法
Map<String, Integer> map = new HashMap<>();

// putIfAbsent: 仅当键不存在时才放入值
map.putIfAbsent("key", 100);

// computeIfAbsent: 仅当键不存在时计算并放入值
map.computeIfAbsent("key", k -> k.length());

// computeIfPresent: 仅当键存在时计算并更新值
map.computeIfPresent("key", (k, v) -> v + 10);

// forEach: 遍历键值对
map.forEach((k, v) -> System.out.println(k + ": " + v));
```

### 10.2 并发API增强

```java
// CompletableFuture: 支持异步编程
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // 异步执行的代码
    return "Result";
});

future.thenAccept(System.out::println);
future.thenApply(s -> s + " processed");
future.thenCombine(otherFuture, (s1, s2) -> s1 + s2);

// StampedLock: 提供乐观读锁
StampedLock lock = new StampedLock();
long stamp = lock.tryOptimisticRead();
// 读取共享数据
if (!lock.validate(stamp)) {
    // 乐观读失败，获取读锁
    stamp = lock.readLock();
    try {
        // 重新读取共享数据
    } finally {
        lock.unlockRead(stamp);
    }
}
```

### 10.3 IO/NIO改进

```java
// Files类的新方法
// 读取所有行
List<String> lines = Files.readAllLines(Paths.get("file.txt"));

// 使用Stream读取行
try (Stream<String> lineStream = Files.lines(Paths.get("file.txt"))) {
    lineStream.forEach(System.out::println);
}

// 遍历目录
try (Stream<Path> pathStream = Files.walk(Paths.get("."))) {
    pathStream.filter(Files::isRegularFile)
              .forEach(System.out::println);
}
```

## 11. 总结

Java 8引入了许多重要的新特性，使Java编程更加现代化和高效：

1. **Lambda表达式和函数式接口**：简化代码，支持函数式编程
2. **Stream API**：提供强大的集合数据处理能力
3. **Optional类**：更优雅地处理null值
4. **新的日期时间API**：解决了旧API的设计缺陷
5. **默认方法**：允许接口演化而不破坏现有实现
6. **方法引用**：提供更简洁的Lambda表达式写法
7. **Nashorn JavaScript引擎**：更好的Java与JavaScript集成
8. **Base64编码**：内置支持，无需第三方库
9. **集合API增强**：更多实用方法
10. **并发API增强**：更强大的异步编程支持

这些特性共同推动了Java向函数式编程方向发展，使代码更加简洁、可读和高效，同时保持了Java的强类型和面向对象特性。
