---
title: " 日期格式和消息转换器"
tags:
  - Converter
  - DateTimeFormatter
  - Date
  - LocalDateTime
categories: 编程
date: 2024-11-23 09:52:54
---

# 日期格式和消息转换器

> 你会遇见很多星星, 而我只会心动💓一个月亮

---
## Date 和 LocalDate 的区别
### 1. 基本概念
- Date：是 Java 1引入的日期时间类，底层存储的是一个UTC时间戳（即自 1970-01-01 00:00:00 UTC 到今日**精确到毫秒的时间戳**）。
- LocalDate：是 Java 8 引入的新日期时间 API，表示一个纯日期（年月日），不包含时间信息和时区信息，具有不可变性和线程安全性。
### 2. 系统时钟影响
 - 不管是Date还是LocalDate, 在new 对象时都是根据本地系统时钟(时区)来设置时间的
### 3. 数据库存储差异
- Date 通常与数据库中的 **timestamp** 或 **datetime** 类型直接映射，数据库和应用服务器时区不一致时，可能出现时间偏移问题。
- LocalDate一般映射为数据库的 **date** 类型，**仅存储日期字符串**，不涉及时间和时区，因此不会出现时间偏移问题。
- **存储时，MySQL 会将当前会话时区下的时间值转换成 UTC（协调世界时）进行内部存储。当查询 `TIMESTAMP` 字段时，MySQL 又会将存储的 UTC 时间转换回当前会话所设置的时区来显示。**
### 4. 区别
- 不可变性与线程安全:
	- Date 是可变对象，线程不安全。 占用4-7字节
	- LocalDate 是不可变对象的字符串，所以线程安全，推荐使用。  占用5-8字节
- Java.util 下Date需要额外的Calendar来进行日期加减, 比较繁琐; 而Java.time 下LocalDate提供丰富的 API
### 5. 时间戳转换区别
- Date 和时间戳的转换是直接基于 getTime()（毫秒值）。
- LocalDate 与时间戳没有直接关联，如果需要转时间戳，必须先转换为 LocalDateTime 并指定时区。
### 6.数值型时间戳
- 底层也是时间戳, 但是由于使用int或者bigint来存储时间戳, 不会出现时区问题, 占用空间更小(4字节), 缺点是不够直观
## 什么是转换器

- **SpringMVC**执行流程: **DispatherServlet** -> **HandlerMapping** -> **HandleAdapt** -> **Resolve**
- 当请求被Tomcat容器捕获时,下一步将交由前置控制器去分发请求, 前置控制器核心方法将调用映射器,去匹配请求(uri匹配失败返回404, 请求方法匹配失败返回405), 成功之后进入适配器, 进行参数的反序列化, handle处理完成要离开适配器的时候也将由适配器去做序列化, 通常返回数据默认加上@ResponseBody(restful风格), 表明返回的是一个对象

- 无论是请求**数据的反序列化**，还是**响应数据的序列化**，最终都需要将数据转换成**字节数组**（`byte[]`）以便进行网络传输,也就是 xxxToByteConverter, 任何转换器最终都需要经过这个进行二次转换

- **`@RequestBody`**: 当你使用 `@RequestBody` 时，Spring 会利用 `HttpMessageConverter` 来处理请求体的转换。**如果你自定义了 `ObjectMapper`**（如 `JacksonObjectMapper`），它会生效并处理请求体的序列化和反序列化。因此，所有通过 `@RequestBody` 接收到的数据，都会通过你自定义的 `ObjectMapper` 来进行类型转换

- @**`RequestParma`/`Pathvariable`**: 当你使用`RequestParma`/`Pathvariable` 时, 它们不会直接使用 `ObjectMapper` (`JacksonObjectMapper`)来转换, 而是使用 Spring 的类型转换器机制 (`ConversionService`),你可以通过实现 `Converter` 接口或 `@InitBinder`注解 来扩展转换逻辑

## 参数绑定核心机制
### `@RequestBody`
- 作用对象：**请求体中的 JSON/XML 等格式的数据**
- 使用工具：**`HttpMessageConverter`（消息转换器）**
- 默认实现：**`MappingJackson2HttpMessageConverter`**
- 影响方式：可通过自定义 `ObjectMapper模块`来定制序列化/反序列化逻辑（如时间格式、字段命名策略等）
### `@RequestParam` & `@PathVariable`
- 作用对象：**URL 查询参数、路径参数**
- 使用工具：**`ConversionService`（类型转换服务）**
- 自定义扩展方式：
    - 实现Converter接口: `org.springframework.core.convert.converter.Converter<S, T>` 
    - 通过 `@InitBinder` 自定义 `WebDataBinder`
### `HttpMessageConverter` 的组成
- Spring Boot 默认引入的是 **Jackson** 相关的 HTTP 消息转换器（除非你引入了其他如 Gson、Fastjson 会自动替换）：
- `MappingJackson2HttpMessageConverter`
- 其中底层依赖的就是 Jackson 的 `ObjectMapper`，你自定义的 `JacksonObjectMapper` 正是通过这个机制生效。

## 自定义转换器
- **`针对@RequestParam`** 
	- 单次生效: 采用注解
		- **`@RequestParam @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")`** 
	- 全局生效: 自定义Converter转换器
```java
@GetMapping  
public String test(@RequestParam LocalDateTime date) {  
    System.out.println("接收到的时间: " + date);  
    return "ok";  
}
```

```java
/**  
 * 自定义转换: @RequestParam  
 * @author cloud_3111  
 * @since 2025-04-16  
 */public class customConvert implements Converter<String, LocalDateTime> {  
    @Override  
    @SneakyThrows    
    public LocalDateTime convert(String source) {  
        source = source.trim();  
        String dateString = DateUtil.format(date, "yyyy-MM-dd HH:mm:ss");
        return dateString;
    }  
}
```

```java
@Configuration  
public class WebMvcConfig implements WebMvcConfigurer {  
    @Override  
    public void addFormatters(FormatterRegistry registry) {  
        registry.addConverter(new customConvert());  
    }  
}
```
## 自定义消息转换器
- 定义消息转换器的两种方式: 
  - 1.继承ObjectMapper,添加进行模块进行功能增强
  - 2.完全自定义消息转换器替换Jackson的converter
- **`针对@RequestBody`**
	- 全局生效: 自定义继承ObjectMapper的消息转换器
	-  单次生效: 采用注解
- 在对象中对属性date使用: **`@JsonFormat(pattern = "yyyy-MM-dd", timezone = "GMT+8")`** 

```java
/**  
 * 拓展新的消息转换器模块  
 * @author cloud_3111  
 * @since 2025-04-16  
 */public class newModelMapper extends ObjectMapper {  
    public static final String DEFAULT_DATE_FORMAT = "yyyy-MM-dd";  
    public static final String DEFAULT_DATE_TIME_FORMAT = "yyyy-MM-dd HH:mm:ss";  
  
    public newModelMapper() {  
        super();  
        //收到未知属性时不报异常 unkonwProperties        this.configure(FAIL_ON_UNKNOWN_PROPERTIES, false);  
        //反序列化时，属性不存在的兼容处理  
        this.getDeserializationConfig().withoutFeatures(FAIL_ON_UNKNOWN_PROPERTIES);  
        // 序列化和反序列化都要添加  
        SimpleModule simpleModule = new SimpleModule()  
                .addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)))  
                .addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)))  
                .addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)))  
                .addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)));  
        //把功能加入
        this.registerModule(simpleModule);  
    }  
}
```

```java
@Configuration  
public class WebMvcConfig implements WebMvcConfigurer {  
    @Override  
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {  
        //创建一个消息转换器对象  
        MappingJackson2HttpMessageConverter converterObject = new MappingJackson2HttpMessageConverter();  
        //需要为消息转换器设置一个对象转换器，对象转换器可以将Java对象序列化为json数据  
        converterObject.setObjectMapper(new newModelMapper());  
        //将自己的消息转化器加入容器中  
        converters.add(0, converterObject);  
    }  
}
```
## 总结
- ⚠️**注意**: 使用了上面的转换器使得只能接收前端的 `yyyy-MM-dd HH:mm:ss` ,当格式`yyyy-MM-ddTHH:mm:ss`变成这个时还是使用上面的转换器的话就会报错!!!

 - **WebMVC框架的转换器**: 针对@**`RequestParma`/`Pathvariable`**
- **JacksonObjectMapper依赖包的消息转换器**: 针对@**`RequestBody/ResponseBody`**

- @JsonForm和@DateTimeForm相当于转换器的小部件
	- **@JsonForm 对 @RequestBody有效, 所以要定义在实体类的属性上**
	- **@DateTimeForm 对 @RequestParma有效**

- **时区的偏移**: Date日期在数据库中默认是 日期+时间 的, 所以在某些特定条件下 保存到数据库的`Date`**只有日期**部分 **时区如果不设定会使时间出现偏移**(默认使用系统时区, 但是开启代理会改变时区)
