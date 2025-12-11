---
title: jdk和cglib代理动态代理
tags:
  - proxy
  - AOP
categories: 编程
date: 2024-11-29 20:42:55
---

# PROXY

## 什么是代理

### 概念
- 代理分为: **静态代理**和**动态代理**
- 动态代理细分为2种: **JDK代理和CGLIB代理**
- 为什么要有代理:  由于某些原因需要给某对象提供一个代理以控制对该对象的访问,访 问对象不适合或者不能直接引用为目标对象, 代理对象作为访问对象和目标对象之间的中介(就像访客 <-> 售票处 <-> 火车站)
### 静态代理(static proxy)
- 静态代理是指代理类提前在编译时期就创建好的, 代理类和被代理类的的关系是固定的，**代理类会持有目标类的引用，并通过该引用调用目标类的方法**。
```java
// 被代理接口
public interface userService {
    void select();
}

// 被代理实现类
public class userServiceImpl implements userService {
    @Override
    public void select() {
        System.out.println("Service is being served.");
    }
}

// 代理类
public class userServiceProxy implements Service {
    private userService userservice;

    public userServiceProxy(userService userservice) {
        this.userservice = userservice;
    }

    @Override
    public void proxyService() {
        System.out.println("Before serving...");
        userservice.select();  // 调用目标对象的方法
        System.out.println("After serving...");
    }
}

// 测试
public class ProxyTest {
    public static void main(String[] args) {
        userService userservice = new userServiceImpl();
        userService proxy = new userServiceProxy(userservice);
        proxy.proxyService();
    }
}
```
### 为什么类可以动态的生成
- Java虚拟机类加载过程主要分为五个阶段：加载、验证、准备、解析、初始化。其中**加载阶段**需要完成以下3件事情：
  - 通过**一个类的全限定名**来获取定义此类的二进制字节流, **每个类都会有一个对应的 `.class` 文件**,该文件存储了类的字节码, 通过类加载器从类路径中查找并加载这个字节流
  - .class 文件中的字节码经过**解码**后，JVM 会将这些**字节码转换为方法区的数据结构**，即方法区的运行时数据（运行时常量池、类的字段、**方法**等）。这些数据结构用于后续的类操作。
  - **每个加载的类都会有一个对应的 `Class` 对象**，它代表了该类的元数据, 通过 `Class` 对象，JVM 可以访问该类的各种信息，如字段、**方法**、构造函数、注解等。这个 `Class` 对象也提供了**反射**机制的入口，可以让你**动态地获取和操作类的信息**（例如获取字段、调用方法、创建对象等)
  - **反射就是通过 `Class` 对象来动态获取类的信息，甚至在运行时动态生成和操作类**
### 静态代理和动态代理的区别
	静态代理靠手动去创建代理类和被代理类, 代理类和被代理类的对象在编译时就被创建了
	动态代理只能由被代理类手动创建对象, 正因如此代理类依托于反射的Constructor.newInstance()才能在运行时动态创建对象
#### 静态代理
  - 静态代理是在编译时就已经明确生成的，它是通过直接`new`出目标对象来完成代理操作的。
  - 在静态代理中，代理类和目标类的关系是硬编码的，代理类必须实现与目标类相同的接口或者直接继承目标类，且代理类的行为在编译时就已经确定。
  - **关键点**：静态代理确实在编译时就已经生成了代理类，且代理类**不依赖反射**。
#### 动态代理
  - 动态代理在运行时**通过反射机制**创建代理对象，并将目标方法的调用交给代理处理器（`InvocationHandler`）。
  - 它不需要手动写代理类，而是通**过`Proxy.newProxyInstance()`方法动态生成代理类**。这样，你可以通过一个通用的`InvocationHandler`处理不同的目标对象，减少了代码重复和硬编码。
  - **动态代理则是利用反射**,通过 `Proxy.newProxyInstance()` 方法，动态代理不需要编写代理类并将方法调用委托给 `InvocationHandler` 中的 `invoke()` 方法。虽然最终也会通过 `new` 创建目标对象，但代理类的生成和方法调用的分发机制是完全动态的。
  - **关键点**：动态代理依赖反射，且代理类是在运行时动态生成的，目标对象的方法调用由`InvocationHandler`来决定。
![17329446161661732944615286.png|700x342](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17329446161661732944615286.png)
## 动态代理的运用
### 代理模式优缺点
- 优点: 
  1. 代理模式在客户端与目标对象之间起到一个中介作用和**保护目标对象**的作用 -> 隐藏卖家
  2. 代理对象可以**增强目标对象的功能**，被用来间接访问底层对象，与原始对象具有相同的 hashCode  -> 增强功能
  3. 代理模式能将客户端与目标对象分离，在一定程度上**降低了系统的耦合度**  -> 降低
- 缺点：增加了系统的复杂度
### JDK动态代理
- **通过代理类实现接口的方式** -> JDK动态代理 -> 反射
- JDK动态代理主要涉及两个类：**java.lang.reflect.Proxy**和 **java.lang.reflect.InvocationHandler**
```java
// 被代理接口
interface Service {
    void serve();
}

// 被代理实现类
public class ServiceImpl implements Service {
    @Override
    public void serve() {
        System.out.println("Service is being served.");
    }
}

// 动态代理类的 InvocationHandler
class ServiceInvocationHandler implements InvocationHandler {
    private Object target;

    public ServiceInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before serving...");
        Object result = method.invoke(target, args);  // 调用目标对象的方法
        System.out.println("After serving...");
        return result;
    }
}

// 测试
class ProxyTest {
    public static void main(String[] args) {
        Service service = new ServiceImpl();
        Service proxy = (Service) Proxy.newProxyInstance(
                service.getClass().getClassLoader(), // 被代理类的类加载器类型
                service.getClass().getInterfaces(),  // 被代理类的接口类型
                new ServiceInvocationHandler(service) // 创建一个将传给代理类的调用请求处理器，处理所有的代理对象上的方法调用
        );
        // 		a.JDK会通过根据传入的参数信息动态地在内存中创建和.class 文件等同的字节码
        //      b.然后根据相应的字节码转换成对应的class，
        //       c.然后调用newInstance()创建代理实例  例如: $proxy0
        proxy.serve();  // 等价于 handle.invoke()
    }
}
```

```java
// 代理类内部结构
public final class $Proxy0 extends Proxy implements Service {
    private static Method m1;
    private static Method m3; // 目标方法server()
    private static Method m2;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void serve() throws  { // 目标方法
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("proxy.Service").getMethod("serve"); // 反射加载方法
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

![17328903987391732890398584.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17328903987391732890398584.png)
### 调用处理器和反射类
- **Proxy**
```java
// Proxy提供用于创建动态类和实例的静态方法, 创建的所有动态代理类的超类
Proxy::newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h);
// 定义了代理对象调用方法时希望执行的动作，用于集中处理在动态代理类对象上的方法调用
Object invoke(Object proxy, Method method, Object[] args)
```
-  **InvocationHandler**
```java
// 用于获取指定代理对象所关联的调用处理器
static InvocationHandler getInvocationHandler(Object proxy);
// 返回指定接口的代理类
static Class<?> getProxyClass(ClassLoader loader, Class<?>... interfaces);
// 构造实现指定接口的代理类的一个新实例，所有方法会调用给定处理器对象的 invoke 方法
static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h);
// 返回 cl 是否为一个代理类
static boolean isProxyClass(Class<?> cl);
```
- 通过`Proxy` 类的 `newProxyInstance()` 创建的代理对象在调用方法的时候，实际会调用到实现`InvocationHandler` 接口的类的 `invoke()`方法
### CGLIB代理
- **通过代理类继承被代理类重写其方法的方式** -> CGLIB代理 -> 字节码
- 在运行时通过继承的方式为目标类创建代理类。与 JDK 动态代理不同，CGLIB 不要求代理类实现接口，而是**直接对被代理类进行字节码增强**
```java
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

// 被代理类
public class Service {
    public void serve() {
        System.out.println("Service is being served.");
    }
}

// 代理类
class ServiceInterceptor implements MethodInterceptor {
    private Object target;

    public ServiceInterceptor(Object target) {
        this.target = target;
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("Before serving...");
        Object result = method.invokeSuper(target, args);
        System.out.println("After serving...");
        return result;
    }
}

// 测试
public class ProxyTest {
    public static void main(String[] args) {
        Service service = new Service();// 被代理类对象
       	// Enhancer：CGLIB 的核心类，用于生成代理类
        // MethodInterceptor：用于定义方法拦截逻辑
        Service proxy = (Service) Enhancer.create(Service.class, new ServiceInterceptor(service));
        // MethodProxy：用于调用父类的方法
        proxy.intercept();
    }
}
```
- 在 CGLIB 动态代理机制中 `MethodInterceptor` 接口和 `Enhancer` 类是核心
- 自定义 `MethodInterceptor` 实现类并重写 `intercept` 方法，`intercept` 用于拦截增强被代理类的方法
- 通过 `Enhancer`类来动态获取被代理类，通过 `Enhancer` 类的 `create()`创建代理类, 当代理类调用方法的时候，实际调用的是 `MethodInterceptor` 中的 `intercept` 方法 这里的 `intercept`类似于 `invoke`方法
## JDK和CGLIB代理的对比

| 对比         | JDK代理                    | CGLIB代理             |
| ---------- | ------------------------ | ------------------- |
| 代理对象生成方式   | **基于反射生成$proxy0代理类**     | **基于字节码生成子类**       |
| 方法的调用方式    | **基于proxy.方法调用**         | **基于底层子类super()调用** |
| 初始化效率      | **反射newInstance生成对象效率高** | 直接生成子类对象效率低         |
| 方法执行效率     | 执行效率低                    | **执行效率高(直接调用父类方法)** |
| spring默认策略 | 有接口                      | 无接口                 |

- 总结: CGLIB 由于主要基于字节码操作，如果是**只要初始化一次且方法要长期被调用**, 性能通常高于 JDK 动态代理, 适用于不需要频繁修改代理逻辑的场景，因为cglib代理是子类,父类修改了子类也要跟着改, 并且内存占用要高于jdk代理
## 工厂代理模式

- 工厂代理模式是一种将**工厂模式**与**代理模式**相结合的设计模式。通过工厂方法创建代理对象，从而**简化代理对象的创建流程**，同时增强代理模式的灵活性
- 工厂方法屏蔽了对象的创建细节，提高了代码的灵活性和扩展性。
### 动态工厂代理模式
- 动态工厂代理模式结合了 **动态代理** 和 **工厂模式**，通过工厂方法动态生成代理对象。
```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

// 定义接口
public interface Service {
    void serve();
}

// 目标类
public class RealService implements Service {
    @Override
    public void serve() {
        System.out.println("RealService is serving...");
    }
}

// 动态代理处理器
class ServiceInvocationHandler implements InvocationHandler {
    private final Object target;

    public ServiceInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before method: " + method.getName());
        Object result = method.invoke(target, args); // 调用目标对象的方法
        System.out.println("After method: " + method.getName());
        return result;
    }
}

// 工厂类
public class ServiceFactory {
    public static Service getProxy() {
        RealService realService = new RealService();

        return (Service) Proxy.newProxyInstance(
            realService.getClass().getClassLoader(),
            realService.getClass().getInterfaces(),
            new ServiceInvocationHandler(realService) // 传入动态代理处理器
        );
    }
}

// 测试
public class Main {
    public static void main(String[] args) {
        Service service = ServiceFactory.getProxy(); // 解耦了代理类创建过程
        service.serve();
    }
}
```
### 工厂代理模式的优缺点
**优点：**
1. **解耦**：将代理对象的创建逻辑封装到工厂类中，客户端代码不需要关心代理对象的实现细节。
2. **灵活性**：可以动态生成代理对象，支持多种代理模式（如静态代理、动态代理、CGLIB 代理）。
3. **扩展性**：如果需要更改代理逻辑或代理方式，只需修改工厂类的实现，而无需影响客户端代码。
**缺点：**
4. **复杂性**：相比直接使用代理模式，工厂代理模式的实现复杂度更高。
5. **性能开销**：动态代理可能增加运行时的性能开销（如反射调用）
## 注意
### 性能开销
- **JDK 动态代理的性能**：JDK 动态代理基于反射机制生成代理对象和方法调用，因此**每次方法调用都会涉及到反射**，性能相对较低，尤其是在大量调用的场景中。如果对性能要求非常高的场合，尽量避免过多使用 JDK 动态代理。
- **CGLIB 代理的性能**：CGLIB 通过字节码技术生成子类代理，因此没有反射带来的性能开销，相对较为高效。不过，CGLIB 的代理对象是通过继承生成的，这可能导致**对目标类的修改（如方法的添加、删除等）对代理类的影响较大**，可能需要重新生成代理类。
### 代理对象的生命周期
- 在使用动态代理时，代理对象的生命周期通常由代理工厂管理。确保代理对象在适当的时机销毁，以避免内存泄漏。特别是在使用动态代理时，通过反射和字节码生成代理对象时，代理对象可能会占用更多的内存，因此需要注意对象的销毁和资源回收。
- 对于 **CGLIB 代理**，需要注意，由于它是通过继承目标类生成的代理对象，所以会增加内存占用。并且，CGLIB 对目标类的生成依赖于字节码操作，因此需要确保代理类在不再使用时被正确清理。
### 动态代理和AOP的结合
- **AOP（面向切面编程）**：动态代理常与 AOP 结合使用，尤其是在 Spring 框架中，动态代理用于增强类的方法。通过 AOP，可以在不改变目标类的代码情况下，在方法执行前后执行额外的操作，例如事务管理、权限验证、日志记录等。
![17601594300691760159429855.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17601594300691760159429855.png)
- 在 Spring 框架中：
  - 如果被代理类实现了接口，Spring 会使用 JDK 动态代理(默认方式)
  - 如果被代理类没有实现接口，Spring 会使用 CGLIB 代理
- 通过 AOP，可以灵活地为目标对象动态织入切面，增强其功能。
![17601593600641760159359315.png|700x691](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17601593600641760159359315.png)
### 获取代理类对象
##### 具体情况：
1. **没有AOP代理的情况：** 如果你注入的是一个普通的类，并且没有使用 Spring AOP 的代理机制，那么 `@Autowired` 会注入这个类的实例对象。
2. **有AOP代理的情况：** 如果你的类上使用了 AOP 功能（比如事务、日志、权限等），Spring 会为这个类创建一个代理对象。这时，`@Autowired` 注入的就是代理类对象，而不是原始类的实例。
3. 使用 **`@Transactional` 注解**的代理对象, spring 会为你创建一个代理对象来处理事务
##### 代理类型：
- **JDK 动态代理**：当你注入的是接口类型的类时，Spring 会通过 JDK 动态代理来创建一个实现了该接口的代理类。
- **CGLIB 代理**：当你注入的是没有接口的类时，Spring 会使用 CGLIB 生成一个该类的子类作为代理对象。
##### 如何判断注入的是代理对象：
可以通过 `getClass().getName()` 来检查实例的类型。如果你看到的是一个类似 `com.sun.proxy.$Proxy` 的类名，那就说明注入的是代理对象。
##### 获取当前代理对象
- Spring AOP 提供了 `AopContext`，它允许你在方法调用中访问当前的代理对象:
```java
// 获取代理对象的 AopProxy
AopProxy aopProxy = (AopProxy) AopContext.currentProxy();
// 获取代理对象
MyService proxy = (MyService) aopProxy.getProxy();
```
-  通过 `ApplicationContext` 获取代理对象: 
```java
MyService proxy = (MyService) applicationContext.getBean(MyService.class);
```
- 如果你想判断某个对象是否为代理对象，可以使用 **AopUtils.isAopProxy()** 方法

