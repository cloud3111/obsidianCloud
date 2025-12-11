---
title: 理解栈和堆在JVM中作用
tags:
  - stack
  - heap
  - Jvm
categories: 编程
date: 2024-12-02 18:22:38
---

# stackHeap

- **栈是运行时的单位，而堆是存储的单位**
- **每个方法调用都会创建一个栈帧，栈帧保存着该方法的局部变量、参数等信息，方法执行完毕后栈帧销毁。**
## 栈和堆的生命周期

通常情况下，在方法内部定义的引用类型变量只在该方法内部有效。一旦方法执行结束，变量将被销毁

![17331392979641733139297889.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17331392979641733139297889.png)

## 栈和堆的作用

  - **栈内存的特性：**
    - 每个线程都有独立的栈内存是私有的
    - 方法调用过程中产生的局部变量（包括基本数据类型和对象引用）
    - 栈内存随着方法的进栈和出栈自动管理，不需要 GC 来回收
    - 栈的生命周期与线程一致，线程结束时(方法执行完毕后)栈内存自动销毁，栈中的数据会自动销毁, 不涉及gc

    **堆内存的特性：**
    - 堆是线程共享的, 所有线程可以访问堆内存中的对象。

    - 堆内存用于存储所有对象实例和数组，生命周期可能超出方法的执行范围。
    - 这些对象由 GC 管理，确保程序运行过程中不会出现内存泄漏。
    - GC 通过算法检测堆中哪些对象不再被引用，从而释放它们的内存。

- 栈是用来执行当用户线程执行到某个方法时用来执行出入栈运算操作

- 堆是程序启动初期用来存储所有实例对象,class对象,字符串对象以及常量池的存储空间(通常和方法区共同作用), 当用户线程进来执行到特定方法而new出来的对象不受容器控制时,就需要gc了

## 例子

- **栈中存储引用变量**： 栈中的局部变量可能保存对堆中对象的引用。例如：

```java
String refer1 = new String("Hello"); // refer1 是存放在栈中的局部变量，指向堆中的 "Hello" 对象
String refer2 = "Hello";             // 字符串常量池中存储了 "Hello"
```

当栈帧退出，`name` 变量从栈中销毁。如果没有其他引用指向堆中的 `"Hello"` 对象，GC 会回收它。

- **对象在堆上分配内存**： 方法执行时，局部变量指向堆中的对象。只要堆中的对象有引用（强引用、弱引用等），它就不会被回收

```java
public void gcExample() {
    String str1 = new String("Hello"); // str1 存在栈中，"Hello" 对象存放在堆中
    String str2 = str1;               // str2 也指向堆中的 "Hello"
    str1 = null;                      // str1 断开引用
    // 由于 str2 仍然引用着堆中的 "Hello", 只要有一个引用, GC就不会回收该对象
    str2 = null;                      // str2 也断开引用，此时 "Hello" 可被 GC 回收
}
```

## 内存泄漏(memoryLeak)

```java
public class MemoryLeakExample {
    // static是常量,无法通过栈自动销毁,也就无法GC(gc需要无指引的对象),当list存储的对象越来越多就会造成内存泄漏
    private static List<Object> list = new ArrayList<>(); 

    public static void createLeak() {
        for (int i = 0; i < 1000000; i++) {
            Object obj = new Object(); // 新对象存储在堆中
            list.add(obj);             // 对象被静态 list 引用
        }
    }
}
```

- **list的引用不是定义在方法内部(即局部变量),也就导致了变量的reference不能被销毁**

```java
public class NormalExample {
    public static void createObjects() {
        for (int i = 0; i < 1000000; i++) {
            Object obj = new Object(); // 局部变量 obj 存储在栈中
        } // 方法结束后，obj 的作用域结束，堆中的对象无引用，GC 会回收
    }
}
```
