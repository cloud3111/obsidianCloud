---
title: 深入理解注解
tags:
  - Annotation
date: 2024-11-01 20:48:29
categories: 编程
---

# Annotation

> 凡心所向🚀素履所往 🌟生如逆旅 ✨一苇以航

## 什么是注解

- **注解和注释不同,当编译器运行时,注释会被跳过**
- 而注解不会,反而会反过来影响程序 如:Tomcat, 编译器, 框架
- 通过直接描述代码的行为来告诉框架或编译器应该做什么
- **注解通常与反射机制结合使用，通过反射来在运行时获取注解信息，从而根据注解内容执行特定逻辑**
## 注解的作用
- **标识 类**:  当程序启动时,扫描所有类的时候, 标识当前类为… 例如:@WebServlet用来标识当前类为Servlet, 并管理类的生命周期(启动,运行,销毁)
- **赋予 类**:  标识类时赋予类的作用时间(`RetentionPolicy.RUNTIME`)   Spring 使用注解来自动配置 Bean(使bean在程序执行运行时return唯一实例)
- **调用 类**:  本身不做实现, 利用反射, 比如加了一个运行时注解, 那这个程序就会在运行时被执行, 而不用去创建实例对象.方法
- **标识方法的过程解析**: 遍历所有方法 找到被标识的方法 反射执行
## 注解的组成
**两个核心元注解:@Target 和 @Retention**
- `@Target`: 指定注解的作用对象  class method field 
- `@Retention`:指定注解的生命周期
	  - SOURCE      源代码级别，由编译器处理，处理之后就不再保留
	  - CLASS         注解信息保留到类对应的字节码文件中
	  - RUNTIME    由JVM(解释器)读取，运行时使用
## 例子
```java
// 作用: 标识类方法,反射调用类方法执行
@Target(ElementType.METHOD) 
@Retention(RetentionPolicy.RUNTIME)
public @interface InitMethod { 
}
```

```java
public class testAnnotation {

    @InitMethod
    public void ImMethod() {
        System.out.println("我在没有实例化的情况下启动了");
    }

    public static void main(String[] args) throws Exception {
        Class<?> clazz = Class.forName("com.function.testAnnotation");
        Constructor<?> constructor = clazz.getDeclaredConstructor();
        Object instance = constructor.newInstance();

        // 标识方法的过程解析: 遍历所有方法 找到被标识的方法 反射执行
        Method[] methods = clazz.getDeclaredMethods();
        // 查证方法是否有这个注解存在
        System.out.println("方法名: 是否存在注解");
        for (Method method : methods) {
            // (本质)注解的字节码文件有一个flag: 所以只要判断flag里面有没有ACC_ANNOTATION
            boolean present = method.isAnnotationPresent(InitMethod.class); 
            System.out.println(method.getName() + ": " + present);
            // 如果存在,反射执行
            if (present) {
                method.invoke(instance);
            }
        }
    }
}
```

## 补充

- `AnnotatedElement` 接口是 Java 反射 API 中的一个关键接口，用于操作和读取注解。它由 `Class`、`Method`、`Field` 等类实现，允许在**运行时检查注解信息**。`AnnotatedElement` 的接口方法使得我们可以在运行时灵活地检查并获取注解的详细信息，从而决定如何动态处理类、方法或字段。
- Spring 就大量使用了这一模式：扫描注解来**注入依赖**、**管理事务**、**控制请求**等。
- **注解底层利用反射实现功能**
- **注解类型本身也是接口, 一个类是注解类型,那它一定是接口类型**

