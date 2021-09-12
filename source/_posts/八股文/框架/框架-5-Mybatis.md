---
title: 框架 - 5.Mybatis
date: 2020-08-16 17:31:03
categories: 
- 八股文
- 框架
tags:
- 框架
- 面试
---

### **Mybatis 5**

#### **Q1：Mybatis 的优缺点？**

**优点**

相比 JDBC 减少了大量代码量，减少冗余代码。

使用灵活，SQL 语句写在 XML 里，从程序代码中彻底分离，降低了耦合度，便于管理。

提供 XML 标签，支持编写动态 SQL 语句。

提供映射标签，支持对象与数据库的 ORM 字段映射关系。

**缺点**

SQL 语句编写工作量较大，尤其是字段和关联表多时。

SQL 语句依赖于数据库，导致数据库移植性差，不能随意更换数据库。

---

#### **Q2：Mybatis 的 XML 文件有哪些标签属性？**

select、insert、update、delete 标签分别对应查询、添加、更新、删除操作。

parameterType 属性表示参数的数据类型，包括基本数据类型和对应的包装类型、String 和 Java Bean 类型，当有多个参数时可以使用 \#{argn} 的形式表示第 n 个参数。除了基本数据类型都要以全限定类名的形式指定参数类型。

resultType 表示返回的结果类型，包括基本数据类型和对应的包装类型、String 和 Java Bean 类型。还可以使用把返回结果封装为复杂类型的 resultMap 。

---

#### **Q3：Mybatis 的一级缓存是什么？**

一级缓存是 SqlSession 级别，默认开启且不能关闭。

操作数据库时需要创建 SqlSession 对象，对象中有一个 HashMap 存储缓存数据，不同 SqlSession 之间缓存数据区域互不影响。

一级缓存的作用域是 SqlSession 范围的，在同一个 SqlSession 中执行两次相同的 SQL 语句时，第一次执行完毕会将结果保存在缓存中，第二次查询直接从缓存中获取。

如果 SqlSession 执行了 DML 操作（insert、update、delete），Mybatis 必须将缓存清空保证数据有效性。

---

#### **Q4：Mybatis 的二级缓存是什么？**

二级缓存是Mapper 级别，默认关闭。

使用二级缓存时多个 SqlSession 使用同一个 Mapper 的 SQL 语句操作数据库，得到的数据会存在二级缓存区，同样使用 HashMap 进行数据存储，相比于一级缓存，二级缓存范围更大，多个 SqlSession 可以共用二级缓存，作用域是 Mapper 的同一个 namespace，不同 SqlSession 两次执行相同的 namespace 下的 SQL 语句，参数也相等，则第一次执行成功后会将数据保存在二级缓存中，第二次可直接从二级缓存中取出数据。

要使用二级缓存，需要在全局配置文件中配置 \<setting name="cacheEnabled" value="true"/\> ，再在对应的映射文件中配置一个 \<cache/\> 标签。

---

#### **Q5：Mybatis** \#{} **和** ${} **的区别？**

使用 ${} 相当于使用字符串拼接，存在 SQL 注入的风险。

使用 \#{} 相当于使用占位符，可以防止 SQL 注入，不支持使用占位符的地方就只能使用 ${} ，典型情况就是动态参数。
