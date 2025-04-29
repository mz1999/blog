# 深入理解 Java Lambda 表达式

## 引言  
在 Java 8 之前，方法无法直接作为值传递，开发者需要通过匿名类或冗长的接口实现来实现行为参数化。Java 8 引入的 **Lambda 表达式**和**方法引用**彻底改变了这一局面，让函数式编程范式在 Java 中落地生根。本文将从行为参数化的设计思想出发，系统讲解 Lambda 的核心概念、语法特性及其在实践中的应用。

---

## 一、行为参数化：函数式编程的基石  
### 1.1 什么是行为参数化？  
**行为参数化（Behavior Parameterization）** 是指将代码逻辑（即“行为”）作为参数传递给其他方法的能力。这种设计允许方法的执行逻辑动态变化，从而提高代码的灵活性和复用性。

例如，一个筛选苹果的方法 `filterApples`，其筛选条件可以是颜色、重量或其他属性。通过行为参数化，我们无需为每种条件编写独立的方法，而是将条件逻辑抽象为接口传递：

```java
public interface ApplePredicate {
    boolean test(Apple apple);
}

// 筛选逻辑由调用者决定
List<Apple> filterApples(List<Apple> inventory, ApplePredicate predicate) {
    List<Apple> result = new ArrayList<>();
    for (Apple apple : inventory) {
        if (predicate.test(apple)) {
            result.add(apple);
        }
    }
    return result;
}
```

### 1.2 从匿名类到 Lambda  
在 Java 8 之前，行为参数化需要通过匿名类实现，但代码臃肿且不够直观：  
```java
filterApples(inventory, new ApplePredicate() {
    @Override
    public boolean test(Apple apple) {
        return "red".equals(apple.getColor());
    }
});
```

Lambda 表达式简化了这一过程，直接传递逻辑：  
```java
filterApples(inventory, (Apple apple) -> "red".equals(apple.getColor()));
```

---

## 二、Lambda 表达式：语法与核心概念  
### 2.1 Lambda 的语法结构  
Lambda 表达式由三部分组成：**参数列表**、**箭头符号** `->` 和 **函数主体**。  
```java
// 基本语法
(参数列表) -> { 函数主体 }

// 示例：比较两个苹果的重量
Comparator<Apple> byWeight = 
    (Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight());
```

### 2.2 函数式接口  
**函数式接口（Functional Interface）** 是只包含一个抽象方法的接口。Lambda 表达式本质上是函数式接口的实例。  

Java 8 提供了 `@FunctionalInterface` 注解标识这类接口，例如：  
```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```

常见的函数式接口：  
- `Predicate<T>`：接收 `T` 类型参数，返回 `boolean`。  
- `Consumer<T>`：接收 `T` 类型参数，无返回值。  
- `Function<T, R>`：接收 `T` 类型参数，返回 `R` 类型结果。  

### 2.3 类型推断与上下文  
Lambda 的类型由上下文自动推断：  
```java
// 显式指定类型
Comparator<Apple> c1 = (Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight());

// 类型推断（编译器根据目标类型推断参数类型）
Comparator<Apple> c2 = (a1, a2) -> a1.getWeight().compareTo(a2.getWeight());
```

---

## 三、方法引用：简化 Lambda 的利器  
方法引用允许直接通过方法名替代完整的 Lambda 表达式，主要分为四类：  

### 3.1 静态方法引用  
语法：`类名::静态方法名`  
```java
Function<Integer, String> intToString = String::valueOf;
```

### 3.2 实例方法引用  
语法：`对象::实例方法名`  
```java
String str = "Hello";
Supplier<Integer> lengthSupplier = str::length;
```

### 3.3 任意对象的实例方法引用  
语法：`类名::实例方法名`（适用于 Lambda 参数作为方法调用者）  
```java
BiFunction<String, String, Integer> compareIgnoreCase = String::compareToIgnoreCase;
```

### 3.4 构造函数引用  
语法：`类名::new`  
```java
Supplier<List<String>> listSupplier = ArrayList::new;
```

---

## 四、复合 Lambda 表达式  
### 4.1 比较器链  
通过 `Comparator` 的链式调用实现多级排序：  
```java
inventory.sort(
    comparing(Apple::getWeight)
        .reversed()
        .thenComparing(Apple::getCountry)
);
```

### 4.2 谓词组合  
使用 `and`、`or`、`negate` 组合多个条件：  
```java
Predicate<Apple> redAndHeavy = 
    apple -> "red".equals(apple.getColor())
        .and(apple -> apple.getWeight() > 150);
```

### 4.3 函数组合  
通过 `andThen` 和 `compose` 实现函数串联：  
```java
Function<Integer, Integer> addOne = x -> x + 1;
Function<Integer, Integer> multiplyByTwo = x -> x * 2;

// 先加 1，再乘以 2
Function<Integer, Integer> combined1 = addOne.andThen(multiplyByTwo);
combined1.apply(3); // 结果为 8

// 先乘以 2，再加 1
Function<Integer, Integer> combined2 = addOne.compose(multiplyByTwo);
combined2.apply(3); // 结果为 7
```

---

## 五、Lambda 的实践与注意事项  
### 5.1 异常处理  
Lambda 表达式无法直接抛出受检异常（Checked Exception），需通过以下方式解决：  
1. 在函数式接口中声明异常：  
```java
@FunctionalInterface
public interface BufferedReaderProcessor {
    String process(BufferedReader br) throws IOException;
}
```

2. 在 Lambda 中捕获异常：  
```java
Function<String, Integer> safeParse = s -> {
    try {
        return Integer.parseInt(s);
    } catch (NumberFormatException e) {
        return 0;
    }
};
```

### 5.2 局部变量限制  
Lambda 可以捕获实例变量和静态变量，但局部变量必须为 `final` 或“等效 final”：  
```java
int localVar = 10;
Runnable r = () -> System.out.println(localVar); // 合法
localVar = 20; // 编译错误：localVar 必须是 final 或等效 final
```

---

## 六、总结  
Lambda 表达式是 Java 函数式编程的核心工具，通过行为参数化显著提升了代码的简洁性和灵活性。结合方法引用、函数式接口及复合操作，开发者可以构建出高度抽象且易于维护的代码结构。理解并掌握这些特性，是迈向现代 Java 开发的关键一步。