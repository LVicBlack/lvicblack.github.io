---
title: getDeclaredField和getField的使用和区别,以及如何获取父类的私有字段
date: 2020-06-23 06:17:03
categories: 
- Java
tags:
- Java
- 面试
---


### 结论

> 1. getDeclaredFields方法仅对**类本身的字段**有效果，对于继承的父类的字段无效
> 2. getFields方法只能获取**类及其父类的公共字段**
> 3. 获取父类的私有字段需要先使用getSuperClass获取父类Class，然后通过父类Class的getDeclaredFields方法获取父类的所有字段

 下面我们举个简单的栗子：

这是我们定义的一个父类，其中定义了一个公共字段superDesc，和一个私有字段superName

```
public class SuperModel {
 
	private String superName;
	public String superDesc;
 
	public String getSuperDesc() {
		return superDesc;
	}
	public void setSuperDesc(String superDesc) {
		this.superDesc = superDesc;
	}
	public String getSuperName() {
		return superName;
	}
	public void setSuperName(String superName) {
		this.superName = superName;
	}
 
}
```

这是我定义的一个子类，其中也定义了一个公共字段modelDesc和一个私有字段modelName

```
public class Model extends SuperModel {
 
	public String modelDesc;
	private String modelName;
 
	public String getModelDesc() {
		return modelDesc;
	}
	public void setModelDesc(String modelDesc) {
		this.modelDesc = modelDesc;
	}
	public String getModelName() {
		return modelName;
	}
	public void setModelName(String modelName) {
		this.modelName = modelName;
	}
}
```

下面我使用了三个测试方法来测试。

```
public class ReflectTest {
	public static void main(String[] args) {
		Model model = new Model();
		model.setSuperDesc("父类公开字段");
		model.setSuperName("父类私有字段");
		model.setModelDesc("子类公开字段");
		model.setModelName("子类私有字段");
 
		//通过getFields方法遍历字段获取model字段
		getNameByGetFields(model);
 
		//通过getDeclaredFeilds方法获取model字段
		getNameByDeclaredFields(model);
 
		//通过getSuperClass方法获取model字段
		getNameBySuperClass(model);
	}
 
	private static void getNameByGetFields(Model model) {
		Class<? extends Model> sonClass = model.getClass();
		try {
			Field[] fields = sonClass.getFields();
			System.out.println("通过getFields方法遍历字段获取model字段:       ");
			for (Field field : fields) {
				System.out.println(field.getName() + ":" + field.get(model));
			}
		} catch (IllegalAccessException e) {
			e.printStackTrace();
		}
	}
 
	private static void getNameByDeclaredFields(Model model) {
		Class<? extends Model> sonClass = model.getClass();
		try {
			System.out.println("通过getDeclaredFeilds方法获取model字段：      ");
			Field[] fields = sonClass.getDeclaredFields();
			for (Field field : fields) {
				field.setAccessible(true);
				System.out.println(field.getName() + ":" + field.get(model));
			}
		} catch (IllegalAccessException e) {
			e.printStackTrace();
		}
	}
 
	private static void getNameBySuperClass(Model model) {
		Class<? extends Model> sonClass = model.getClass();
		try {
			System.out.println("通过getSuperClass方法获取superName字段    ");
			Class<?> superclass = sonClass.getSuperclass();
			Field[] fields = superclass.getDeclaredFields();
			for (Field field : fields) {
				field.setAccessible(true);
				System.out.println(field.getName() + ":" + field.get(model));
			}
		} catch (IllegalAccessException e) {
			e.printStackTrace();
		}
	}
}
```

---

 当我使用getFields方法遍历所有字段时，输出了以下信息：

![20191117210235351.png](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20191117210235351.png)

这说明**getFileds方法只能获取公共字段，但是可以同时获取本身和父类的公共字段**

以下是JDK文档中的描述：

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/getFields.png)

---

 当我使用getDecleadFields遍历所有字段时，输出了以下信息：

![20191117211210889.png](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20191117211210889.png)

这说明**getDecleadFields方法只能获取类本身的字段，但是不能获取父类的字段**。

以下时JDK文档中的描述：

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/getDeclaredFields.png)

---

当我使用getSuperClass获取父类Class后再使用父类的Class的getDecleadFields方法，不出所料，输出的父类所有的字段。

![20191117211620573.png](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20191117211620573.png)

这说明**要获取父类的私有字段必须先获取它的Class对象**。而要遍历一个对象应该出现的所有字段，就需要遍历这个对象和这个对象的父类所有字段，不能一次性直接获取到所有字段。

