---
title: Java 常用数据结构与工具类速查
date: 2026-03-30 10:30:00
tags: [Java, 数据结构, 学习笔记]
categories: [技术, Java]
description: Java 常用集合类、字符串操作及输入输出快速参考指南
cover: /img/139917560_p14.png
---

# Java 常用数据结构与工具类速查

> 本文整理了 Java 开发中常用的数据结构、集合类和工具方法，适合快速查阅和复习。

---

##  一、HashMap（哈希映射）

**特点**：Key-Value 存储，Key 唯一，通过 Key 快速查找 Value

```java
import java.util.HashMap;

HashMap<String, Integer> map = new HashMap<>();

// 添加键值对
map.put("Alice", 25);

// 获取值
int age = map.get("Alice");  // 返回 25

// 检查是否存在指定 Key
boolean hasKey = map.containsKey("Bob");  // false

// 删除键值对
map.remove("Bob");
```

---

##  二、HashSet（哈希集合）

**特点**：存储唯一元素，无重复，无顺序，只能判断"是否存在"

```java
import java.util.HashSet;

HashSet<String> set = new HashSet<>();

// 添加元素
set.add("Apple");

// 判断元素是否存在
boolean exists = set.contains("Apple");  // true

// 删除元素
set.remove("Banana");
```

---

##  三、List 列表

**特点**：有序、可重复的集合接口

```java
import java.util.ArrayList;
import java.util.List;

List<String> list = new ArrayList<>();

// 增 - 末尾添加
list.add("Apple");

// 删 - 删除指定元素（第一个匹配）
list.remove("Apple");

// 改 - 修改指定位置元素
list.set(0, "Orange");

// 查 - 获取指定位置元素
String first = list.get(0);
```

---

##  四、Arrays 数组工具类

**用于操作数组的各种实用方法**

```java
import java.util.Arrays;

// 数组升序排序
Arrays.sort(array);

// 将数组全部填充为指定值
Arrays.fill(memo, -1);

// 判断两个数组是否相同
boolean isEqual = Arrays.equals(array1, array2);
```

---

##  五、栈（Stack）

**特点**：后进先出（LIFO）

```java
import java.util.ArrayDeque;
import java.util.Deque;

Deque<String> st = new ArrayDeque<>();

// 入栈
st.push("First");
st.push("Second");

// 查看栈顶元素（不移除）
String top = st.peek();  // "Second"

// 移除并返回栈顶元素
String popped = st.pop();  // "Second"

// 检查栈是否为空
boolean empty = st.isEmpty();
```

---

##  六、Scanner 输入

**接收用户输入的数据**

```java
import java.util.Scanner;

Scanner scanner = new Scanner(System.in);

// 读取整数
int x1 = scanner.nextInt();

// 读取字符串
String str = scanner.next();

// 读取一行
String line = scanner.nextLine();
```

---

##  七、字符与字符串操作

### 字符串转字符数组

```java
char[] chars = str.toCharArray();
```

### 字符判断

```java
// 判断字符是否为字母或数字
boolean isLetterOrDigit = Character.isLetterOrDigit(ch);
```

### 提取字符

```java
// 提取字符串中的第 i 个字符
char c = s.charAt(i);
```

### 大小写转换

```java
// 将字符转换为小写（数字、符号原样返回）
char lower = Character.toLowerCase(ch);
```

---

##   总结

| 数据结构 | 特点 | 主要用途 |
|---------|------|---------|
| **HashMap** | Key-Value，Key 唯一 | 快速查找、映射关系 |
| **HashSet** | 元素唯一，无序 | 去重、存在性判断 |
| **List** | 有序，可重复 | 序列数据、索引访问 |
| **Stack** | 后进先出 | 递归、回溯、表达式求值 |
| **Queue** | 先进先出 | 队列处理、广度优先搜索 |
| **Arrays** | 数组工具类 | 排序、填充、比较 |

---

## 💡 使用建议

1. **需要 Key-Value 映射时** → 选择 `HashMap`
2. **需要去重时** → 选择 `HashSet`
3. **需要有序且可重复** → 选择 `List`（如 `ArrayList`）
4. **需要后进先出** → 选择 `Stack`（使用 `Deque` 实现）
5. **需要先进先出** → 选择 `Queue`（使用 `ArrayDeque` 实现）
6. **数组操作** → 使用 `Arrays` 工具类

---

>  **提示**：本文档可作为日常开发的快速参考，建议收藏备用。
