---
title: 函数式接口和stream流API
tags:
  - stream
categories: 编程
date: 2024-11-28 14:48:20
banner: "[[pixel-banner-image.png]]"
---

# FunctionStream
## 初识stream流
### 什么是stream流
- stream流是`jdk8`引入的一种简化编程效率的工具, 通过stream流API,可以很轻松的把一种形式的对象转换成另一种对象
- 通过一系列**中间操作**和**最终操作**来进行**功能增强**, 中间操作通常会引用一个函数式接口作为参数, 利用**参数行为化隐式进行方法重写,** 最终操作则会触发流的遍历并执行最终操作(把返回值传给下一个流)
### 函数式接口(方法重写)
- 函数式接口是只包含一个方法的接口
- 通过函数式接口对隐式方法的重写, 可以通过lambda表达式显示进行方法重写, 也可以通过方法引用(引用类的方法来覆盖重写接口方法)
- 4中基础函数式接口类型: Predicat  Consumer  Supplier  Function  Comparator(int compare(T o1, T o2))
- lambda表达式: 当在方法内使用lambda其实是对函数接口唯一方法的重写, 通过隐式捕获上下文的变量来操作流中的元素
![17327779764721732777975546.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17327779764721732777975546.png)
```java
1.Predicate
    List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    Predicate<Integer> isEvenPredicate = num -> num % 2 == 0;
	numbers.stream()
    .filter(isEvenPredicate)
    .forEach(System.out::println);
2.Consumer
    List<String> names = new ArrayList<>();
    names.add("Alice");
    names.add("Bob");
    names.add("Charlie");
    Consumer<String> printConsumer = System.out::println;
    在 forEach 方法内部，它会被隐式调用以打印列表中的每个元素。
    names.forEach(printConsumer);
3.Function
    Function<String, Integer> parseIntFunction = Integer::parseInt;
    int result = parseIntFunction.apply("123");
    System.out.println(result);
4.Supplier (重点)
    Supplier<Double> randomSupplier = Math::random;
	System.out.println(randomSupplier.get());
```
### 自定义函数式接口
- @FunctionInterface注解只是一个规范性的对象,加了注解在接口上有两个方法编译都过不了
- 示例:
```java
// 接口规范了重写方法的返回值和参数
@FunctionalInterface
public interface returnName<T,V> {
    V accept(T t);
}
```

```java
// 定义了一个方法: 接收一个函数式接口作为参数
public void print(returnName<Person,String> returnName){
    // 里面又把被告作为参数传了进去
    returnName.accept(new Person("jack", 18, age.MAN));
}
```

```java
// 这里调用的实例确实是通过上下文拿到的
@Test
public void test() {
    print(Person::getName); // 结果:jack 
}
```
- 重点:
  - **print方法体里其实才是对方法的重写: 传入了一个Person对象**
  - **而在Test里面其实是对返回值的一个筛选输出: print(person -> person.getName());**
  - **反过头来userSet.stream() -> 其实userSet是被当做重写方法的参数放了进去**
```java
public void ext(Function<Person, String> function, Person person){
    function.apply(person);
}

@Test
public void test1() {
    ext(person -> person.getName(), new Person("jack", 18, age.MAN));
}
```
- 在stream流中其实对方法参数的传入进行了隐藏,而不是像上面自己传参
## 深入stream
### 流程stream
```java
@Test
public void test() {
    Set<user> userSet = new HashSet<>();
    userSet.add(new user());
    List<userDTO> perfect = userSet.stream()
        .map(user -> new userDTO())
        .toList();
}
```

![17328538657361732853865377.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17328538657361732853865377.png)
### 闭包
-  Lambda 表达式闭包特性，能够捕获外部作用域的局部变量
- 这是jdk的新特性, 而不是语法糖
```java
public void test3(Consumer<Integer> consumer){
    Integer inner = 20;
    consumer.accept(inner); //  <- 重写传参
}

@Test
public void test2(){
    Integer outer = 19;  <- 闭包
	    test3(inner -> System.out.println(inner + ":" + outer)); // outer闭包读取,因为outer不在作用域之内
}
```
## Lambda语法糖
```java
// lambda写法 
select(list, (Person) -> Person.getGender() == age.MAN && Person.getName().equals("任小粟"));

// 实际语法糖
select(list, new lambda<Person>() {
    @Override
    public boolean select(Person Person) {
        return Person.getGender().equals(age.MAN) && Person.getName().equals("任小粟");
    }
});
```
## 结论
- 函数式接口负责中间操作: 传参  返回结果值
- Stream流负责对结果值的过滤filter, 转换map, 遍历foreach  
- Lambda其实是在简化对结果值的判断
- Stream流所有你需要做的事确实是对函数式方法的重写,内部隐藏了传参以及各种逻辑转化
![17328588597371732858859198.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17328588597371732858859198.png)
