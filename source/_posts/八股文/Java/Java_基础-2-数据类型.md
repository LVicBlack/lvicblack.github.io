---
title: Java 基础 - 2.数据类型
date: 2020-08-20 15:18:03
categories: 
- 八股文
- Java
tags:
- Java
- 面试
---
# 

### 数据类型 5

#### **Q1：Java 有哪些基本数据类型？**

|**数据类型**|**内存大小**                           |**默认值**|**取值范围**                |
|------------|---------------------------------------|----------|----------------------------|
|byte        |1 B                                    |\(byte\)0 |\-128 ~ 127                 |
|short       |2 B                                    |\(short\)0|\-2^15^ ~ 2^15^\-1          |
|int         |4 B                                    |0         |\-2^31^ ~ 2^31^\-1          |
|long        |8 B                                    |0L        |\-2^63^ ~ 2^63^\-1          |
|float       |4 B                                    |0.0F      |±3.4E\+38（有效位数 6~7 位）|
|double      |8 B                                    |0.0D      |±1.7E\+308（有效位数 15 位）|
|char        |英文 1B，中文 UTF\-8 占 3B，GBK 占 2B。|'\\u0000' |'\\u0000' ~ '\\uFFFF'       |
|boolean     |单个变量 4B / 数组 1B                  |false     |true、false                 |

JVM 没有 boolean 赋值的专用字节码指令，boolean f = false 就是使用 ICONST\_0 即常数 0 赋值。单个 boolean 变量用 int 代替，boolean 数组会编码成 byte 数组。

---

#### **Q2：自动装箱/拆箱是什么？**

每个基本数据类型都对应一个包装类，除了 int 和 char 对应 Integer 和 Character 外，其余基本数据类型的包装类都是首字母大写即可。

**自动装箱：** 将基本数据类型包装为一个包装类对象，例如向一个泛型为 Integer 的集合添加 int 元素。

**自动拆箱：** 将一个包装类对象转换为一个基本数据类型，例如将一个包装类对象赋值给一个基本数据类型的变量。

比较两个包装类数值要用 equals ，而不能用 == 。

---

#### **Q3：String 是不可变类为什么值可以修改？**

String 类和其存储数据的成员变量 value 字节数组都是 final 修饰的。对一个 String 对象的任何修改实际上都是创建一个新 String 对象，再引用该对象。只是修改 String 变量引用的对象，没有修改原 String 对象的内容。

---

#### **Q4：字符串拼接的方式有哪些？**

1. 直接用 \+ ，底层用 StringBuilder 实现。只适用小数量，如果在循环中使用 \+ 拼接，相当于不断创建新的 StringBuilder 对象再转换成 String 对象，效率极差。

2.  使用 String 的 concat 方法，该方法中使用 Arrays.copyOf 创建一个新的字符数组 buf 并将当前字符串 value 数组的值拷贝到 buf 中，buf 长度 = 当前字符串长度 \+ 拼接字符串长度。之后调用 getChars 方法使用 System.arraycopy 将拼接字符串的值也拷贝到 buf 数组，最后用 buf 作为构造参数 new 一个新的 String 对象返回。效率稍高于直接使用 \+。

3. 使用 StringBuilder 或 StringBuffer，两者的 append 方法都继承自 AbstractStringBuilder，该方法首先使用 Arrays.copyOf 确定新的字符数组容量，再调用 getChars 方法使用 System.arraycopy 将新的值追加到数组中。StringBuilder 是 JDK5 引入的，效率高但线程不安全。StringBuffer 使用 synchronized 保证线程安全。

---

#### **Q5：String a = "a" \+ new String\("b"\) 创建了几个对象？**

常量和常量拼接仍是常量，结果在常量池，只要有变量参与拼接结果就是变量，存在堆。

使用字面量时只创建一个常量池中的常量，使用 new 时如果常量池中没有该值就会在常量池中新创建，再在堆中创建一个对象引用常量池中常量。因此 String a = "a" \+ new String\("b"\) 会创建四个对象，常量池中的 a 和 b，堆中的 b 和堆中的 ab。

---
