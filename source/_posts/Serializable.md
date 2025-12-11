---
title: 逻辑与物理序列化
tags:
  - Serializable
  - IO
  - InputStream
  - OutputStream
categories: 编程
date: 2024-11-19 14:52:38
---

------
# 逻辑与物理序列化

🤔💬`如果一个人不知道要驶向哪一个码头, 那么任何风都不会是顺风

------
## 序列化和反序列化的区别
**逻辑序列化:	对象	-> 	JSON**
**物理序列化	 JSON	->	字节**
- **序列化是将 对象 转化成 字节 存储在磁盘(Redis通过RDB或AOF的持久化数据就是放在磁盘上)** 
- **反序列化是读取磁盘 将字节 转化成 对象**
- 当然不只是磁盘, 因为是字节, 所以当成字节流可以在网络中运输
- 不同进程/程序间进行远程通信时，可以相互发送各种类型的数据，包括文本、图片、音频、视频等，而这些数据都会以二进制序列(字节流)的形式在网络上传送。
- `transient` 关键字修饰的成员变量，将不参与序列化  
## 序列化接口和序列化ID
-  只有实现了Serializable或者Externalizable接口的类的对象才能被序列化为字节序列
-  序列化类的属性没有实现 Serializable接口 那么在序列化就会报错: NotSerializableException
- Java的序列化机制是通过判断运行时类的serialVersionUID来验证版本一致性的，在进行反序列化时，JVM会把传进来的字节流中的serialVersionUID与本地实体类中的serialVersionUID进行比较，如果相同则认为是一致的，便可以进行反序列化，否则就会报序列化版本不一致的异常。
- 步骤:
  - 若User类仅仅实现了Serializable接口定义了自己的序列化id, 则可以按照以下方式进行序列化和反序列化。
  - ObjectOutputStream采用默认的序列化方式，对User对象的非transient的实例变量进行序列化。
  - ObjcetInputStream采用默认的反序列化方式，对对User对象的非transient的实例变量进行反序列化。
## 序列化器
- **网络直接传输数据，但是无法直接传输Json对象或者XML对象，必须在传输前序列化，传输完成后反序列化成对象。**
- **网络底层只能识别和传输二进制数据。无论是 JSON、XML，还是 Java 对象，这些高层次的数据结构都需要转化为字节序列才能传输。**
- 所以所有可在网络上传输的对象都必须是可序列化的。
- springboot中是通过Jackson的**objectMapper序列化器**来实现序列化和反序列化的, 并不依赖java原生序列化接口
  - Jackson 通过**反射读取类的字段**。
  - 利用 `ObjectMapper` 序列化为 JSON 或反序列化为 Java 对象。
  - 与传统的 Java 序列化不同，Jackson 不依赖 `Serializable` 接口。
## 自定义序列化器

- @JsonFormat
  - Jackson 在处理对象转 JSON 时，会使用字段的默认序列化器
  - Jackson 在序列化和反序列化时会扫描字段上的注解
  - 如果字段上有 `@JsonFormat` 注解，Jackson 会为该字段生成一个自定义的序列化器。
  - 这个序列化器会按照 `@JsonFormat` 指定的格式（例如 `pattern` 和 `timezone`）格式化数据。
- @JsonSerialize(using = ToStringSerializer.class)
	- 返回前端是进行 将其他类型转换成String的操作
	- 避免诸如Long类型过长导致的JS精度丢失
## 几种Json与字符串换转工具
- **概念理解: 序列化是比较宽泛的词, 将普通字符串转换为JSON格式字符串也可以叫序列化**
  - Gson 是 Google 开源的 JSON 库
  - FastJSON 是阿里巴巴开源的高性能 JSON 处理库
  - Jackson 是 Spring 框架默认集成的 JSON 序列化工具(SpringMVC 转换默认使用 Jackson)
```java
//      3种将实体类对象转化成json格式的字符串方法
        Object heike = new HeiKe();

        //第一种：谷歌旗下的gson
        Gson gson = new Gson();
        String json1 = gson.toJson(heike);

        //第二种：阿里巴巴的fastJson
        String json2 = JSONObject.toJSONString(heike);

        //第三种：spring框架的jackJson   
        ObjectMapper mapper = new ObjectMapper();
        String json3 = mapper.writeValueAsString(heike);

        System.out.println(json1);
        System.out.println(json2);
        System.out.println(json3);

//         3种将JSON字符串转换为对象

        //第一种：谷歌旗下的gson
        gson.fromJson(json1, HeiKe.class);

        //第二种：fastJson
        JSON.parseObject(json2, HeiKe.class);

        //第三种：jackJson
        mapper.readValue(json3, HeiKe.class);

    }
```

![17320061339471732006133927.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17320061339471732006133927.png)
## Redis序列化器
- spring-data-redis是整合了Lettuce和Jedis的java客户端, 提供了RedisTemplate这样的api来操作Redis
- 通过 **`RedisConnectionFactory`** 来建立 Redis 连接。
- **对不同的数据类型存放容器采取不同序列化器**
- 使用 `Jackson2JsonRedisSerializer` 将 Java 对象序列化为 JSON 格式的字符串，并存入 Redis
- **序列化成JSON字符串, 反序列化成Java对象:** 
	- **Key的序列化**
	  - 如果 key 是字符串类型：使用 `StringRedisSerializer` 将 key 序列化为 UTF-8 编码的字节流。
	  - 如果 key 是对象类型: 一把来说用不到，需要用 `Jackson2JsonRedisSerializer` 或 `JdkSerializationRedisSerializer` 等将对象序列化为 Redis 存储格式（最终是字节流）。
	- **Value的序列化**
	  - 对象类型的 value 通常需要两步：
	    - **JSON 转换**（逻辑序列化）：例如用 `Jackson` 或 `Gson` **将对象序列化为 JSON 字符串**
	    - **字节流转换**（物理序列化）：例如用 `UTF-8` 编码**将 JSON 字符串转换为字节流**。
	    - JSON序列化器会将**类的class类型**写入Json结果中(**很重要**)
- 注意: **Redis 的原生序列化器**是 **JDK 序列化机制**，即通过 Java 的 `Serializable` 接口进行序列化和反序列化。JDK 将对象转换为字节 存到Redis, 并没有进行JSON的转化
```java
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        // 实例化自己的RedisTemplate
        RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
        
        // 设置使用连接工厂连接
        template.setConnectionFactory(factory);

		// key的序列化器
		StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();

		// value的序列化器
        Jackson2JsonRedisSerializer jacksonToJsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jacksonToJsonRedisSerializer.setObjectMapper(om);
        
        // key采用redis原生的String序列化器
        template.setKeySerializer(stringRedisSerializer);
        
        // hash的key也采用redis原生的String序列化器
        template.setHashKeySerializer(stringRedisSerializer);
        
        // value序列化方式采用jackson ObjectMapper的序列化器
        template.setValueSerializer(jacksonToJsonRedisSerializer);
        
        // hash的value序列化方式采用jackson
        template.setHashValueSerializer(jacksonToJsonRedisSerializer);
        
        template.afterPropertiesSet();
        return template;
    }
```

```java
// 开启默认类型（兼容多态反序列化）就是处理嵌套类时保留类型信息,让反序列化顺利,解决出现复杂嵌套反序列化失败问题  
//{  
//  "@class": "com.example.对象",  
//  "id": "1"  
//}  
objectMapper.activateDefaultTyping(  
        objectMapper.getPolymorphicTypeValidator(),  
        ObjectMapper.DefaultTyping.NON_FINAL,  
        JsonTypeInfo.As.PROPERTY  
);
```
## Json格式的转化
注:本文使用ObjectMapper工具类
- Json转实体类
```java
String json = "{\"name\": \"黎淳美\", \"age\": 21}";
ObjectMapper objectMapper = new ObjectMapper();
person person = objectMapper.readValue(json, person.class);

class person{
    String name;
    Integer age;
    ...
}
```
- Json转Map
```java
String json = "{\"key1\": \"value1\", \"key2\": 123}";
        ObjectMapper objectMapper = new ObjectMapper();

	// 方法一: 未显式声明泛型的 Map 默认会将值类型解析为 Object
        Map map = objectMapper.readValue(json, Map.class);
        System.out.println(map.get("key1"));

	// 方法二: TypeReference是抽象类
        Map<String, Object> objectMap = objectMapper.readValue(json, new TypeReference<Map<String, Object>>() {});
```
- Json转数组
```java
String jsonArray = "[{\"name\":\"Tom\",\"age\":25},{\"name\":\"Jerry\",\"age\":22}]";

List<Person> personList = objectMapper.readValue(jsonArray, new TypeReference<List<Person>>() {});
List<Person> personList = objectMapper.readValue(jsonArray, List<person>.class);

```
## IO 流的体系：分块 缓冲 压缩 多线程
- Java IO 流广泛采用装饰者模式，通过各种装饰器类来增强功能,，比如缓冲流，打印流
- 根据数据流向分: 输入流(反序列化) 和 输出流(序列化)
- 根据数据类型分: 字节流 和 字符流  (Byte 和 Char)  
- 根据源文件类型分:  音频视频图片等二进制文件 和 文本文件(方便我们平时对字符进行流操作)  

| 分类/对象 | 字节输入流                    | 字节输出流                     | 字符输入流           | 字符输出流           |     |     |
| ----- | ------------------------ | ------------------------- | --------------- | --------------- | --- | --- |
| 基类    | InputStream              | OutputStream              | Reader          | Writer          |     |     |
| 文件流   | FileInputStream          | FileOutputStream          | FileReader      | FileWriter      |     |     |
| 缓冲流   | BufferedInputStream      | BufferedOutputStream      | BufferedReader  | BufferedWriter  |     |     |
| 转化流   | InputStreamReader(字节→字符) | OutputStreamWriter(字符→字节) | -               | -               |     |     |
| 对象流   | ObjectInputStream        | ObjectOutputStream        | -               | -               |     |     |
| 打印流   | -                        | PrintStream               | -               | PrintWriter     |     |     |
| 数组流   | ByteArrayInputStream     | ByteArrayOutputStream     | CharArrayReader | CharArrayWriter |     |     |
| 管道流   | PipeInputStream          | PipedOutputStream         | PipedReader     | PipedWriter     |     |     |
|       |                          |                           |                 |                 |     |     |
 
- 字节型缓冲流高效的原因：  
	- `BufferedInputStream`：在该类型中准备了一个缓存数组，存储字节信息，当外界调用 `read` 方法想获取一个字节的时候，该对象从文件中一次性读取了8192 个字节到数组中，只返回了第一个字节给调用者。将来调用者再次调用 `read` 方法时，当前对象就不需要再次访问磁盘，只需要从数组中取出字节返回给调用者即可，由于读取的是数组，所以速度非常快。当 8192 个字节全都读取完成之后，再需要读取一个字节，就得让该对象到文件中读取下一个 8192 个字节
	- `BufferedOutputStream`：在该类型中准备了一个数组，存储字节信息，当外界调用 `write`  
	  方法想写出一个字节的时候，该对象直接将这个字节存储到了自己的数组中，而不刷新到文件中。一直到该数组所有 8192  个位置全都占满，该对象才把这个数组中的所有数据一次性写出到目标文件中。如果最后一次循环没有将数组写满，最终在关闭流对象的时候，也会将该数组中的数据刷新到文件中
- 字符型缓冲流高效的原因：
	- `BufferedReader` ：每次调用 `read` 方法，只有第一次从磁盘中读取了 8192个字符，存储到该类型对象的缓冲区数组中，将其中一个返回给调用者，再次调用
	  `read` 方法时，就不需要访问磁盘，直接从缓冲区中拿出一个数据即可，提升了效率  
	- `BufferedWriter`：每次调用 `write` 方法，不会直接将字符刷新到文件中，而是存储到字符数组中，等字符数组写满了，才一次性刷新到文件中，减少了IO次数，提升了效率 
## Tips
- 将数据存储到数据库要不要经过序列化和反序列化?
	- 数据库操作直接通过 ORM 框架 或 SQL 将数据与数据库表中的记录映射(mapper)，不涉及 Java 的序列化机制, 如果需要将对象存储为 JSON 格式，可以通过序列化工具（如 Jackson、Gson）将对象转换为 JSON 字符串存储到数据库的对应字段(一般是String )
- Java 缓冲区溢出主要是由于向缓冲区写入的数据超过其能够存储的数据量?
	- 合理设置缓冲区大小
	- 控制写入数据量